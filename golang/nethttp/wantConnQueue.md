````go
type wantConnQueue struct {
	// This is a queue, not a deque.
	// It is split into two stages - head[headPos:] and tail.
	// popFront is trivial (headPos++) on the first stage, and
	// pushBack is trivial (append) on the second stage.
	// If the first stage is empty, popFront can swap the
	// first and second stages to remedy the situation.
	//
	// This two-stage split is analogous to the use of two lists
	// in Okasaki's purely functional queue but without the
	// overhead of reversing the list when swapping stages.
	head    []*wantConn				
	headPos int							// 数据消费索引
	tail    []*wantConn
}

````



````go
// len returns the number of items in the queue.
// 返回队列长度
func (q *wantConnQueue) len() int {
	return len(q.head) - q.headPos + len(q.tail)
}

// pushBack adds w to the back of the queue.
// 加到tail队列
func (q *wantConnQueue) pushBack(w *wantConn) {
	q.tail = append(q.tail, w)
}
````



````go
// popFront removes and returns the wantConn at the front of the queue.
// 消费数据
func (q *wantConnQueue) popFront() *wantConn {
  // 第一次消弹出或者head队列数据已经消费完成时，需要交换队列
	if q.headPos >= len(q.head) {
	// 如果head队列和tail队列都没有内容可以消费，直接返回
		if len(q.tail) == 0 {
			return nil
		}
		// Pick up tail as new head, clear tail.
		// 交换head和tail队列，重置headPos
		q.head, q.headPos, q.tail = q.tail, 0, q.head[:0]
	}
  // 消费head队列的第一个元素
	w := q.head[q.headPos]
	q.head[q.headPos] = nil
  // 消费位置后移
	q.headPos++
	return w
}
````



````go
// peekFront returns the wantConn at the front of the queue without removing it.
// 返回将要消费的数据
func (q *wantConnQueue) peekFront() *wantConn {
  // 如果head队列有数据没消费直接返回
	if q.headPos < len(q.head) {
		return q.head[q.headPos]
	}
  // 如果head队列没数据了，tail队列有数据，返回tail队列的第一个数据
	if len(q.tail) > 0 {
		return q.tail[0]
	}
	return nil
}
````



````go
// cleanFront pops any wantConns that are no longer waiting from the head of the
// queue, reporting whether any were popped.
// 从队列弹出不再waiting的数据,这个需要结合wantConn去看
func (q *wantConnQueue) cleanFront() (cleaned bool) {
	for {
		w := q.peekFront()
		if w == nil || w.waiting() {
			return cleaned
		}
		q.popFront()
		cleaned = true
	}
}
````

