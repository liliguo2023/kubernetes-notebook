~~~go
// ClaimReplicaSets tries to take ownership of a list of ReplicaSets.
//
// It will reconcile the following:
//   - Adopt orphans if the selector matches.
//   - Release owned objects if the selector no longer matches.
// 
func (m *ReplicaSetControllerRefManager) ClaimReplicaSets(ctx context.Context, sets []*apps.ReplicaSet) ([]*apps.ReplicaSet, error) {
	var claimed []*apps.ReplicaSet
	var errlist []error

  // 判断匹配的功能函数
	match := func(obj metav1.Object) bool {
		return m.Selector.Matches(labels.Set(obj.GetLabels()))
	}
  // 认领功能函数
	adopt := func(ctx context.Context, obj metav1.Object) error {
		return m.AdoptReplicaSet(ctx, obj.(*apps.ReplicaSet))
	}
  // 放弃功能函数
	release := func(ctx context.Context, obj metav1.Object) error {
		return m.ReleaseReplicaSet(ctx, obj.(*apps.ReplicaSet))
	}

  // 遍历所有rs
	for _, rs := range sets {
		ok, err := m.ClaimObject(ctx, rs, match, adopt, release)
		if err != nil {
			errlist = append(errlist, err)
			continue
		}
		if ok {
			claimed = append(claimed, rs)
		}
	}
  // 返回认领列表
	return claimed, utilerrors.NewAggregate(errlist)
}
~~~

