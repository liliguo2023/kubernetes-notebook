~~~go
// 用于获取最新并匹配的RS，如果RS之前已存在则更新注解等信息，如果RS之前不存在，则创建RS
func (dc *DeploymentController) getNewReplicaSet(ctx context.Context, d *apps.Deployment, rsList, oldRSs []*apps.ReplicaSet, createIfNotExisted bool) (*apps.ReplicaSet, error) {
	logger := klog.FromContext(ctx)
  // 找到newRS
	existingNewRS := deploymentutil.FindNewReplicaSet(d, rsList)

	// 计算oldRS的最大revision
	maxOldRevision := deploymentutil.MaxRevision(logger, oldRSs)
	// 计算newRS应该使用的revision
	newRevision := strconv.FormatInt(maxOldRevision+1, 10)

	// Latest replica set exists. We need to sync its annotations (includes copying all but
	// annotationsToSkip from the parent deployment, and update revision, desiredReplicas,
	// and maxReplicas) and also update the revision annotation in the deployment with the
	// latest revision.
  // 如果存在newRS
	if existingNewRS != nil {
		rsCopy := existingNewRS.DeepCopy()

		// Set existing new replica set's annotation
    // 判断注解是否需要变更
		annotationsUpdated := deploymentutil.SetNewReplicaSetAnnotations(ctx, d, rsCopy, newRevision, true, maxRevHistoryLengthInChars)
    // 判断minReadySeconds是否相等，minReadySeconds表示pod再ready状态持续多少秒才算ready
		minReadySecondsNeedsUpdate := rsCopy.Spec.MinReadySeconds != d.Spec.MinReadySeconds
    // 更新newRS
		if annotationsUpdated || minReadySecondsNeedsUpdate {
			rsCopy.Spec.MinReadySeconds = d.Spec.MinReadySeconds
			return dc.client.AppsV1().ReplicaSets(rsCopy.ObjectMeta.Namespace).Update(ctx, rsCopy, metav1.UpdateOptions{})
		}

		// Should use the revision in existingNewRS's annotation, since it set by before
    // 更新deployment的revision为newRS的revision
		needsUpdate := deploymentutil.SetDeploymentRevision(d, rsCopy.Annotations[deploymentutil.RevisionAnnotation])
		// If no other Progressing condition has been recorded and we need to estimate the progress
		// of this deployment then it is likely that old users started caring about progress. In that
		// case we need to take into account the first time we noticed their new replica set.
    // 判断是否需要更新Progressing condition
		cond := deploymentutil.GetDeploymentCondition(d.Status, apps.DeploymentProgressing)
		if deploymentutil.HasProgressDeadline(d) && cond == nil {
			msg := fmt.Sprintf("Found new replica set %q", rsCopy.Name)
			condition := deploymentutil.NewDeploymentCondition(apps.DeploymentProgressing, v1.ConditionTrue, deploymentutil.FoundNewRSReason, msg)
			deploymentutil.SetDeploymentCondition(&d.Status, *condition)
			needsUpdate = true
		}

    // 判断是否要更新deployment的状态
		if needsUpdate {
			var err error
			if _, err = dc.client.AppsV1().Deployments(d.Namespace).UpdateStatus(ctx, d, metav1.UpdateOptions{}); err != nil {
				return nil, err
			}
		}
		return rsCopy, nil
	}

  // 判断是否创建新的RS
	if !createIfNotExisted {
		return nil, nil
	}

	// new ReplicaSet does not exist, create one.
  // 创建新Template并且增加key为pod-template-hash的label和selector
	newRSTemplate := *d.Spec.Template.DeepCopy()
	podTemplateSpecHash := controller.ComputeHash(&newRSTemplate, d.Status.CollisionCount)
	newRSTemplate.Labels = labelsutil.CloneAndAddLabel(d.Spec.Template.Labels, apps.DefaultDeploymentUniqueLabelKey, podTemplateSpecHash)
	// Add podTemplateHash label to selector.
	newRSSelector := labelsutil.CloneSelectorAndAddLabel(d.Spec.Selector, apps.DefaultDeploymentUniqueLabelKey, podTemplateSpecHash)

	// Create new ReplicaSet
  // 构造新的RS
	newRS := apps.ReplicaSet{
		ObjectMeta: metav1.ObjectMeta{
			// Make the name deterministic, to ensure idempotence
			Name:            d.Name + "-" + podTemplateSpecHash,
			Namespace:       d.Namespace,
			OwnerReferences: []metav1.OwnerReference{*metav1.NewControllerRef(d, controllerKind)},
			Labels:          newRSTemplate.Labels,
		},
		Spec: apps.ReplicaSetSpec{
			Replicas:        new(int32),
			MinReadySeconds: d.Spec.MinReadySeconds,
			Selector:        newRSSelector,
			Template:        newRSTemplate,
		},
	}
	allRSs := append(oldRSs, &newRS)
  // 计算replicas
	newReplicasCount, err := deploymentutil.NewRSNewReplicas(d, allRSs, &newRS)
	if err != nil {
		return nil, err
	}

  // 设置.spec.replicas
	*(newRS.Spec.Replicas) = newReplicasCount
	// Set new replica set's annotation
  // 增加注解
	deploymentutil.SetNewReplicaSetAnnotations(ctx, d, &newRS, newRevision, false, maxRevHistoryLengthInChars)
	// Create the new ReplicaSet. If it already exists, then we need to check for possible
	// hash collisions. If there is any other error, we need to report it in the status of
	// the Deployment.
	alreadyExists := false
  // 创建RS
	createdRS, err := dc.client.AppsV1().ReplicaSets(d.Namespace).Create(ctx, &newRS, metav1.CreateOptions{})
	switch {
	// We may end up hitting this due to a slow cache or a fast resync of the Deployment.
    // 如果想要创建的RS因为cache过慢已经存在(之前查RS的时候是从cache中查的)，则判断是否可用,如果不可用则更新status重新入队
	case errors.IsAlreadyExists(err):
		alreadyExists = true

		// Fetch a copy of the ReplicaSet.
		rs, rsErr := dc.rsLister.ReplicaSets(newRS.Namespace).Get(newRS.Name)
		if rsErr != nil {
			return nil, rsErr
		}

		// If the Deployment owns the ReplicaSet and the ReplicaSet's PodTemplateSpec is semantically
		// deep equal to the PodTemplateSpec of the Deployment, it's the Deployment's new ReplicaSet.
		// Otherwise, this is a hash collision and we need to increment the collisionCount field in
		// the status of the Deployment and requeue to try the creation in the next sync.
    // 判断RS是否能直接使用
		controllerRef := metav1.GetControllerOf(rs)
		if controllerRef != nil && controllerRef.UID == d.UID && deploymentutil.EqualIgnoreHash(&d.Spec.Template, &rs.Spec.Template) {
			createdRS = rs
			err = nil
			break
		}

		// Matching ReplicaSet is not equal - increment the collisionCount in the DeploymentStatus
		// and requeue the Deployment.
		if d.Status.CollisionCount == nil {
			d.Status.CollisionCount = new(int32)
		}
		preCollisionCount := *d.Status.CollisionCount
		*d.Status.CollisionCount++
		// Update the collisionCount for the Deployment and let it requeue by returning the original
		// error.
		_, dErr := dc.client.AppsV1().Deployments(d.Namespace).UpdateStatus(ctx, d, metav1.UpdateOptions{})
		if dErr == nil {
			logger.V(2).Info("Found a hash collision for deployment - bumping collisionCount to resolve it", "deployment", klog.KObj(d), "oldCollisionCount", preCollisionCount, "newCollisionCount", *d.Status.CollisionCount)
		}
		return nil, err
	case errors.HasStatusCause(err, v1.NamespaceTerminatingCause):
		// 如果namespace是 terminating 状态, 直接放弃
		return nil, err
	case err != nil:
		msg := fmt.Sprintf("Failed to create new replica set %q: %v", newRS.Name, err)
    // 如果报错，更新状态记录事件
		if deploymentutil.HasProgressDeadline(d) {
			cond := deploymentutil.NewDeploymentCondition(apps.DeploymentProgressing, v1.ConditionFalse, deploymentutil.FailedRSCreateReason, msg)
			deploymentutil.SetDeploymentCondition(&d.Status, *cond)
			// We don't really care about this error at this point, since we have a bigger issue to report.
			// TODO: Identify which errors are permanent and switch DeploymentIsFailed to take into account
			// these reasons as well. Related issue: https://github.com/kubernetes/kubernetes/issues/18568
			_, _ = dc.client.AppsV1().Deployments(d.Namespace).UpdateStatus(ctx, d, metav1.UpdateOptions{})
		}
		dc.eventRecorder.Eventf(d, v1.EventTypeWarning, deploymentutil.FailedRSCreateReason, msg)
		return nil, err
	}
  // 创建成功记录事件
	if !alreadyExists && newReplicasCount > 0 {
		dc.eventRecorder.Eventf(d, v1.EventTypeNormal, "ScalingReplicaSet", "Scaled up replica set %s to %d", createdRS.Name, newReplicasCount)
	}

  // 判断是否需要更新revision和status等
	needsUpdate := deploymentutil.SetDeploymentRevision(d, newRevision)
	if !alreadyExists && deploymentutil.HasProgressDeadline(d) {
		msg := fmt.Sprintf("Created new replica set %q", createdRS.Name)
		condition := deploymentutil.NewDeploymentCondition(apps.DeploymentProgressing, v1.ConditionTrue, deploymentutil.NewReplicaSetReason, msg)
		deploymentutil.SetDeploymentCondition(&d.Status, *condition)
		needsUpdate = true
	}
	if needsUpdate {
		_, err = dc.client.AppsV1().Deployments(d.Namespace).UpdateStatus(ctx, d, metav1.UpdateOptions{})
	}
  // 返回新创建的RS
	return createdRS, err
}
~~~

