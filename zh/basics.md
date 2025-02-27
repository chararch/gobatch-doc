# 基础知识

## 核心概念

### Job（任务）
Job代表一个完整的批处理任务。它由一个或多个按特定顺序执行的Step组成。每个Job都有唯一的名称，可以配置各种参数和监听器。Job的详细说明参见[Job](job.md)。

### Step（步骤）
Step是Job中的一个独立处理单元。GoBatch支持三种类型的步骤：
- **简单步骤**：在单个线程中执行一个任务
- **分块步骤**：以块为单位处理数据（读取-处理-写入模式）
- **分区步骤**：将大任务拆分为多个子任务并行处理

关于Step的详细说明，参见[Step](step.md)。

### JobInstance（任务实例）
JobInstance表示Job的一次逻辑运行，由Job名称和任务参数唯一标识。如果执行失败，同一个JobInstance可能会创建多个JobExecution。

### JobExecution（任务执行）
JobExecution表示JobInstance的一次执行尝试。每次执行都会跟踪其状态、开始时间、结束时间和执行结果。

### StepExecution（步骤执行）
StepExecution表示JobExecution中某个Step的一次执行尝试。它包含步骤的执行状态和结果信息。

