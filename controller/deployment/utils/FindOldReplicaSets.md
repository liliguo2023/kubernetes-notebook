~~~~go
// 找出不同类型的RS
func FindOldReplicaSets(deployment *apps.Deployment, rsList []*apps.ReplicaSet) ([]*apps.ReplicaSet, []*apps.ReplicaSet) {
	var requiredRSs []*apps.ReplicaSet
	var allRSs []*apps.ReplicaSet
  // 找到 newRS
	newRS := FindNewReplicaSet(deployment, rsList)
	for _, rs := range rsList {
		// Filter out new replica set
    // 排除掉 newRS
		if newRS != nil && rs.UID == newRS.UID {
			continue
		}
    // 所有RS加到allRSs
		allRSs = append(allRSs, rs)
    // replica不为0的加到requiredRSs
    // 例如deployment rollout undo和pod-template-hash变更时就会出现多个RS的replica不为0
		if *(rs.Spec.Replicas) != 0 {
			requiredRSs = append(requiredRSs, rs)
		}
	}
	return requiredRSs, allRSs
}
~~~~

