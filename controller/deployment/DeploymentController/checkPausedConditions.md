~~~go
// 检测deployment的status和暂停状态是否匹配，如果不匹配就更新status
func (dc *DeploymentController) checkPausedConditions(ctx context.Context, d *apps.Deployment) error {
	// 如果没设置progressDeadlineSeconds或者progressDeadlineSeconds等于math.MaxInt32,不用调整status
	if !deploymentutil.HasProgressDeadline(d) {
		return nil
	}
  // 获取type是Progressing的conditions
	cond := deploymentutil.GetDeploymentCondition(d.Status, apps.DeploymentProgressing)
  // 如果reason是ProgressDeadlineExceeded 不用调整status
	if cond != nil && cond.Reason == deploymentutil.TimedOutReason {
		// If we have reported lack of progress, do not overwrite it with a paused condition.
		return nil
	}
  //判断conditions是否是DeploymentPaused
	pausedCondExists := cond != nil && cond.Reason == deploymentutil.PausedDeployReason

	needsUpdate := false
  // 以deployment spec.pause 为准，调整status
  // kubectl rollout pause 暂停
  // kubectl rollout resume 恢复
	if d.Spec.Paused && !pausedCondExists {
		condition := deploymentutil.NewDeploymentCondition(apps.DeploymentProgressing, v1.ConditionUnknown, deploymentutil.PausedDeployReason, "Deployment is paused")
		deploymentutil.SetDeploymentCondition(&d.Status, *condition)
		needsUpdate = true
	} else if !d.Spec.Paused && pausedCondExists {
		condition := deploymentutil.NewDeploymentCondition(apps.DeploymentProgressing, v1.ConditionUnknown, deploymentutil.ResumedDeployReason, "Deployment is resumed")
		deploymentutil.SetDeploymentCondition(&d.Status, *condition)
		needsUpdate = true
	}

	if !needsUpdate {
		return nil
	}

	var err error
  // 更新状态
	_, err = dc.client.AppsV1().Deployments(d.Namespace).UpdateStatus(ctx, d, metav1.UpdateOptions{})
	return err
}
~~~

