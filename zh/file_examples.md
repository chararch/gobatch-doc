# 高级使用示例

本指南通过一个完整的贷款处理示例演示GoBatch的高级用法。完整的示例代码可以在[gobatch-example/file_usage](https://github.com/chararch/gobatch-example/file_usage)目录下找到。

## 任务功能概述

本示例实现了一个完整的贷款处理批处理任务，主要包含以下步骤：
1. 从TSV文件导入贷款交易数据到数据库表
2. 基于交易数据生成还款计划
3. 统计汇总还款计划数据并生成CSV文件
4. 将统计结果文件上传到FTP服务器

## 前提条件

在开始之前，请确保已满足所有[依赖项要求](dependencies.md)，特别是MySQL数据库的配置。

## 数据库设置

1. 创建必要的表：

```sql
-- 贷款交易表
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

-- 还款计划表
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

## 定义数据模型

1. **交易模型**

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

2. **还款计划模型**

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

3. **统计结果模型**

```go
type RepayPlanStats struct {
    Term       int
    TotalPrincipal  float64
    TotalInterest   float64
}
```

## 文件处理组件

### 1. 配置文件模型

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

### 2. 实现数据处理器

#### 1. 交易数据导入数据库

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

#### 2. 根据交易数据生成还款计划

从t_trade表读取交易记录：

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

根据交易数据计算并生成还款计划：

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

#### 3. 统计还款计划数据生成CSV文件

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

## 构建并运行任务

```go
func buildAndRunJob() {
    // 初始化数据库
    sqlDb := openDB()
    gobatch.SetDB(sqlDb)
    gobatch.SetTransactionManager(gobatch.NewTransactionManager(sqlDb))

    // 构建步骤
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

    // 构建并注册任务
    job := gobatch.NewJob("accounting_job").
        Step(step1, step2, step3, step4).
        Build()
    gobatch.Register(job)

    // 运行任务并传入参数
    params, _ := util.JsonString(map[string]interface{}{
        "date": time.Now().Format("2006-01-02"),
        "rand": time.Now().Nanosecond(),
    })
    gobatch.Start(context.Background(), job.Name(), params)
}
```

## 任务流程说明

1. **导入交易数据 (step1)**
   - 从TSV文件读取交易数据
   - 将数据导入t_trade表
   - 使用10个分区并行处理

2. **生成还款计划 (step2)**
   - 从数据库读取交易记录
   - 计算还款计划
   - 将计划保存到t_repay_plan表
   - 使用并行处理

3. **统计数据导出文件 (step3)**
   - 汇总还款数据
   - 计算统计结果
   - 将结果写入CSV文件

4. **上传统计结果到FTP (step4)**
   - 将统计结果CSV文件和校验文件上传到FTP服务器