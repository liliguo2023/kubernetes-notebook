~~~go
// scaleDownOldReplicaSetsForRollingUpdate scales down old replica sets when deployment strategy is "RollingUpdate".
// Need check maxUnavailable to ensure availability
func (dc *DeploymentController) scaleDownOldReplicaSetsForRollingUpdate(ctx context.Context, allRSs []*apps.ReplicaSet, oldRSs []*apps.ReplicaSet, deployment *apps.Deployment) (int32, error) {
	logger := klog.FromContext(ctx)
  // 计算最大不可用pod数
	maxUnavailable := deploymentutil.MaxUnavailable(*deployment)

	// Check if we can scale down.
  // 计算最小可用pod数
	minAvailable := *(deployment.Spec.Replicas) - maxUnavailable
	// Find the number of available pods.
  // 计算当前ready的pod数
	availablePodCount := deploymentutil.GetAvailableReplicaCountForReplicaSets(allRSs)
	if availablePodCount <= minAvailable {
		// Cannot scale down.
		return 0, nil
	}
	logger.V(4).Info("Found available pods in deployment, scaling down old RSes", "deployment", klog.KObj(deployment), "availableReplicas", availablePodCount)
	// 按创建时间升序
	sort.Sort(controller.ReplicaSetsByCreationTimestamp(oldRSs))

  // 已经scale down的pod数
	totalScaledDown := int32(0)
  // 计算要scale down的pod数
	totalScaleDownCount := availablePodCount - minAvailable
	for _, targetRS := range oldRSs {
		if totalScaledDown >= totalScaleDownCount {
			// No further scaling required.
			break
		}
		if *(targetRS.Spec.Replicas) == 0 {
			// cannot scale down this ReplicaSet.
			continue
		}
		// Scale down.
    // 计算当前RS 可以scale down的pod数
		scaleDownCount := int32(integer.IntMin(int(*(targetRS.Spec.Replicas)), int(totalScaleDownCount-totalScaledDown)))
    // 计算当前RS scale down之后的pod数
		newReplicasCount := *(targetRS.Spec.Replicas) - scaleDownCount
		if newReplicasCount > *(targetRS.Spec.Replicas) {
			return 0, fmt.Errorf("when scaling down old RS, got invalid request to scale down %s/%s %d -> %d", targetRS.Namespace, targetRS.Name, *(targetRS.Spec.Replicas), newReplicasCount)
		}
		_, _, err := dc.scaleReplicaSetAndRecordEvent(ctx, targetRS, newReplicasCount, deployment)
		if err != nil {
			return totalScaledDown, err
		}

    // 更新已经scale down的pod数
		totalScaledDown += scaleDownCount
	}

	return totalScaledDown, nil
}
~~~

