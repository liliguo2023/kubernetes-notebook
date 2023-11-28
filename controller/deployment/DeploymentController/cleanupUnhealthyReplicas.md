~~~go
// cleanupUnhealthyReplicas will scale down old replica sets with unhealthy replicas, so that all unhealthy replicas will be deleted.
func (dc *DeploymentController) cleanupUnhealthyReplicas(ctx context.Context, oldRSs []*apps.ReplicaSet, deployment *apps.Deployment, maxCleanupCount int32) ([]*apps.ReplicaSet, int32, error) {
	logger := klog.FromContext(ctx)
  // 创建时间升序
	sort.Sort(controller.ReplicaSetsByCreationTimestamp(oldRSs))
	// Safely scale down all old replica sets with unhealthy replicas. Replica set will sort the pods in the order
	// such that not-ready < ready, unscheduled < scheduled, and pending < running. This ensures that unhealthy replicas will
	// been deleted first and won't increase unavailability.
  // 记录已经被删除的pod数
	totalScaledDown := int32(0)
	for i, targetRS := range oldRSs {
		if totalScaledDown >= maxCleanupCount {
			break
		}
    // 跳过不能scale down的rs
		if *(targetRS.Spec.Replicas) == 0 {
			// cannot scale down this replica set.
			continue
		}
		logger.V(4).Info("Found available pods in old RS", "replicaSet", klog.KObj(targetRS), "availableReplicas", targetRS.Status.AvailableReplicas)
    // 如果没有unhealthy(没ready)的pod则跳过
		if *(targetRS.Spec.Replicas) == targetRS.Status.AvailableReplicas {
			// no unhealthy replicas found, no scaling required.
			continue
		}

    // 取剩余可scale down的pod数和unhealthy pod中较小的值
		scaledDownCount := int32(integer.IntMin(int(maxCleanupCount-totalScaledDown), int(*(targetRS.Spec.Replicas)-targetRS.Status.AvailableReplicas)))
    // 计算scale down之后pod数
		newReplicasCount := *(targetRS.Spec.Replicas) - scaledDownCount
		if newReplicasCount > *(targetRS.Spec.Replicas) {
			return nil, 0, fmt.Errorf("when cleaning up unhealthy replicas, got invalid request to scale down %s/%s %d -> %d", targetRS.Namespace, targetRS.Name, *(targetRS.Spec.Replicas), newReplicasCount)
		}
    // 进行scale操作
		_, updatedOldRS, err := dc.scaleReplicaSetAndRecordEvent(ctx, targetRS, newReplicasCount, deployment)
		if err != nil {
			return nil, totalScaledDown, err
		}
    // 更新已经scale down的RS并记录scale down的pod数
		totalScaledDown += scaledDownCount
		oldRSs[i] = updatedOldRS
	}
  // 返回已经scale down的pod数(unhealthy)
	return oldRSs, totalScaledDown, nil
}
~~~

