---
title: Golang Interface
date : 2023-12-10 23:00
category : [Study, Golang]
tags : [golang, 인터페이스, duck typing]
---

Golang의 문법중 부족했던 부분들만 따로 요약하여 포스팅하려한다.  
이번 포스팅의 주제는 Interface다.

## 👉Interface  
구체화된 객체 (Concrete Object)가 아닌 추상화된 상호작용으로 `관계`를 표현  
> 구체화된 객체 ? `구현`이 있는 객체. 즉, Method의 내용물이 있는 객체

``` golang
type DuckInterface interface {
    // 메서드 집합
}
```

택배 배송을 위해 Fedex API를 사용하고 있다.  
``` go
// Fedex 패키지 ----------->
type FedexSender struct {
    // 내용물 ..
}

func (f *FedexSender) Send (parcel string) {
    fmt.Printf("Fedex sends %s parcel \n", parcel)
}

// <-----------Fedex 패키지 

// main 패키지 ----------->
func SendBook (name stirng, sender *fedex.FedexSender) {
    sender.Send(name)
}

func main() {
    sender := &fedex.FedexSender{}
    SendBook("어린왕자", sender)
    SendBook("golang", sender)
}
// <-----------main 패키지 
```

이렇게 구축되어있는 상황에서 Fedex보다 우체국이 더 조건이 좋아서 변경하려하면, 기존 코드를 변경해야만 한다. (산탄총 코드)  

이런 문제는 interface를 사용하여 쉽게 변경할 수 있다.  

``` go
// koreaPost 패키지 ----------->
type PostSender struct {
    // 내용물 ..
}

func (f *PostSender) Send (parcel string) {
    fmt.Printf("Post sends %s parcel \n", parcel)
}
```
```go
// main 패키지 ----------->
// 배송 interface 선언
type Sender interface {
    Send(parcel string)
}

func SendBook(name string, sender Sender) {
    sender.Send(name)
}

func main() {
    /* 
    추후 sender의 타입만 fedex, koreaPost 등의 다른 택배사 구조체로 선언해주면 
    변경을 빠르게 할 수 있다.
    */
    var sender Sender = &fedex.FedexSender{}
    SendBook("어린왕자", sender)
    SendBook("golang", sender)
}
```
FedexSender, PostSender 모두 Send() 가 구현되어있으므로 interface를 받을 수 있다.  

---

### 👉추상화
내부 동작(구현)을 감춰서 서비스 제공자와 사용자 모두에게 자유를 주는 방식  
택배를 Fedex로 보내든, 우체국으로 보내든 내부 발송 과정을 자세히 몰라도 보낼 수 있듯이 추상화를 하면 의존성을 끊을 수 있다.(decoupling)  

![](/assets/img/YY-MM/2023-12-10-23-35-34.png)  

### 👉Duck Typing  
"어떤 새를 봤는데 그 새가 오리처럼 걷고, 오리처럼 날고, 오리처럼 소리내면 나는 그 새를 오리라고 부르겠다"  

위의 Sender 인터페이스를 main 패키지에서 정의 했다.  
Java등의 언어에서는 서비스 제공 패키지내부에서 선언하고, Class를 선언할 때 어떤 Class를 상속했는지 명시해주어야 한다.
``` Java
public class FedexSender implements Sender {
    // ...
}
``` 
사용자측에서 어떤 구현이 되어있는지를 확인하고 사용하는 것이다.  
즉, 오리가 "나는 오리야"라고 밝히고, 사용자는 "오리니까 오리처럼 걷고, 오리처럼 날겠구나" 하는 것이다.  

`Duck Typing`을 제공하는 언어에서는 서비스 제공을 어떻게 하든, 내가 구현한 메서드가 포함되어 있으면 내가 정의한 인터페이스라고 부르겠다고 할 수 있다.  

``` go
// 배송 interface 선언
type Sender interface {
    Send(parcel string)
}

func SendBook(name string, sender Sender) {
    sender.Send(name)
}
```
위의 Sender 인터페이스가 추상화한 Send() 메서드를 FedexSender도, PostSender도 갖고 있으니, 두 구조체를 Sender라고 부를 수 있는 것이다.  

> 새가 스스로 오리라고 주장한다 -> Java (제공자 입장)  
> 오리처럼 울고, 오리처럼 날면 넌 오리다 -> Go (사용자 입장)  

`Duck Typing`을 통해 사용자가 필요에 따라 인터페이스를 정의해서 사용할 수 있다.  

- `python`은 스크립트 언어라 런타임 시 객체가 인터페이스를 구현하고 있는지 확인하므로 성능상 문제가 될 수 있지만, `go`는 컴파일 시 검사하므로 성능상 문제는 없다.  
- 설계 -> 구현 (폭포수 방식)이 아니므로 Agile의 Scrum에 부합한다.

---

### 👉Adapter 패턴
**서로 다른 인터페이스를 맞춰주는 패턴**  
만일 Fedex와 우체국이 서로 다른 이름, 다른 파라미터의 API를 제공한다면 interface를 어떻게 사용해야 하는가?  

구조체를 생성해 wrapping해준다.  

``` go
// Fedex 패키지 
type FedexSender struct {
    // 내용물 ..
}

func (f *FedexSender) Send (parcel string) {
    fmt.Printf("Fedex sends %s parcel \n", parcel)
}
```
``` go
// koreaPost 패키지 ----------->
type PostSender struct {
    // 내용물 ..
}

func (f *PostSender) SendPost (parcel string) {
    fmt.Printf("Post sends %s parcel \n", parcel)
}
```
``` go
// main 패키지 ----------->
type Sender interface {
    Send(parcel string)
}

func SendBook(name string, sender Sender) {
    sender.Send(name)
}

// method 맞춰주기 위해 wrapping
type SenderWrapper struct {
    postSender koreaPost.PostSender
}

func (w SenderWrapper) Send(parcel string) {
    w.postSender.SendPost(parcel)
}


func main() {
    var sender Sender
    var postSender koreaPost.postSender
    // wrapping
    sender = SenderWrapper{postSender}

    SendBook("어린왕자", sender)
    SendBook("golang", sender)
}
```

### Reference
[Tucker의 Go 언어 프로그래밍](https://www.youtube.com/watch?v=IV6zYG3GY5s&list=PLy-g2fnSzUTBHwuXkWQ834QHDZwLx6v6j&index=29)