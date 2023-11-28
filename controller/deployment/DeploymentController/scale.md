~~~go
func (dc *DeploymentController) scale(ctx context.Context, deployment *apps.Deployment, newRS *apps.ReplicaSet, oldRSs []*apps.ReplicaSet) error {
	// If there is only one active replica set then we should scale that up to the full count of the
	// deployment. If there is no active replica set, then we should scale up the newest replica set.
  // 如果有可用的RS则调整RS的replicas和deployment一样
	if activeOrLatest := deploymentutil.FindActiveOrLatest(newRS, oldRSs); activeOrLatest != nil {
		if *(activeOrLatest.Spec.Replicas) == *(deployment.Spec.Replicas) {
			return nil
		}
    // 对比RS和deployment的replica数量进行up或者down操作
		_, _, err := dc.scaleReplicaSetAndRecordEvent(ctx, activeOrLatest, *(deployment.Spec.Replicas), deployment)
		return err
	}

	// If the new replica set is saturated, old replica sets should be fully scaled down.
	// This case handles replica set adoption during a saturated new replica set.
  // 当newRS所有pod启动之后，删除之前RS的pod
	if deploymentutil.IsSaturated(deployment, newRS) {
		for _, old := range controller.FilterActiveReplicaSets(oldRSs) {
			if _, _, err := dc.scaleReplicaSetAndRecordEvent(ctx, old, 0, deployment); err != nil {
				return err
			}
		}
		return nil
	}

	// There are old replica sets with pods and the new replica set is not saturated.
	// We need to proportionally scale all replica sets (new and old) in case of a
	// rolling deployment.
  // 判断spec.strategy.type是不是RollingUpdate
  // 测试: 创建一个2副本的deployment，然后更新label，新副本有一个pod之后，再次更新deployment同时调整replicas和label
	if deploymentutil.IsRollingUpdate(deployment) {
    // 计算出现在spec.replicas不为0的RS
		allRSs := controller.FilterActiveReplicaSets(append(oldRSs, newRS))
    // 计算出RS期望的replica总数
		allRSsReplicas := deploymentutil.GetReplicaCountForReplicaSets(allRSs)

		allowedSize := int32(0)
    // 计算允许存在的最大pod数
		if *(deployment.Spec.Replicas) > 0 {
			allowedSize = *(deployment.Spec.Replicas) + deploymentutil.MaxSurge(*deployment)
		}

		// Number of additional replicas that can be either added or removed from the total
		// replicas count. These replicas should be distributed proportionally to the active
		// replica sets.
    // 计算需要调整的pod数，deploymentReplicasToAdd是可能为0的，滚动升级时
		deploymentReplicasToAdd := allowedSize - allRSsReplicas


		// In such a case when scaling up, we should scale up newer replica sets first, and
		// when scaling down, we should scale down older replica sets first.
    // 判断是scale up 还是scale down，由于deploymentReplicasToAdd可能为0，所以scalingOperation 可能为空
		var scalingOperation string
		switch {
		case deploymentReplicasToAdd > 0:
      // 按照replicas降序排序，如果数量一样，按照创建时间戳降序
			sort.Sort(controller.ReplicaSetsBySizeNewer(allRSs))
			scalingOperation = "up"

		case deploymentReplicasToAdd < 0:
       // 按照replicas降序排序，如果数量一样，按照创建时间戳升序
			sort.Sort(controller.ReplicaSetsBySizeOlder(allRSs))
			scalingOperation = "down"
		}

		// Iterate over all active replica sets and estimate proportions for each of them.
		// The absolute value of deploymentReplicasAdded should never exceed the absolute
		// value of deploymentReplicasToAdd.
		deploymentReplicasAdded := int32(0)
		nameToSize := make(map[string]int32)
		logger := klog.FromContext(ctx)
		for i := range allRSs {
			rs := allRSs[i]

			// Estimate proportions if we have replicas to add, otherwise simply populate
			// nameToSize with the current sizes for each replica set.
			if deploymentReplicasToAdd != 0 {
        // 如果deployment需要调整的pod数不为0，则计算RS需要调整的pod数
				proportion := deploymentutil.GetProportion(logger, rs, *deployment, deploymentReplicasToAdd, deploymentReplicasAdded)

        // 记录RS应达到的pod数
				nameToSize[rs.Name] = *(rs.Spec.Replicas) + proportion
        // 记录已经分配给RS的可调整pod数，下一个RS可调整的数就会减少
				deploymentReplicasAdded += proportion
			} else {
        // 如果不能调整pod数，直接记录当前replicas
				nameToSize[rs.Name] = *(rs.Spec.Replicas)
			}
		}

		// Update all replica sets
    // 遍历RS调整repicas
		for i := range allRSs {
			rs := allRSs[i]

			// Add/remove any leftovers to the largest replica set.
      // 如果还有可以调整的pod数，则分配给第一个RS(allRSs是按照当前replicas数量降序排序的)
			if i == 0 && deploymentReplicasToAdd != 0 {
				leftover := deploymentReplicasToAdd - deploymentReplicasAdded
				nameToSize[rs.Name] = nameToSize[rs.Name] + leftover
				if nameToSize[rs.Name] < 0 {
					nameToSize[rs.Name] = 0
				}
			}

			// TODO: Use transactions when we have them.
			if _, _, err := dc.scaleReplicaSet(ctx, rs, nameToSize[rs.Name], deployment, scalingOperation); err != nil {
				// Return as soon as we fail, the deployment is requeued
				return err
			}
		}
	}
	return nil
}
~~~

