~~~go
// 清理deployment的RS
func (dc *DeploymentController) cleanupDeployment(ctx context.Context, oldRSs []*apps.ReplicaSet, deployment *apps.Deployment) error {
	logger := klog.FromContext(ctx)
  // 如果deployment的spec.revisionHistoryLimit没设置或者是math.MaxInt32 直接返回
	if !deploymentutil.HasRevisionHistoryLimit(deployment) {
		return nil
	}

	// Avoid deleting replica set with deletion timestamp set
  // 排除掉即将删除的RS
	aliveFilter := func(rs *apps.ReplicaSet) bool {
		return rs != nil && rs.ObjectMeta.DeletionTimestamp == nil
	}
	cleanableRSes := controller.FilterReplicaSets(oldRSs, aliveFilter)

  // 如果没到上限直接返回
	diff := int32(len(cleanableRSes)) - *deployment.Spec.RevisionHistoryLimit
	if diff <= 0 {
		return nil
	}

  // 升序
	sort.Sort(deploymentutil.ReplicaSetsByRevision(cleanableRSes))
	logger.V(4).Info("Looking to cleanup old replica sets for deployment", "deployment", klog.KObj(deployment))

  // 开始清理旧的RS
	for i := int32(0); i < diff; i++ {
		rs := cleanableRSes[i]
		// Avoid delete replica set with non-zero replica counts
		if rs.Status.Replicas != 0 || *(rs.Spec.Replicas) != 0 || rs.Generation > rs.Status.ObservedGeneration || rs.DeletionTimestamp != nil {
			continue
		}
		logger.V(4).Info("Trying to cleanup replica set for deployment", "replicaSet", klog.KObj(rs), "deployment", klog.KObj(deployment))
		if err := dc.client.AppsV1().ReplicaSets(rs.Namespace).Delete(ctx, rs.Name, metav1.DeleteOptions{}); err != nil && !errors.IsNotFound(err) {
			// Return error instead of aggregating and continuing DELETEs on the theory
			// that we may be overloading the api server.
			return err
		}
	}

	return nil
}
~~~

