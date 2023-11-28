~~~go
// 用于设置RS的注解并返回RS注解是否变更过
func SetNewReplicaSetAnnotations(ctx context.Context, deployment *apps.Deployment, newRS *apps.ReplicaSet, newRevision string, exists bool, revHistoryLimitInChars int) bool {
	logger := klog.FromContext(ctx)
	// First, copy deployment's annotations (except for apply and revision annotations)
  // 获取RS是否需要变更注解
	annotationChanged := copyDeploymentAnnotationsToReplicaSet(deployment, newRS)
	// Then, update replica set's revision annotation
	if newRS.Annotations == nil {
		newRS.Annotations = make(map[string]string)
	}
  // 获取newRS的revision
	oldRevision, ok := newRS.Annotations[RevisionAnnotation]

	oldRevisionInt, err := strconv.ParseInt(oldRevision, 10, 64)
	if err != nil {
		if oldRevision != "" {
			logger.Info("Updating replica set revision OldRevision not int", "err", err)
			return false
		}
		//If the RS annotation is empty then initialise it to 0
		oldRevisionInt = 0
	}
	newRevisionInt, err := strconv.ParseInt(newRevision, 10, 64)
	if err != nil {
		logger.Info("Updating replica set revision NewRevision not int", "err", err)
		return false
	}
  // 如果newRS的 revision比通过oldRSs计算出来的小，则更新
  // 比如通过kubectl rollout undo回滚到之前的RS，此时RS可以复用，但是revision需要更新
	if oldRevisionInt < newRevisionInt {
		newRS.Annotations[RevisionAnnotation] = newRevision
		annotationChanged = true
		logger.V(4).Info("Updating replica set revision", "replicaSet", klog.KObj(newRS), "newRevision", newRevision)
	}
	// If a revision annotation already existed and this replica set was updated with a new revision
	// then that means we are rolling back to this replica set. We need to preserve the old revisions
	// for historical information.
	if ok && oldRevisionInt < newRevisionInt {
    // 获取 RevisionHistoryAnnotation , 类似: deployment.kubernetes.io/revision-history: 2,4,记录着RS用过的revision
    // 测试: 创建一个deployment，变更一次产生两个rs，然后kubectl rollout undo多次之后就能看到效果
		revisionHistoryAnnotation := newRS.Annotations[RevisionHistoryAnnotation]
    // 生成revision的切片
		oldRevisions := strings.Split(revisionHistoryAnnotation, ",")
    // 判断是否没有RevisionHistoryAnnotation
    // 即使revisionHistoryAnnotation是空字符串，切片长度也是1，不是0，所以能这么判断,但是这个判断方式看着真难受
		if len(oldRevisions[0]) == 0 {
      // 增加一个注解，内容是RS使用过的revision
			newRS.Annotations[RevisionHistoryAnnotation] = oldRevision
		} else {
      // 因为revisionHistoryAnnotation有长度限制，所以当长度不够的时候，要将最早的revision删除掉，再增加新的
			totalLen := len(revisionHistoryAnnotation) + len(oldRevision) + 1
			// index for the starting position in oldRevisions
			start := 0
			for totalLen > revHistoryLimitInChars && start < len(oldRevisions) {
				totalLen = totalLen - len(oldRevisions[start]) - 1
				start++
			}
			if totalLen <= revHistoryLimitInChars {
				oldRevisions = append(oldRevisions[start:], oldRevision)
				newRS.Annotations[RevisionHistoryAnnotation] = strings.Join(oldRevisions, ",")
			} else {
				logger.Info("Not appending revision due to revision history length limit reached", "revisionHistoryLimit", revHistoryLimitInChars)
			}
		}
	}
	// If the new replica set is about to be created, we need to add replica annotations to it.
  // 如果newRS不存在(创建的时候)，设置注解deployment.kubernetes.io/desired-replicas、deployment.kubernetes.io/max-replicas，MaxSurge函数根据deployment计算出rolling update时最大的pod数
	if !exists && SetReplicasAnnotations(newRS, *(deployment.Spec.Replicas), *(deployment.Spec.Replicas)+MaxSurge(*deployment)) {
		annotationChanged = true
	}
  // 返回newRS注解是否变更过
	return annotationChanged
}
~~~

