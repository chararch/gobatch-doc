# Configuration Guide

## Global Settings

### Database Configuration

GoBatch needs a database to store job and step execution contexts, so you must set up the database connection before running any job.

```go
gobatch.SetDB(sqlDb)
```

### Transaction Manager

For chunk steps, you must register a TransactionManager instance with GoBatch. The transaction manager interface is defined as:

```go
type TransactionManager interface {
    BeginTx() (tx interface{}, err BatchError)
    Commit(tx interface{}) BatchError
    Rollback(tx interface{}) BatchError
}
```

If you have set up the database but haven't set a transaction manager, GoBatch will create a default transaction manager instance for you.

### Concurrency Control

GoBatch uses internal task pools to run jobs and steps. You can set the maximum concurrency using:

```go
// Set maximum running jobs (default: 10)
gobatch.SetMaxRunningJobs(100)

// Set maximum running steps (default: 1000)
gobatch.SetMaxRunningSteps(5000)
```