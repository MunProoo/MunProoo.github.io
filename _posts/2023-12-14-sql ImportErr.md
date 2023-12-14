---
title: unknown driver "mysql" (forgotten import?)
date : 2023-12-14 20:00
category : [Study, Golang]
tags : [트러블 슈팅, 에러, mysql, mariaDB, gorm]
---

프로젝트를 리팩토링 하려 하는 중에 발견한 에러를 기록하려 한다.




## Gorm MariaDB 연결 에러
DB.Close , HasTable()등 편리한 기능을 제공해주는   github.com/blue1004jy/gorm 으로 gorm을 변경하는데 Open이 안되는 이슈가 있었다.

### 에러 내용
``` go
dbm.DB, err = gorm.Open("mysql", connectionString)
	if err != nil {
		fmt.Printf("%v \n", err)
		// 로그
		// 연결 실패

		return err
	}
```

```
sql: unknown driver "mysql" (forgotten import?) 
```

go.mod에 github.com/go-sql-driver/mysql도 잘 들어가 있는데 여러 doc를 보며 헤맸다..

### 💡해결
Open()하는 파일에 
``` go
import (
    _ "github.com/go-sql-driver/mysql"
)
```
해주면 된다.

import한 mysql driver의 init()을 통해 패키지 초기화작업은 하지만 다른 기능은 사용하지 않을 때 Blank Identifier ("_")를 사용해 import한다.