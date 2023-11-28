~~~go
// 计算新的RS的replica数量
func NewRSNewReplicas(deployment *apps.Deployment, allRSs []*apps.ReplicaSet, newRS *apps.ReplicaSet) (int32, error) {
	switch deployment.Spec.Strategy.Type {
	case apps.RollingUpdateDeploymentStrategyType:
		// Check if we can scale up.
    // 计算 maxSurge
		maxSurge, err := intstrutil.GetScaledValueFromIntOrPercent(deployment.Spec.Strategy.RollingUpdate.MaxSurge, int(*(deployment.Spec.Replicas)), true)
		if err != nil {
			return 0, err
		}
		// Find the total number of pods
    // 计算当前pod数量
		currentPodCount := GetReplicaCountForReplicaSets(allRSs)
    // 计算最大允许的pod数量
		maxTotalPods := *(deployment.Spec.Replicas) + int32(maxSurge)
    // 如果到达阈值则不能scale up
		if currentPodCount >= maxTotalPods {
			// Cannot scale up.
			return *(newRS.Spec.Replicas), nil
		}
		// Scale up.
    // 计算还可以扩容的数量
		scaleUpCount := maxTotalPods - currentPodCount
		// Do not exceed the number of desired replicas.
    // 新的RS的replica数量，不能超过期望数值,新创建的RS scaleUpCount是会比spec.replicas多的
		scaleUpCount = int32(integer.IntMin(int(scaleUpCount), int(*(deployment.Spec.Replicas)-*(newRS.Spec.Replicas))))
		return *(newRS.Spec.Replicas) + scaleUpCount, nil
	case apps.RecreateDeploymentStrategyType:
		return *(deployment.Spec.Replicas), nil
	default:
		return 0, fmt.Errorf("deployment type %v isn't supported", deployment.Spec.Strategy.Type)
	}
}
~~~

