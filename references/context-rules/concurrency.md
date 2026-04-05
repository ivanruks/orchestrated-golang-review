# Context Rules: Concurrency Agent

Before analyzing diffs, read this table. When you see a trigger pattern in the diff, load the additional context described.

| Trigger in diff | What to load | How (MCP operation) | Why |
|---|---|---|---|
| `go func` or `go methodCall` | Entire file containing the goroutine | `get_file_contents` (check `files/` first) | Understand goroutine lifecycle — is there a done channel, context, WaitGroup? |
| `sync.Mutex` or `sync.RWMutex` field in struct | All methods of that struct | `get_file_contents` for the file | Check lock ordering across methods — detect deadlock potential |
| Write to map or slice from multiple goroutines | All functions writing to the same variable | `get_file_contents` + search for variable name | Verify synchronization — maps are not goroutine-safe |
| `sync.WaitGroup` | Full function + goroutines it spawns | `get_file_contents` | Check Add/Done/Wait balance — Add must happen before goroutine starts |
| Channel creation (`make(chan ...)`) | Full function and all readers/writers of the channel | `get_file_contents` | Check for deadlock: unbuffered channel with single goroutine, or nobody reads |
| HTTP handler modifying package-level variable | Package-level var declaration + all handlers | `get_file_contents` for the file | Each HTTP handler runs in its own goroutine — shared state needs sync |
| `select` statement | All cases and the channel sources | `get_file_contents` | Check for missing `default` or missing `ctx.Done()` case |
| Loop variable used in goroutine (pre-Go 1.22) | go.mod for Go version | `get_file_contents` for go.mod | If Go < 1.22, loop var capture is a race condition |
