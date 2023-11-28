~~~go
// 将deployment的注解拷贝给RS
func copyDeploymentAnnotationsToReplicaSet(deployment *apps.Deployment, rs *apps.ReplicaSet) bool {
	rsAnnotationsChanged := false
	if rs.Annotations == nil {
		rs.Annotations = make(map[string]string)
	}
	for k, v := range deployment.Annotations {
		// 1.跳过已经存在并相等的注解
    // 2.跳过annotationsToSkip中的注解
		//var annotationsToSkip = map[string]bool{
		//	v1.LastAppliedConfigAnnotation: true,
		//	RevisionAnnotation:             true,
		//	RevisionHistoryAnnotation:      true,
		//	DesiredReplicasAnnotation:      true,
		//	MaxReplicasAnnotation:          true,
		//	apps.DeprecatedRollbackTo:      true,
		//}

		if _, exist := rs.Annotations[k]; skipCopyAnnotation(k) || (exist && rs.Annotations[k] == v) {
			continue
		}
		rs.Annotations[k] = v
		rsAnnotationsChanged = true
	}
  // 返回RS是否需要变更注解
	return rsAnnotationsChanged
}

~~~

