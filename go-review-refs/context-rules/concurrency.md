# Context Rules: Concurrency Agent

When trigger seen in diff, load context described. **How**: File Access instructions from agent prompt; search repo when noted.

| Trigger | Load | Why |
|---|---|---|
| `go func` or `go methodCall` | Entire file w/ goroutine | Goroutine lifecycle — done chan, ctx, WaitGroup? |
| `sync.Mutex`/`sync.RWMutex` field | All methods of struct | Lock ordering across methods — deadlock? |
| Write to map/slice from multiple goroutines | All funcs writing same var (search repo) | Verify sync — maps not goroutine-safe |
| `sync.WaitGroup` | Full func + spawned goroutines | Add/Done/Wait balance — Add before goroutine |
| Channel creation `make(chan ...)` | Full func + all readers/writers | Deadlock: unbuffered w/ single goroutine, or nobody reads |
| HTTP handler modifying package-level var | Package-level var + all handlers | Each handler = goroutine — needs sync |
| `select` statement | All cases + channel sources | Missing `default` or `ctx.Done()`? |
| Loop var in goroutine (pre-Go 1.22) | go.mod for Go version | Go <1.22 loop var capture = race |
| `sync.Once` | Full file — what `Do()` executes | If Do() can fail, error swallowed on retry |
| `atomic.` operations | All access sites for same var (search repo) | Mixed atomic/non-atomic = data race |
