~~~go
// GetProportion will estimate the proportion for the provided replica set using 1. the current size
// of the parent deployment, 2. the replica count that needs be added on the replica sets of the
// deployment, and 3. the total replicas added in the replica sets of the deployment so far.
func GetProportion(logger klog.Logger, rs *apps.ReplicaSet, d apps.Deployment, deploymentReplicasToAdd, deploymentReplicasAdded int32) int32 {
  // 没有可调整的pod数之后，返回
	if rs == nil || *(rs.Spec.Replicas) == 0 || deploymentReplicasToAdd == 0 || deploymentReplicasToAdd == deploymentReplicasAdded {
		return int32(0)
	}

  // 计算RS应该增加或者减少的pod数
	rsFraction := getReplicaSetFraction(logger, *rs, d)
  // 计算还可以增加几个pod会到达deployment的允许的最大值
	allowed := deploymentReplicasToAdd - deploymentReplicasAdded

  // 如果是scale up返回最小值
	if deploymentReplicasToAdd > 0 {
		// Use the minimum between the replica set fraction and the maximum allowed replicas
		// when scaling up. This way we ensure we will not scale up more than the allowed
		// replicas we can add.
		return integer.Int32Min(rsFraction, allowed)
	}
	// Use the maximum between the replica set fraction and the maximum allowed replicas
	// when scaling down. This way we ensure we will not scale down more than the allowed
	// replicas we can remove.
  // 如果是scale down 返回最大值
	return integer.Int32Max(rsFraction, allowed)
}
~~~

