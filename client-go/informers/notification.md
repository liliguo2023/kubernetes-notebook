````go
// 资源对象的更新通知
type updateNotification struct {
	oldObj interface{}										// 表示更新前的旧对象。
	newObj interface{}										// 表示更新后的新对象
}

// 资源对象的新增通知
type addNotification struct {
	newObj          interface{}						// 表示新增的对象
	isInInitialList bool									// 表示该对象是否在初始列表中
}

// 资源对象的删除通知
type deleteNotification struct {
	oldObj interface{}									// 表示删除的对象
}
````

