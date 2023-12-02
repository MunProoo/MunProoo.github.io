---
title: Struct의 Field zero value 지정
date : 2023-12-02 16:55
category : [Study, Go]
tags : [go, golang, struct, zero value]
---

Struct의 Filed에 타입 별로 zero value를 지정해야 하는 경우가 생겼다.

golang에선 reflect 패키지를 활용하여 해당 작업을 할 수 있어 테스트 코드를 기록해놓으려 한다.

``` go
type user struct {
	Name string `json:"name"`
	Age  int32  `json:"age"`
}

func LoopObjectField() *user {
	initUser := &user{}

	fmt.Println(reflect.TypeOf(initUser))
	element := reflect.ValueOf(initUser).Elem()
	fieldNum := element.NumField()

	for i := 0; i < fieldNum; i++ {
		v := element.Field(i)

		kind := v.Kind()
		
		if kind == reflect.String {
			v.SetString("defaultString")
		} else if kind == reflect.Int32 {
			v.SetInt(-1)
		}
	}

	return initUser
}
```

Field의 Kind를 통해 Type을 확인하고, Set 메서드를 통해 원하는 zero value를 입력해준다.