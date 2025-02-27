# Quick Start

This guide will help you get started with GoBatch quickly.

## Prerequisites

Before getting started, please ensure all [dependency requirements](dependencies.md) are met, especially the MySQL database configuration.

## Writing Your First Batch Job

In this example, we will read user data from a CSV file, process it, and write it to a database. All Go code will be placed in a single file named `main.go`. You can choose to directly check out the [gobatch-example/quickstart](https://github.com/chararch/gobatch-example/quickstart) code or manually create it step by step as follows.

### Create Project

Create a new Go project directory in your working directory, for example, `gobatch-quickstart`.

```bash
mkdir gobatch-quickstart
cd gobatch-quickstart
go mod init
```

### Installation

To install GoBatch, use the following command:

```shell
go get -u github.com/chararch/gobatch
```

### Add CSV File

Create a file named `users.csv` in the project directory and add the following content:

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

### Create `users` Table

First, create a table named `users` in your database to store user information.

```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);
```

### Write Go Code

Create a `main.go` file in the project root directory. All Go code in the following steps will be written in the `main.go` file.

#### Import GoBatch

Import the GoBatch package and other necessary libraries.

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

#### Define Steps

Create steps using readers, processors, and writers.

```go
// Define a reader
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

// Define a processor
type userProcessor struct {}

func (p *userProcessor) Process(item interface{}, chunkCtx *gobatch.ChunkContext) (interface{}, gobatch.BatchError) {
    record := item.([]string)
    return map[string]interface{}{
        "id":    record[0],
        "name":  record[1],
        "email": record[2],
    }, nil
}

// Define a writer
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

#### Build and Register Job

Build your job with the defined steps and register it with GoBatch.

```go
func main() {
    // Set up database connection
    // Please modify the username, password, host, and database name according to your MySQL configuration
    db, err := sql.Open("mysql", "user:password@tcp(localhost:3306)/gobatch")
    if err != nil {
        panic(err)
    }
    gobatch.SetDB(db)

    // Build steps
    step := gobatch.NewStep("csvToDb").Reader(&csvReader{}).Processor(&userProcessor{}).Writer(&dbWriter{db: db}).Build()

    // Build job
    job := gobatch.NewJob("csvImportJob").Step(step).Build()

    // Register job
    gobatch.Register(job)

    // Run job
    gobatch.Start(context.Background(), "csvImportJob", "")
}
```

### Run the Job

Execute your job using GoBatch's start function.

```go
gobatch.Start(context.Background(), "csvImportJob", "")
```
> If you have checked out the [gobatch-example/quickstart](https://github.com/chararch/gobatch-example/quickstart) code, you can execute the `run.sh` script in the directory.

### Check Results

After the job execution is complete, you can check the data in the `users` table using the following SQL query:

```sql
SELECT * FROM users;
```

This will return all inserted user data, helping you verify the execution results of the batch job.


The complete example code can be found in [gobatch-example/quickstart](https://github.com/chararch/gobatch-example/quickstart).

For more detailed examples, please refer to the [Example 1](usage_examples.md) and [Example 2](file_examples.md).
