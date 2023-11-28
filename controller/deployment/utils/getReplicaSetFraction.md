~~~go
// getReplicaSetFraction estimates the fraction of replicas a replica set can have in
// 1. a scaling event during a rollout or 2. when scaling a paused deployment.
func getReplicaSetFraction(logger klog.Logger, rs apps.ReplicaSet, d apps.Deployment) int32 {
	// If we are scaling down to zero then the fraction of this replica set is its whole size (negative)
	if *(d.Spec.Replicas) == int32(0) {
		return -*(rs.Spec.Replicas)
	}

  // 先计算出deployment允许的最大pod数
	deploymentReplicas := *(d.Spec.Replicas) + MaxSurge(d)
  // 获取RS的 deployment.kubernetes.io/max-replicas
	annotatedReplicas, ok := getMaxReplicasAnnotation(logger, &rs)
	if !ok {
		// If we cannot find the annotation then fallback to the current deployment size. Note that this
		// will not be an accurate proportion estimation in case other replica sets have different values
		// which means that the deployment was scaled at some point but we at least will stay in limits
		// due to the min-max comparisons in getProportion.
		annotatedReplicas = d.Status.Replicas
	}

	// We should never proportionally scale up from zero which means rs.spec.replicas and annotatedReplicas
	// will never be zero here.
  // rs.Spec.Replicas / annotatedReplicas算出来RS已有pod数占比
  // 用已有pod数占比 * deploymentReplicas(deployment允许的最大pod数)，算出RS pod数应该是多少
  // 再用上一步算出来的pod数(向上取整) - 当前replica，算出还需要的pod数
	newRSsize := (float64(*(rs.Spec.Replicas) * deploymentReplicas)) / float64(annotatedReplicas)
	return integer.RoundToInt32(newRSsize) - *(rs.Spec.Replicas)
}
~~~

