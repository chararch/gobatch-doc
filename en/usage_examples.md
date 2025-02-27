# Usage Examples

This guide demonstrates how to use GoBatch through practical examples. The complete example code can be found in the [gobatch-example/basic](https://github.com/chararch/gobatch-example/basic) directory.

## Prerequisites

Before getting started, please ensure all [dependency requirements](dependencies.md) are met, especially the MySQL database configuration.

## Writing Steps

### Simple Step Example

A simple step executes business logic in a single thread:

```go
// Define a simple task function
func mytask() {
    fmt.Println("mytask executed")
}

// Build the step
step1 := gobatch.NewStep("mytask").Handler(mytask).Build()
```

### Chunk Step Example

A chunk step processes data in chunks using the read-process-write pattern.

1. **Create Reader**

```go
type myReader struct {}

func (r *myReader) Read(chunkCtx *gobatch.ChunkContext) (interface{}, gobatch.BatchError) {
    curr, _ := chunkCtx.StepExecution.StepContext.GetInt("read.num", 0)
    if curr < 100 {
        chunkCtx.StepExecution.StepContext.Put("read.num", curr+1)
        return fmt.Sprintf("value-%v", curr), nil
    }
    return nil, nil
}
```

2. **Create Processor**

```go
type myProcessor struct {}

func (r *myProcessor) Process(item interface{}, chunkCtx *gobatch.ChunkContext) (interface{}, gobatch.BatchError) {
    return fmt.Sprintf("processed-%v", item), nil
}
```

3. **Create Writer**

```go
type myWriter struct {}

func (r *myWriter) Write(items []interface{}, chunkCtx *gobatch.ChunkContext) gobatch.BatchError {
    fmt.Printf("write: %v\n", items)
    return nil
}
```

4. **Build Chunk Step**

```go
step2 := gobatch.NewStep("my_step").
    Reader(&myReader{}).
    Processor(&myProcessor{}).
    Writer(&myWriter{}).
    ChunkSize(10).
    Build()
```

## Building and Running a Job

### 1. Configure Database Connection

```go
db, err := sql.Open("mysql", "user:password@tcp(127.0.0.1:3306)/gobatch?charset=utf8&parseTime=true")
if err != nil {
    panic(err)
}
gobatch.SetDB(db)
```
> Note: Please replace `user`, `password`, `127.0.0.1:3306`, and `gobatch` with your actual database username, password, host:port, and database name.

### 2. Build and Register Job

```go
// Build job with steps
job := gobatch.NewJob("my_job").Step(step1, step2).Build()

// Register job
gobatch.Register(job)
```

### 3. Run Job

```go
// Run synchronously with parameters
params, _ := util.JsonString(map[string]interface{}{
    "rand": time.Now().Nanosecond(),
})
gobatch.Start(context.Background(), job.Name(), params)

// Or run asynchronously
// gobatch.StartAsync(context.Background(), job.Name(), "")
```

## Complete Example

The complete example code can be found in [gobatch-example/basic/main.go](https://github.com/chararch/gobatch-example/basic/main.go).