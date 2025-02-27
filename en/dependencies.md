# Dependencies

## Database Dependencies

### MySQL

GoBatch requires MySQL database to store job execution states and context information.

#### Version Requirements
- MySQL 5.7 or higher
- MariaDB 10.2 or higher

#### Database Setup
1. **Create Database**
```sql
CREATE DATABASE gobatch DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

2. **Create Runtime Tables**
Use the script in [schema_mysql.sql](https://github.com/chararch/gobatch/blob/master/sql/schema_mysql.sql) to create necessary tables.

#### Connection Configuration
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

## External Dependencies

### Required Dependencies
- `github.com/go-sql-driver/mysql` - MySQL driver
- `github.com/panjf2000/ants/v2` - Goroutine pool management
- `github.com/pkg/errors` - Error handling

### Optional Dependencies
- `github.com/jlaffaye/ftp` - FTP file processing support