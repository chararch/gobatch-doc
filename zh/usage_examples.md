# 使用示例

本指南通过实际示例演示如何使用GoBatch。完整的示例代码可以在[gobatch-example/basic](https://github.com/chararch/gobatch-example/basic)目录下找到。

## 前提条件

在开始之前，请确保已满足所有[依赖项要求](dependencies.md)，特别是MySQL数据库的配置。

## 编写步骤

### 简单步骤示例

简单步骤在单个线程中执行业务逻辑：

```go
// 定义一个简单的任务函数
func mytask() {
    fmt.Println("mytask executed")
}

// 构建步骤
step1 := gobatch.NewStep("mytask").Handler(mytask).Build()
```

### 分块步骤示例

分块步骤使用读取-处理-写入模式按块处理数据。

1. **创建Reader**

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

2. **创建Processor**

```go
type myProcessor struct {}

func (r *myProcessor) Process(item interface{}, chunkCtx *gobatch.ChunkContext) (interface{}, gobatch.BatchError) {
    return fmt.Sprintf("processed-%v", item), nil
}
```

3. **创建Writer**

```go
type myWriter struct {}

func (r *myWriter) Write(items []interface{}, chunkCtx *gobatch.ChunkContext) gobatch.BatchError {
    fmt.Printf("write: %v\n", items)
    return nil
}
```

4. **构建分块步骤**

```go
step2 := gobatch.NewStep("my_step").
    Reader(&myReader{}).
    Processor(&myProcessor{}).
    Writer(&myWriter{}).
    ChunkSize(10).
    Build()
```

## 构建和运行任务

### 1. 配置数据库连接

```go
db, err := sql.Open("mysql", "user:password@tcp(127.0.0.1:3306)/gobatch?charset=utf8&parseTime=true")
if err != nil {
    panic(err)
}
gobatch.SetDB(db)
```
> 注意：需要将用户名、密码、主机和数据库名称替换为实际值。

### 2. 构建和注册任务

```go
// 使用步骤构建任务
job := gobatch.NewJob("my_job").Step(step1, step2).Build()

// 注册任务
gobatch.Register(job)
```

### 3. 运行任务

```go
// 同步运行并传入参数
params, _ := util.JsonString(map[string]interface{}{
    "rand": time.Now().Nanosecond(),
})
gobatch.Start(context.Background(), job.Name(), params)

// 或异步运行
// gobatch.StartAsync(context.Background(), job.Name(), "")
```

## 完整示例

完整示例代码可以在[gobatch-example/basic/main.go](https://github.com/chararch/gobatch-example/basic/main.go)中找到。