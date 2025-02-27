# 依赖项

## 数据库依赖

### MySQL

GoBatch 需要 MySQL 数据库来存储任务执行状态和上下文信息。

#### 版本要求
- MySQL 5.7 及以上版本
- MariaDB 10.2 及以上版本

#### 数据库配置
1. **创建数据库**
```sql
CREATE DATABASE gobatch DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

2. **创建运行时表**
使用 [schema_mysql.sql](https://github.com/chararch/gobatch/blob/master/sql/schema_mysql.sql) 中的脚本创建必要的表。

#### 连接配置
```go
import (
    "database/sql"
    _ "github.com/go-sql-driver/mysql"
)

db, err := sql.Open("mysql", "user:password@tcp(127.0.0.1:3306)/gobatch?charset=utf8mb4&parseTime=true")
if err != nil {
    panic(err)
}
gobatch.SetDB(db)
```

## 外部依赖

### 必需依赖
- `github.com/go-sql-driver/mysql` - MySQL driver
- `github.com/panjf2000/ants/v2` - Goroutine pool management
- `github.com/pkg/errors` - Error handling

### 可选依赖
- `github.com/jlaffaye/ftp` - FTP文件处理支持