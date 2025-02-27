# Advanced Usage Examples

This guide demonstrates advanced usage of GoBatch through a complete loan processing example. The complete example code can be found in the [gobatch-example/file_usage](https://github.com/chararch/gobatch-example/file_usage) directory.

## Task Overview

This example implements a complete loan processing batch task, including the following steps:
1. Import loan transaction data from TSV file to database table
2. Generate repayment plans based on transaction data
3. Generate statistics summary of repayment plans and export to CSV file
4. Upload the statistics file to FTP server

## Prerequisites

Before getting started, please ensure all [dependency requirements](dependencies.md) are met, especially the MySQL database configuration.

## Database Setup

1. Create necessary tables:

```sql
CREATE TABLE t_trade (
    id bigint NOT NULL AUTO_INCREMENT,
    trade_no varchar(64) NOT NULL,
    account_no varchar(60) NOT NULL,
    type varchar(20) NOT NULL,
    amount decimal(10,2) NOT NULL,
    terms int(11) NOT NULL,
    interest_rate decimal(10,6) NOT NULL,
    trade_time datetime NOT NULL,
    status varchar(10) NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uniq_trade_no (trade_no)
);

CREATE TABLE t_repay_plan (
    id bigint NOT NULL AUTO_INCREMENT,
    account_no varchar(60) NOT NULL,
    loan_no varchar(64) NOT NULL,
    term int(11) NOT NULL,
    principal decimal(10,2) NOT NULL,
    interest decimal(10,2) NOT NULL,
    init_date date NOT NULL,
    repay_date date NOT NULL,
    repay_state varchar(10) NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uniq_loan_no_term (loan_no, term)
);
```

## Data Models

1. **Trade Model**

```go
type Trade struct {
    TradeNo      string    `order:"0" header:"trade_no"`
    AccountNo    string    `order:"1" header:"account_no"`
    Type         string    `order:"2" header:"type"`
    Amount       float64   `order:"3" header:"amount"`
    Terms        int       `order:"4" header:"terms"`
    InterestRate float64   `order:"5" header:"interest_rate"`
    Status       string    `order:"6" header:"status"`
    TradeTime    time.Time `order:"7" header:"trade_time"`
}
```

2. **Repayment Plan Model**

```go
type RepayPlan struct {
    AccountNo  string
    LoanNo     string
    Term       int
    Principal  float64
    Interest   float64
    InitDate   time.Time
    RepayDate  time.Time
    RepayState string
}
```

3. **Statistics Model**

```go
type RepayPlanStats struct {
    Term       int
    TotalPrincipal  float64
    TotalInterest   float64
}
```

## File Processing Components

### 1. Configure File Models

```go
var tradeFile = file.FileObjectModel{
    FileStore:     &file.LocalFileSystem{},
    FileName:      "res/trade.data",
    Type:          file.TSV,
    Encoding:      "utf-8",
    Header:        false,
    ItemPrototype: &Trade{},    // 文件记录自动映射到交易数据模型
}

var statsFileExport = file.FileObjectModel{
    FileStore:     &file.LocalFileSystem{},
    FileName:      "res/{date,yyyyMMdd}/stats.csv",
    Type:          file.CSV,
    Encoding:      "utf-8",
    Checksum:      file.MD5,
    ItemPrototype: &RepayPlanStats{},   // 文件记录自动映射到统计结果模型
}

var ftp = &file.FTPFileSystem{
	Hort:        "localhost",
	Port:        21,
	User:        "yourftpuser",
	Password:    "yourftppassword",
	ConnTimeout: time.Second * 10,
}

var copyFileToFtp = file.FileMove{
	FromFileName:  "res/{date,yyyyMMdd}/stats.csv",
	FromFileStore: &file.LocalFileSystem{},
	ToFileStore:   ftp,
	ToFileName:    "stats/{date,yyyyMMdd}/stats.csv",
}
var copyChecksumFileToFtp = file.FileMove{
	FromFileName:  "res/{date,yyyyMMdd}/stats.csv.md5",
	FromFileStore: &file.LocalFileSystem{},
	ToFileStore:   ftp,
	ToFileName:    "stats/{date,yyyyMMdd}/stats.csv.md5",
}
```

### 2. Implement Data Handlers 

#### 1. Import Transaction Data to Database

```go
type tradeImporter struct {
    db *sql.DB
}

func (p *tradeImporter) Write(items []interface{}, chunkCtx *gobatch.ChunkContext) gobatch.BatchError {
    for _, item := range items {
        trade := item.(*Trade)
        _, err := p.db.Exec("INSERT INTO t_trade(trade_no, account_no, type, amount, terms, interest_rate, trade_time, status) values (?,?,?,?,?,?,?,?)",
            trade.TradeNo, trade.AccountNo, trade.Type, trade.Amount, trade.Terms, trade.InterestRate, trade.TradeTime, trade.Status)
        if err != nil {
            return gobatch.NewBatchError(gobatch.ErrCodeDbFail, "插入交易记录失败", err)
        }
    }
    return nil
}
```

#### 2. Generate Repayment Plans from Transaction Data

Read transaction records from t_trade:

```go
import (
	"database/sql"
	"fmt"
	"github.com/chararch/gobatch"
)

type tradeReader struct {
	db *sql.DB
}

func (h *tradeReader) Open(execution *gobatch.StepExecution) gobatch.BatchError {
	return nil
}
func (h *tradeReader) Close(execution *gobatch.StepExecution) gobatch.BatchError {
	return nil
}
func (h *tradeReader) ReadKeys() ([]interface{}, error) {
	rows, err := h.db.Query("select id from t_trade")
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var result []interface{}
	var id int64
	for rows.Next() {
		err = rows.Scan(&id)
		if err != nil {
			return nil, err
		}
		result = append(result, id)
	}
	return result, nil
}
func (h *tradeReader) ReadItem(key interface{}) (interface{}, error) {
	id := int64(0)
	switch r := key.(type) {
	case int64:
		id = r
	case float64:
		id = int64(r)
	default:
		return nil, fmt.Errorf("key type error, type:%T, value:%v", key, key)
	}
	rows, err := h.db.Query("select trade_no, account_no, type, amount, terms, interest_rate, trade_time, status from t_trade where id = ?", id)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	trade := &Trade{}
	if rows.Next() {
		err = rows.Scan(&trade.TradeNo, &trade.AccountNo, &trade.Type, &trade.Amount, &trade.Terms, &trade.InterestRate, &trade.TradeTime, &trade.Status)
		if err != nil {
			return nil, err
		}
	}

	return trade, nil
}
```

Calculate and generate repayment plans:

```go
import (
	"database/sql"
	"github.com/chararch/gobatch"
	"time"
)

type repayPlanHandler struct {
	db *sql.DB
}

func (h *repayPlanHandler) Process(item interface{}, chunkCtx *gobatch.ChunkContext) (interface{}, gobatch.BatchError) {
	trade := item.(*Trade)
	plans := make([]*RepayPlan, 0)
	restPrincipal := trade.Amount
	for i := 1; i <= trade.Terms; i++ {
		principal := restPrincipal / float64(trade.Terms-i+1)
		interest := restPrincipal * trade.InterestRate / 12
		repayPlan := &RepayPlan{
			AccountNo:  trade.AccountNo,
			LoanNo:     trade.TradeNo,
			Term:       i,
			Principal:  principal,
			Interest:   interest,
			InitDate:   time.Now(),
			RepayDate:  time.Now().AddDate(0, 1, 0),
			RepayState: "",
			CreateTime: time.Now(),
			UpdateTime: time.Now(),
		}
		plans = append(plans, repayPlan)
		restPrincipal -= principal
	}
	return plans, nil
}

func (h *repayPlanHandler) Write(items []interface{}, chunkCtx *gobatch.ChunkContext) gobatch.BatchError {
	for _, item := range items {
		plans := item.([]*RepayPlan)
		for _, plan := range plans {
			_, err := h.db.Exec("INSERT INTO t_repay_plan(account_no, loan_no, term, principal, interest, init_date, repay_date, repay_state) values (?,?,?,?,?,?,?,?)",
				plan.AccountNo, plan.LoanNo, plan.Term, plan.Principal, plan.Interest, plan.InitDate, plan.RepayDate, plan.RepayState)
			if err != nil {
				return gobatch.NewBatchError(gobatch.ErrCodeDbFail, "insert t_repay_plan failed", err)
			}
		}
	}
	return nil
}
```

#### 3. Generate Statistics CSV File from Repay Plans

```go   
import (
	"database/sql"
	"fmt"
	"github.com/chararch/gobatch"
)

type statsHandler struct {
	db *sql.DB
}

func (h *statsHandler) Open(execution *gobatch.StepExecution) gobatch.BatchError {
	return nil
}
func (h *statsHandler) Close(execution *gobatch.StepExecution) gobatch.BatchError {
	return nil
}
func (h *statsHandler) ReadKeys() ([]interface{}, error) {
	rows, err := h.db.Query("select distinct(term) as term from t_trade")
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var result []interface{}
	var id int64
	for rows.Next() {
		err = rows.Scan(&id)
		if err != nil {
			return nil, err
		}
		result = append(result, id)
	}
	return result, nil
}
func (h *statsHandler) ReadItem(key interface{}) (interface{}, error) {
	term := int64(0)
	switch r := key.(type) {
	case int64:
		term = r
	case float64:
		term = int64(r)
	default:
		return nil, fmt.Errorf("key type error, type:%T, value:%v", key, key)
	}
	rows, err := h.db.Query("select sum(principal) as total_principal, sum(interest) as total_interest from t_repay_plan where term = ?", term)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	stats := &RepayPlanStats{}
	if rows.Next() {
		err = rows.Scan(&stats.Term, &stats.TotalPrincipal, &stats.TotalInterest)
		if err != nil {
			return nil, err
		}
	}

	return stats, nil
}

func (ss *statsHandler) Process(item interface{}, chunkCtx *gobatch.ChunkContext) (interface{}, gobatch.BatchError) {
	return item, nil
}

```

## Build And Run Job

```go
func buildAndRunJob() {
    // Initialize database
    sqlDb := openDB()
    gobatch.SetDB(sqlDb)
    gobatch.SetTransactionManager(gobatch.NewTransactionManager(sqlDb))

    // Build steps
    step1 := gobatch.NewStep("import_trade").
        ReadFile(tradeFile).
        Writer(&tradeImporter{sqlDb}).
        Partitions(10).
        Build()

    step2 := gobatch.NewStep("gen_repay_plan").
        Reader(&tradeReader{sqlDb}).
        Handler(&repayPlanHandler{sqlDb}).
        Partitions(10).
        Build()

    step3 := gobatch.NewStep("stats").
        Reader(&statsHandler{sqlDb}).
        Processor(&statsHandler{sqlDb}).
        WriteFile(statsFileExport).
        Partitions(2).
        Build()

    step4 := gobatch.NewStep("upload_file_to_ftp").
        CopyFile(copyFileToFtp, copyChecksumFileToFtp).
        Build()

    // Build and register job
    job := gobatch.NewJob("accounting_job").
        Step(step1, step2, step3, step4).
        Build()
    gobatch.Register(job)

    // Run job with parameters
    params, _ := util.JsonString(map[string]interface{}{
        "date": time.Now().Format("2006-01-02"),
        "rand": time.Now().Nanosecond(),
    })
    gobatch.Start(context.Background(), job.Name(), params)
}
```

## Job Flow Explanation

1. **Import Transaction Data (step1)**
   - Read transaction data from TSV file
   - Import data to t_trade table
   - Process in parallel using 10 partitions

2. **Generate Repayment Plans (step2)**
   - Read transaction records from database
   - Calculate repayment plans
   - Save plans to t_repay_plan table
   - Use parallel processing

3. **Export Statistics Data (step3)**
   - Aggregate repayment data
   - Calculate statistics
   - Write results to CSV file

4. **Upload Statistics to FTP (step4)**
   - Upload statistics CSV file and checksum file to FTP server