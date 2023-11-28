~~~go
// isScalingEvent checks whether the provided deployment has been updated with a scaling event
// by looking at the desired-replicas annotation in the active replica sets of the deployment.
//
// rsList should come from getReplicaSetsForDeployment(d).
func (dc *DeploymentController) isScalingEvent(ctx context.Context, d *apps.Deployment, rsList []*apps.ReplicaSet) (bool, error) {
  // 获取newRs和oldRS
	newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(ctx, d, rsList, false)
	if err != nil {
		return false, err
	}
	allRSs := append(oldRSs, newRS)
	logger := klog.FromContext(ctx)
  // 筛选出期望pod数不为0的RS
	for _, rs := range controller.FilterActiveReplicaSets(allRSs) {
    // 获取注解 deployment.kubernetes.io/desired-replicas
		desired, ok := deploymentutil.GetDesiredReplicasAnnotation(logger, rs)
		if !ok {
			continue
		}
    // 判断是否是scale操作
		if desired != *(d.Spec.Replicas) {
			return true, nil
		}
	}
	return false, nil
}
~~~

