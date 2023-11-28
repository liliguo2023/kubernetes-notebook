~~~go
// FindNewReplicaSet returns the new RS this given deployment targets (the one with the same pod template).
// 创建/变更:			 因为没有匹配的RS，所以返回的是nil
// scale/delete:	因为存在RS，所以返回的就是当前再用的RS
// rollout undo:	因为要回滚，所以返回的就是即将要回滚到的RS
func FindNewReplicaSet(deployment *apps.Deployment, rsList []*apps.ReplicaSet) *apps.ReplicaSet {
  // 先根据创建时间排序，升序
	sort.Sort(controller.ReplicaSetsByCreationTimestamp(rsList))
	for i := range rsList {
    // 忽略pod-template-hash，然后进行比较
		if EqualIgnoreHash(&rsList[i].Spec.Template, &deployment.Spec.Template) {
			// In rare cases, such as after cluster upgrades, Deployment may end up with
			// having more than one new ReplicaSets that have the same template as its template,
			// see https://github.com/kubernetes/kubernetes/issues/40415
			// We deterministically choose the oldest new ReplicaSet.
      // 如果有多个rs能和deploy匹配，选择其中创建时间最老的那个当做newRS返回
			return rsList[i]
		}
	}
	// new ReplicaSet does not exist.
	return nil
}
~~~

