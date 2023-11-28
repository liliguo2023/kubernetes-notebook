~~~go
// 先查出所有pod，因为不同名字的deployment的selector可以是一样的，所以还需要判断pod的ownerReferences
func (dc *DeploymentController) getPodMapForDeployment(d *apps.Deployment, rsList []*apps.ReplicaSet) (map[types.UID][]*v1.Pod, error) {
	// Get all Pods that potentially belong to this Deployment.
  // 获取label
	selector, err := metav1.LabelSelectorAsSelector(d.Spec.Selector)
	if err != nil {
		return nil, err
	}
  // 查找出所有label匹配的pod
	pods, err := dc.podLister.Pods(d.Namespace).List(selector)
	if err != nil {
		return nil, err
	}
	// Group Pods by their controller (if it's in rsList).
  // 初始化podMap
	podMap := make(map[types.UID][]*v1.Pod, len(rsList))
	for _, rs := range rsList {
		podMap[rs.UID] = []*v1.Pod{}
	}
  // 如果pod的ownerReferences在rs列表中就记录到podMap
	for _, pod := range pods {
		// Do not ignore inactive Pods because Recreate Deployments need to verify that no
		// Pods from older versions are running before spinning up new Pods.
		controllerRef := metav1.GetControllerOf(pod)
		if controllerRef == nil {
			continue
		}
		// Only append if we care about this UID.
		if _, ok := podMap[controllerRef.UID]; ok {
			podMap[controllerRef.UID] = append(podMap[controllerRef.UID], pod)
		}
	}
	return podMap, nil
}
~~~

