~~~go
// 返回当前再用的RS: newRS 以及旧的RS列表: allOldRSs
func (dc *DeploymentController) getAllReplicaSetsAndSyncRevision(ctx context.Context, d *apps.Deployment, rsList []*apps.ReplicaSet, createIfNotExisted bool) (*apps.ReplicaSet, []*apps.ReplicaSet, error) {
  // 获取allOldRS
	_, allOldRSs := deploymentutil.FindOldReplicaSets(d, rsList)

	// Get new replica set with the updated revision number
  // 获取newRS
	newRS, err := dc.getNewReplicaSet(ctx, d, rsList, allOldRSs, createIfNotExisted)
	if err != nil {
		return nil, nil, err
	}

	return newRS, allOldRSs, nil
}
~~~

