~~~go
// FindActiveOrLatest returns the only active or the latest replica set in case there is at most one active
// replica set. If there are more active replica sets, then we should proportionally scale them.

func FindActiveOrLatest(newRS *apps.ReplicaSet, oldRSs []*apps.ReplicaSet) *apps.ReplicaSet {
	// 没有RS直接返回
  if newRS == nil && len(oldRSs) == 0 {
		return nil
	}

  // 按照创建时间逆序
	sort.Sort(sort.Reverse(controller.ReplicaSetsByCreationTimestamp(oldRSs)))
  // 找出期望pod数不为0的RS
	allRSs := controller.FilterActiveReplicaSets(append(oldRSs, newRS))

  // len(allRSs) 是有机会大于1的，比如在做滚动升级期间两个RS都有pod的时候
	switch len(allRSs) {
	case 0:
		// If there is no active replica set then we should return the newest.
		if newRS != nil {
			return newRS
		}
		return oldRSs[0]
	case 1:
		return allRSs[0]
	default:
		return nil
	}
}
~~~