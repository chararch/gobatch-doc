# Basics

## Core Concepts

### Job
A Job represents a complete batch processing task. It consists of one or more Steps that are executed in a specific sequence. Each Job has a unique name and can be configured with various parameters and listeners. For detailed information about Jobs, see [Job](job.md).

### Step
A Step is a single phase in a Job that encapsulates an independent unit of processing. GoBatch supports three types of steps:
- **Simple Step**: Executes a single task in one thread
- **Chunk Step**: Processes data in chunks (read-process-write pattern)
- **Partition Step**: Splits a large task into multiple sub-tasks for parallel processing

For detailed information about Steps, see [Step](step.md).

### JobInstance
A JobInstance represents a logical run of a Job, uniquely identified by the Job name and job parameters. Multiple JobExecutions may be created for a single JobInstance in case of failures.

### JobExecution
A JobExecution represents a single attempt to execute a JobInstance. Each execution tracks its status, start time, end time, and execution results.

### StepExecution
A StepExecution represents a single attempt to execute a Step within a JobExecution. It contains information about the step's execution status and results.
