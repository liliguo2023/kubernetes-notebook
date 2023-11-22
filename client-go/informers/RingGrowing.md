````go
// RingGrowing is a growing ring buffer.
// Not thread safe.
type RingGrowing struct {
	data     []interface{}														// 存储数据的切片
	n        int // Size of Data											// 切片长度
	beg      int // First available element						// 第一个可用元素的索引
	readable int // Number of data items available		// 记录有多少元素可用
}

// NewRingGrowing constructs a new RingGrowing instance with provided parameters.
// sharedIdexInformer 创建 Listener 时 initialSize 是1024
// 构造函数
func NewRingGrowing(initialSize int) *RingGrowing {
	return &RingGrowing{
		data: make([]interface{}, initialSize),
		n:    initialSize,
	}
}
````



~~~go
// WriteOne adds an item to the end of the buffer, growing it if it is full.
func (r *RingGrowing) WriteOne(data interface{}) {
  // 如果写满了，则扩容
	if r.readable == r.n {
		// Time to grow
    // 扩容长度是2倍
		newN := r.n * 2
		newData := make([]interface{}, newN)
		to := r.beg + r.readable
		if to <= r.n {
      // 如果没有循环写，直接拷贝没有消费 数据
			copy(newData, r.data[r.beg:to])
		} else {
      // 如果有循环写，需要分两次拷贝
			copied := copy(newData, r.data[r.beg:])
			copy(newData[copied:], r.data[:(to%r.n)])
		}
    
    // 重置beg和变更数据切片，长度计数
		r.beg = 0
		r.data = newData
		r.n = newN
	}
  // 计算新通知索引位置并写入，增加可用计数
	r.data[(r.readable+r.beg)%r.n] = data
	r.readable++
}
~~~





~~~~go
// ReadOne reads (consumes) first item from the buffer if it is available, otherwise returns false.
// 消费数据, beg 循环移动
func (r *RingGrowing) ReadOne() (data interface{}, ok bool) {
	if r.readable == 0 {
		return nil, false
	}
	r.readable--
	element := r.data[r.beg]
	r.data[r.beg] = nil // Remove reference to the object to help GC
	if r.beg == r.n-1 {
		// Was the last element
		r.beg = 0
	} else {
		r.beg++
	}
	return element, true
}
~~~~

