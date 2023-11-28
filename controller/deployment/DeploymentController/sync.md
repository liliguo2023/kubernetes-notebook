~~~go
func (dc *DeploymentController) sync(ctx context.Context, d *apps.Deployment, rsList []*apps.ReplicaSet) error {
	// 获取newRS 和oldRS
  newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(ctx, d, rsList, false)
	if err != nil {
		return err
	}
  // 执行scale up或者down
	if err := dc.scale(ctx, d, newRS, oldRSs); err != nil {
		// If we get an error while trying to scale, the deployment will be requeued
		// so we can abort this resync
		return err
	}

	// Clean up the deployment when it's paused and no rollback is in flight.
  // 清理历史RS，getRollbackTo需要看看是不是之前版本的deployment,在pause的时候可以回滚
	if d.Spec.Paused && getRollbackTo(d) == nil {
		if err := dc.cleanupDeployment(ctx, oldRSs, d); err != nil {
			return err
		}
	}

	allRSs := append(oldRSs, newRS)
  // 同步状态
	return dc.syncDeploymentStatus(ctx, allRSs, newRS, d)
}
~~~

