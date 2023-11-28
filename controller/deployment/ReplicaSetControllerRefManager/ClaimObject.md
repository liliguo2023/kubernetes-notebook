~~~go
// 判断RS是否可以被认领
func (m *BaseControllerRefManager) ClaimObject(ctx context.Context, obj metav1.Object, match func(metav1.Object) bool, adopt, release func(context.Context, metav1.Object) error) (bool, error) {
	// 获取RS的ownerReferences
  controllerRef := metav1.GetControllerOfNoCopy(obj)
	// 如果不是孤儿走下面的逻辑
  if controllerRef != nil {
    // 判断是否为自己管理的RS，不是直接返回
		if controllerRef.UID != m.Controller.GetUID() {
			return false, nil
		}
    // 判断标签是否匹配，如果标签也匹配，则认领
		if match(obj) {
			return true, nil
		}
		
    // 判断deployment是否被删除，如果被删除则不认领
		if m.Controller.GetDeletionTimestamp() != nil {
			return false, nil
		}
    // 放弃ownerReferences是自己，但是标签不匹配的rs
    // 构造一个delete类型的patch，然后执行patch操作删除相关的ownerReferences
    // 测试1: 可以创建一个deployment，经过变更产生多个RS时，将老的RS label修改，然后看到效果
		if err := release(ctx, obj); err != nil {
			// If the pod no longer exists, ignore the error.
			if errors.IsNotFound(err) {
				return false, nil
			}
			// Either someone else released it, or there was a transient error.
			// The controller should requeue and try again if it's still stale.
			return false, err
		}
		// Successfully released.
		return false, nil
	}

	// It's an orphan.
  // 下面的判断前提是RS没有ownerReferences，也就是是个孤儿
  // 如果当前deployment已经被删除或者标签不匹配，则放弃
	if m.Controller.GetDeletionTimestamp() != nil || !match(obj) {
		// Ignore if we're being deleted or selector doesn't match.
		return false, nil
	}
  // 如果RS已经被删除则放弃
	if obj.GetDeletionTimestamp() != nil {
		// Ignore if the object is being deleted
		return false, nil
	}

  // 放弃namespace不匹配的
  // 获取RS列表的时候传递的是deployment的namesapce，出于什么情况才会造成namespace不匹配呢？
	if len(m.Controller.GetNamespace()) > 0 && m.Controller.GetNamespace() != obj.GetNamespace() {
		// Ignore if namespace not match
		return false, nil
	}

	// Selector matches. Try to adopt.
  // 当前deployment和rs都没有被删除，并且标签匹配，则开始认领
  // 首先会调用 canAdoptFunc 然后会打个patch
  // 测试2: 把测试1修改过的RS的label改回去，然后就可以发现RS又增加了ownerReferences
	if err := adopt(ctx, obj); err != nil {
		// If the pod no longer exists, ignore the error.
		if errors.IsNotFound(err) {
			return false, nil
		}
		// Either someone else claimed it first, or there was a transient error.
		// The controller should requeue and try again if it's still orphaned.
		return false, err
	}
	// Successfully adopted.
	return true, nil
}
~~~

