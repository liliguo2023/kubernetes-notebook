# 						ParallelizeUntil

````go
// options模式，设置chunkSize
type options struct {
	chunkSize int
}

type Options func(*options)

// WithChunkSize allows to set chunks of work items to the workers, rather than
// processing one by one.
// It is recommended to use this option if the number of pieces significantly
// higher than the number of workers and the work done for each item is small.
func WithChunkSize(c int) func(*options) {
	return func(o *options) {
		o.chunkSize = c
	}
}
````



````go
// ParallelizeUntil is a framework that allows for parallelizing N
// independent pieces of work until done or the context is canceled.
// workers设置并行执行的worker数量，pieces设置总的任务数，doWorkPiece具体执行的任务，opts选项
func ParallelizeUntil(ctx context.Context, workers, pieces int, doWorkPiece DoWorkPieceFunc, opts ...Options) {
	// 任务数为0 直接返回
  if pieces == 0 {
		return
	}
  // 设置选项
	o := options{}
	for _, opt := range opts {
		opt(&o)
	}
  // 每组最低1个任务
	chunkSize := o.chunkSize
	if chunkSize < 1 {
		chunkSize = 1
	}

  // 根据总任务和每组任务数，计算分组
	chunks := ceilDiv(pieces, chunkSize)
	toProcess := make(chan int, chunks)
	for i := 0; i < chunks; i++ {
		toProcess <- i
	}
	close(toProcess)

	var stop <-chan struct{}
	if ctx != nil {
		stop = ctx.Done()
	}
  // 如果worker大于任务分组，则降低worker数量
	if chunks < workers {
		workers = chunks
	}
  // 通过WaitGroup等待任务执行结束
	wg := sync.WaitGroup{}
	wg.Add(workers)
	for i := 0; i < workers; i++ {
		go func() {
			defer utilruntime.HandleCrash()
			defer wg.Done()
			for chunk := range toProcess {
        // 计算每组的任务(0-100] (100-200]
				start := chunk * chunkSize
				end := start + chunkSize
        // 如果任务数大于pices总数，调整end
				if end > pieces {
					end = pieces
				}
				for p := start; p < end; p++ {
					select {
           // 任务取消或者超时，直接返回
					case <-stop:
						return
					default:
            //执行任务
						doWorkPiece(p)
					}
				}
			}
		}()
	}
	wg.Wait()
}

func ceilDiv(a, b int) int {
	return (a + b - 1) / b
}
````

