~~~go
// 
func (dc *DeploymentController) scaleReplicaSetAndRecordEvent(ctx context.Context, rs *apps.ReplicaSet, newScale int32, deployment *apps.Deployment) (bool, *apps.ReplicaSet, error) {
	// 不用scale 直接返回
	if *(rs.Spec.Replicas) == newScale {
		return false, rs, nil
	}
  // 通过副本数判断是scale up 还是scale down
	var scalingOperation string
	if *(rs.Spec.Replicas) < newScale {
		scalingOperation = "up"
	} else {
		scalingOperation = "down"
	}
  // 获取RS是否需要进行scale，并记录scale状态
	scaled, newRS, err := dc.scaleReplicaSet(ctx, rs, newScale, deployment, scalingOperation)
	return scaled, newRS, err
}
~~~

