~~~go
// syncStatusOnly only updates Deployments Status and doesn't take any mutating actions.
func (dc *DeploymentController) syncStatusOnly(ctx context.Context, d *apps.Deployment, rsList []*apps.ReplicaSet) error {
  // 获取newRS 和oldRSs
	newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(ctx, d, rsList, false)
	if err != nil {
		return err
	}

	allRSs := append(oldRSs, newRS)
  // 更新deployment状态
	return dc.syncDeploymentStatus(ctx, allRSs, newRS, d)
}
~~~

