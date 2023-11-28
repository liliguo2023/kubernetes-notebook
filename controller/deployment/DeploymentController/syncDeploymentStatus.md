~~~go
func (dc *DeploymentController) syncDeploymentStatus(ctx context.Context, allRSs []*apps.ReplicaSet, newRS *apps.ReplicaSet, d *apps.Deployment) error {
  // 获取deployment 最新的status
	newStatus := calculateStatus(allRSs, newRS, d)

  // 如果和当前的status 相等，直接返回
	if reflect.DeepEqual(d.Status, newStatus) {
		return nil
	}

  // 更新deployment的status
	newDeployment := d
	newDeployment.Status = newStatus
	_, err := dc.client.AppsV1().Deployments(newDeployment.Namespace).UpdateStatus(ctx, newDeployment, metav1.UpdateOptions{})
	return err
}


~~~

