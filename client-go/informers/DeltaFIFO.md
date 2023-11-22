~~~~go
// DeltaFIFO 类似于 FIFO，但有两个不同之处。首先，与给定对象键关联的累加器不是该对象本身，而是一个 Deltas，它是该对象的 Delta 值的切片。
// 对对象应用于 Deltas 意味着追加一个 Delta，但是在 Deltas 已以 Deleted 结尾时，如果将追加的 Delta 是 Deleted，则 Deltas 不会增长。
// 在这种情况下，如果旧的 Deleted 的对象是 DeletedFinalStateUnknown，则终端 Deleted 将被新的 Deleted 替换，尽管 Deltas 不会增长。
//
// 另一个区别是 DeltaFIFO 有两种额外的方式，可以将对象应用于累加器：Replaced 和 Sync。
// 如果 EmitDeltaTypeReplaced 未设置为 true，则在向后兼容性的替换事件中将使用 Sync。Sync 用于定期重新同步事件。
//
// DeltaFIFO 是一个生产者-消费者队列，其中 Reflector 应该是生产者，而调用 Pop() 方法的任何内容都是消费者。
//
// DeltaFIFO 解决了这个用例：
//   - 您希望最多处理每个对象更改（delta）一次。
//   - 处理对象时，您希望看到自上次处理以来发生的所有事情。
//   - 您希望处理其中一些对象的删除。
//   - 您可能希望定期重新处理对象。
//
// DeltaFIFO 的 Pop()、Get() 和 GetByKey() 方法返回 interface{}，以满足 Store/Queue 接口，但它们始终返回 Deltas 类型的对象。
// List() 从 FIFO 中返回每个累加器的最新对象。
//
// DeltaFIFO 的 knownObjects KeyListerGetter 提供了列出 Store 键和按 Store 键获取对象的功能。
// 所涉及的对象称为“已知对象”，并且这组对象修改了 Delete、Replace 和 Resync 方法的行为（每个方法以不同的方式）。
//
// 关于线程的说明：如果从多个线程并行调用 Pop()，则可能导致多个线程处理稍有不同版本的相同对象。


type DeltaFIFO struct {
	// lock/cond protects access to 'items' and 'queue'.
  // lock/cond 用于保护对 'items' 和 'queue' 的访问。
	lock sync.RWMutex
	cond sync.Cond

	// `items` maps a key to a Deltas.
	// Each such Deltas has at least one Delta.
  // `items` 将键映射到 Deltas。每个 Deltas 至少有一个 Delta。
	items map[string]Deltas

	// `queue` maintains FIFO order of keys for consumption in Pop().
	// There are no duplicates in `queue`.
	// A key is in `queue` if and only if it is in `items`.
  // `queue` 用于维护键的先进先出顺序，以便在 Pop() 中消费。
	// `queue` 中没有重复的键。如果键在 `items` 中，则它就在 `queue` 中。
	queue []string

	// populated is true if the first batch of items inserted by Replace() has been populated
	// or Delete/Add/Update/AddIfNotPresent was called first.
  // populated 表示是否已经填充了由 Replace() 插入的第一批项，或者首先调用了 Delete/Add/Update/AddIfNotPresent。
	populated bool
  
	// initialPopulationCount is the number of items inserted by the first call of Replace()
  // initialPopulationCount 是由 Replace() 第一次调用插入的对象数。
	initialPopulationCount int

	// keyFunc is used to make the key used for queued item
	// insertion and retrieval, and should be deterministic.
	keyFunc KeyFunc

	// knownObjects list keys that are "known" --- affecting Delete(),
	// Replace(), and Resync()
	knownObjects KeyListerGetter

	// Used to indicate a queue is closed so a control loop can exit when a queue is empty.
	// Currently, not used to gate any of CRUD operations.
	// 用于指示队列已关闭，以便在队列为空时退出控制循环。
	// 目前，未用于控制 CRUD 操作中的任何一个。  
	closed bool

	// emitDeltaTypeReplaced is whether to emit the Replaced or Sync
	// DeltaType when Replace() is called (to preserve backwards compat).
	// emitDeltaTypeReplaced 表示在调用 Replace() 时是否发出 Replaced 或 Sync DeltaType（以保持向后兼容性）。  
	emitDeltaTypeReplaced bool

	// Called with every object if non-nil.
	// transformer 如果非空，将在每个对象上调用。  
	transformer TransformFunc
}

const (
  // 当一个新的对象被添加到监视的资源中时，会触发 Added 类型的 Delta。这表示资源中有一个新的对象
	Added   DeltaType = "Added"
	// 当监视的资源中的现有对象发生更新时，会触发 Updated 类型的 Delta。这表示资源中的某个对象经历了变更
	Updated DeltaType = "Updated"
	// 当监视的资源中的对象被删除时，会触发 Deleted 类型的 Delta。这表示资源中的某个对象不再存在  
	Deleted DeltaType = "Deleted"

  // 当由于监视错误而不得不执行重新列举（relist）时，会触发 Replaced 类型的 Delta。这表示重新列举可能会导致对象的替换，但我们不知道替换的对象是否发生了变化。
  // 注意：在 DeltaFIFO 的先前版本中，对于替换事件，会使用 Sync 而不是 Replaced。因此，只有在选项 EmitDeltaTypeReplaced 为 true 时，才会发出 Replaced。
	Replaced DeltaType = "Replaced"
	// Sync 类型的 Delta 用于表示在定期重新同步期间的合成事件。这些事件不是由于实际的对象变更引起的，而是由于定期重新同步的周期性触发。
	Sync DeltaType = "Sync"
)
~~~~



~~~go
// 获取key使用
func MetaNamespaceKeyFunc(obj interface{}) (string, error) {
	if key, ok := obj.(ExplicitKey); ok {
		return string(key), nil
	}
	objName, err := ObjectToName(obj)
	if err != nil {
		return "", err
	}
	return objName.String(), nil
}

//
func (f *DeltaFIFO) Replace(list []interface{}, _ string) error {
	f.lock.Lock()
	defer f.lock.Unlock()
	keys := make(sets.String, len(list))

	// keep backwards compat for old clients
	action := Sync
	if f.emitDeltaTypeReplaced {
		action = Replaced
	}

	// Add Sync/Replaced action for each new item.
	for _, item := range list {
    // 默认通过MetaNamespaceKeyFunc获取
    // The key uses the format <namespace>/<name> unless <namespace> is empty, then
		// it's just <name>.
		key, err := f.KeyOf(item)
		if err != nil {
			return KeyError{item, err}
		}
		keys.Insert(key)
    // 更新queue和item
		if err := f.queueActionLocked(action, item); err != nil {
			return fmt.Errorf("couldn't enqueue object: %v", err)
		}
	}

	// Do deletion detection against objects in the queue
  // 清理已经删除对象
	queuedDeletions := 0
	for k, oldItem := range f.items {
    // 判断item中的对象是否在此次的同步列表
		if keys.Has(k) {
			continue
		}
		// Delete pre-existing items not in the new list.
		// This could happen if watch deletion event was missed while
		// disconnected from apiserver.
		var deletedObj interface{}
		if n := oldItem.Newest(); n != nil {
			deletedObj = n.Object

			// if the previous object is a DeletedFinalStateUnknown, we have to extract the actual Object
			if d, ok := deletedObj.(DeletedFinalStateUnknown); ok {
				deletedObj = d.Obj
			}
		}
    // 记录清理对象的个数
		queuedDeletions++
    // 添加已删除的Delta
		if err := f.queueActionLocked(Deleted, DeletedFinalStateUnknown{k, deletedObj}); err != nil {
			return err
		}
	}

	if f.knownObjects != nil {
		// Detect deletions for objects not present in the queue, but present in KnownObjects
    // 获取index中的所有key
		knownKeys := f.knownObjects.ListKeys()
		for _, k := range knownKeys {
      // 如果列表中有，则跳过
			if keys.Has(k) {
				continue
			}
      // 如果items中有 则跳过，上面那次循环处理过的，不再处理
			if len(f.items[k]) > 0 {
				continue
			}

      // 获取object
			deletedObj, exists, err := f.knownObjects.GetByKey(k)
			if err != nil {
				deletedObj = nil
				klog.Errorf("Unexpected error %v during lookup of key %v, placing DeleteFinalStateUnknown marker without object", err, k)
			} else if !exists {
				deletedObj = nil
				klog.Infof("Key %v does not exist in known objects store, placing DeleteFinalStateUnknown marker without object", k)
			}
      // 计数增加
			queuedDeletions++
      // 添加已删除的Delta
			if err := f.queueActionLocked(Deleted, DeletedFinalStateUnknown{k, deletedObj}); err != nil {
				return err
			}
		}
	}

  // 记录同步状态和initialPopulationCount
	if !f.populated {
		f.populated = true
		f.initialPopulationCount = keys.Len() + queuedDeletions
	}

	return nil
}
~~~



~~~~go

func (f *DeltaFIFO) queueActionLocked(actionType DeltaType, obj interface{}) error {
  // 获取key
	id, err := f.KeyOf(obj)
	if err != nil {
		return KeyError{obj, err}
	}

	// Every object comes through this code path once, so this is a good
	// place to call the transform func.  If obj is a
	// DeletedFinalStateUnknown tombstone, then the containted inner object
	// will already have gone through the transformer, but we document that
	// this can happen. In cases involving Replace(), such an object can
	// come through multiple times.
  // 调用转换函数
	if f.transformer != nil {
		var err error
		obj, err = f.transformer(obj)
		if err != nil {
			return err
		}
	}

  // 取出Deltas 并append 新的 Deltas
	oldDeltas := f.items[id]
	newDeltas := append(oldDeltas, Delta{actionType, obj})
  
  // 判断最后两个Delta如果都是已删除的，则保留非DeletedFinalStateUnknown(被删除对象的最终状态未知的情况)状态的Delta
	newDeltas = dedupDeltas(newDeltas)

	if len(newDeltas) > 0 {
    // 如果在queue中不存在，则append进去
		if _, exists := f.items[id]; !exists {
			f.queue = append(f.queue, id)
		}
    // 更新Deltas为最新的
		f.items[id] = newDeltas
		f.cond.Broadcast()
	} else {
		// This never happens, because dedupDeltas never returns an empty list
		// when given a non-empty list (as it is here).
		// If somehow it happens anyway, deal with it but complain.
		if oldDeltas == nil {
			klog.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; ignoring", id, oldDeltas, obj)
			return nil
		}
		klog.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; breaking invariant by storing empty Deltas", id, oldDeltas, obj)
		f.items[id] = newDeltas
		return fmt.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; broke DeltaFIFO invariant by storing empty Deltas", id, oldDeltas, obj)
	}
	return nil
}
~~~~



~~~go
func (f *DeltaFIFO) Pop(process PopProcessFunc) (interface{}, error) {
	f.lock.Lock()
	defer f.lock.Unlock()
	for {
    // 如果队列关闭，返回错误
		for len(f.queue) == 0 {
			if f.closed {
				return nil, ErrFIFOClosed
			}

      // 进入等待状态
			f.cond.Wait()
		}
		isInInitialList := !f.hasSynced_locked()
    // 消费头部元素
		id := f.queue[0]
		f.queue = f.queue[1:]
		depth := len(f.queue)
		if f.initialPopulationCount > 0 {
			f.initialPopulationCount--
		}
		item, ok := f.items[id]
		if !ok {
			// This should never happen
			klog.Errorf("Inconceivable! %q was in f.queue but not f.items; ignoring.", id)
			continue
		}
		delete(f.items, id)
		// Only log traces if the queue depth is greater than 10 and it takes more than
		// 100 milliseconds to process one item from the queue.
		// Queue depth never goes high because processing an item is locking the queue,
		// and new items can't be added until processing finish.
		// https://github.com/kubernetes/kubernetes/issues/103789
		if depth > 10 {
      // 如果队列深度大于10，且处理一个队列项超过100毫秒，记录跟踪信息
			trace := utiltrace.New("DeltaFIFO Pop Process",
				utiltrace.Field{Key: "ID", Value: id},
				utiltrace.Field{Key: "Depth", Value: depth},
				utiltrace.Field{Key: "Reason", Value: "slow event handlers blocking the queue"})
			defer trace.LogIfLong(100 * time.Millisecond)
		}
    
    // 调用处理函数controller.config.Process --> sharedIndexInformer.HandleDeltas
		err := process(item, isInInitialList)
    // 如果处理函数返回 ErrRequeue，将该项重新添加到队列
    // addIfNotPresent加入到队列前会去重
		if e, ok := err.(ErrRequeue); ok {
			f.addIfNotPresent(id, item)
			err = e.Err
		}
		// Don't need to copyDeltas here, because we're transferring
		// ownership to the caller.
		return item, err
	}
}
~~~

