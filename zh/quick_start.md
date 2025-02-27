# 快速开始

本指南将帮助您快速开始使用GoBatch。

## 前提条件

在开始之前，请确保已满足所有[依赖项要求](dependencies.md)，特别是MySQL数据库的配置。

## 编写您的第一个批处理任务

在这个示例中，我们将从CSV文件读取用户数据，处理后将其写入数据库。所有的Go代码都将放在一个名为`main.go`的文件中。您可以选择直接检出[gobatch-example/quickstart](https://github.com/chararch/gobatch-example/quickstart)代码，或者按照以下步骤手动创建。

### 创建项目

首先，在您的工作目录中创建一个新的Go项目目录，例如`gobatch-quickstart`。

```bash
mkdir gobatch-quickstart
cd gobatch-quickstart
go mod init
```

### 安装 GoBatch

使用以下命令安装GoBatch：

```shell
go get -u github.com/chararch/gobatch
```

### 添加CSV文件

在项目目录中创建一个名为`users.csv`的文件，并添加以下内容：

```plaintext
id,name,email
1,John Doe,john.doe@example.com
2,Jane Smith,jane.smith@example.com
3,Bob Johnson,bob.johnson@example.com
4,Alice Brown,alice.brown@example.com
5,Charlie Davis,charlie.davis@example.com
6,David Evans,david.evans@example.com
7,Eva Foster,eva.foster@example.com
8,Frank Green,frank.green@example.com
9,Grace Harris,grace.harris@example.com
10,Hank Ingram,hank.ingram@example.com
11,Ivy Johnson,ivy.johnson@example.com
12,Jack King,jack.king@example.com
13,Kate Lewis,kate.lewis@example.com
14,Liam Miller,liam.miller@example.com
15,Mia Nelson,mia.nelson@example.com
16,Noah Owens,noah.owens@example.com
17,Olivia Parker,olivia.parker@example.com
18,Paul Quinn,paul.quinn@example.com
19,Quinn Roberts,quinn.roberts@example.com
20,Rachel Scott,rachel.scott@example.com
```

### 创建`users`表

在您的数据库中创建一个名为`users`的表，用于存储用户信息。

```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);
```

### 编写Go代码

在项目根目录下创建一个`main.go`文件，以下步骤中的所有Go代码都将在`main.go`文件中编写。

#### 导入 GoBatch

导入GoBatch包及其他必要的库。

```go
import (
    "github.com/chararch/gobatch"
    "context"
    "database/sql"
    "encoding/csv"
    "fmt"
    "os"
    _ "github.com/go-sql-driver/mysql"
)
```

#### 定义步骤

使用Reader、Processor和Writer创建步骤。

```go
// 定义读取器
type csvReader struct {
    file *os.File
    reader *csv.Reader
}

func (r *csvReader) Open(execution *gobatch.StepExecution) gobatch.BatchError {
    file, err := os.Open("users.csv")
    if err != nil {
        return gobatch.NewBatchError("CSV_OPEN_ERROR", "Failed to open CSV file", err)
    }
    r.file = file
    r.reader = csv.NewReader(file)
    _, _ = r.reader.Read() // Skip header
    return nil
}

func (r *csvReader) Read(chunkCtx *gobatch.ChunkContext) (interface{}, gobatch.BatchError) {
	record, err := r.reader.Read()
	if err != nil {
		if err == io.EOF {
			return nil, nil // End of file
		}
		return nil, gobatch.NewBatchError("CSV_OPEN_ERROR", "Failed to read file", err)
	}
	return record, nil
}

func (r *csvReader) Close(execution *gobatch.StepExecution) gobatch.BatchError {
	err := r.file.Close()
	if err != nil {
		return gobatch.NewBatchError("CSV_CLOSE_ERROR", "Failed to close CSV file", err)
	}
	return nil
}

// 定义处理器
type userProcessor struct {}

func (p *userProcessor) Process(item interface{}, chunkCtx *gobatch.ChunkContext) (interface{}, gobatch.BatchError) {
    record := item.([]string)
    return map[string]interface{}{
        "id":    record[0],
        "name":  record[1],
        "email": record[2],
    }, nil
}

// 定义写入器
type dbWriter struct {
    db *sql.DB
}

func (w *dbWriter) Write(items []interface{}, chunkCtx *gobatch.ChunkContext) gobatch.BatchError {
    for _, item := range items {
        user := item.(map[string]interface{})
        _, err := w.db.Exec("INSERT INTO users (id, name, email) VALUES (?, ?, ?)", user["id"], user["name"], user["email"])
        if err != nil {
            return gobatch.NewBatchError("DB_WRITE_ERROR", "Failed to write to database", err)
        }
    }
    return nil
}
```

#### 构建和注册任务

使用定义的步骤构建您的任务，并将其注册到 GoBatch。

```go
func main() {
    // 设置数据库连接
    // 请根据您的MySQL配置修改用户名、密码、主机和数据库名称
    db, err := sql.Open("mysql", "user:password@tcp(localhost:3306)/gobatch")
    if err != nil {
        panic(err)
    }
    gobatch.SetDB(db)

    // 构建步骤
    step := gobatch.NewStep("csvToDb").Reader(&csvReader{}).Processor(&userProcessor{}).Writer(&dbWriter{db: db}).Build()

    // 构建任务
    job := gobatch.NewJob("csvImportJob").Step(step).Build()

    // 注册任务
    gobatch.Register(job)

    // 运行任务
    gobatch.Start(context.Background(), "csvImportJob", "")
}
```

### 运行任务

使用 GoBatch 的启动函数执行您的任务。

```go
gobatch.Start(context.Background(), "csvImportJob", "")
```
> 如果你已检出[gobatch-example/quickstart](https://github.com/chararch/gobatch-example/quickstart)代码，可以执行目录下的`run.sh`脚本。

### 检查结果

在任务执行完毕后，您可以通过以下SQL查询来检查`users`表中的数据：

```sql
SELECT * FROM users;
```

这将返回所有已插入的用户数据，帮助您验证批处理任务的执行结果。


以上完整代码可以在[gobatch-example/quickstart](https://github.com/chararch/gobatch-example/quickstart)中找到。

更多详细示例，请参考[示例1](usage_examples.md)、[示例2](file_examples.md)。
