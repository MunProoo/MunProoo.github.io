---
title: Golang Channel
date : 2023-12-12 00:12
category : [Study, Golang]
tags : [golang, channel, goroutine]
---

이번 포스팅의 주제는 Channel이다.

## 👉Channel
채널은 고루틴끼리 메세지를 전달할 수 있는 메세지 큐이다.  
멀티 스레드환경에서 Lock없이 쓸 수 있다! (Thread-Safe)  

``` go
var messages chan string = make(chan string, 3)
messages <- "This is a message" // 송신
var msg string = <- messages // 수신
```

![](/assets/img/YY-MM/2023-12-12-00-42-44.png)  
채널이 무한히 기다리게하지 않으려면 크기를 지정한다.  
``` go

func main() {
    // 무한히 기다리는 코드
	// ch := make(chan int)

    // 정상 종료 가능한 코드
	ch := make(chan int, 2)

	go square()
	ch <- 9
	fmt.Println("Never Print")
}

func square() {
	for {
		time.Sleep(2 * time.Second)
		fmt.Println("Sleep")
	}
}
```

`for range문`을 사용하면 채널의 값을 받으려 무한 대기를 한다.
``` go 
func main() {
	ch := make(chan int, 2)

	var wg sync.WaitGroup

	wg.Add(1)
	go square(&wg, ch)
	for i := 0; i < 10; i++ {
		ch <- i * 2
	}
    // close(ch)
	wg.Wait()
}

func square(wg *sync.WaitGroup, ch chan int) {
	for n := range ch {
		fmt.Println("Square:", n)
		time.Sleep(time.Second)
	}
	wg.Done()
}
```
위 코드를 실행 시 좀비 고루틴이 되어 무한 대기를 하게 된다.  
main 고루틴에서 ch 입력을 다 하고 close(ch)로 닫아준다.  
> **성능에 영향을 주고, 코드 상으로 찾기 어려우니 조심하자!**


### Select문
여러 채널에서 동시에 데이터를 기다릴 때 사용  
Select문은 한번 실행 후 종료되므로 보통 무한 루프와 같이 사용
``` go
func main() {
	ch := make(chan int, 2)

	var wg sync.WaitGroup

	wg.Add(1)
	go square(&wg, ch)
	for i := 0; i < 10; i++ {
		ch <- i * 2
	}
	wg.Wait()
}

func square(wg *sync.WaitGroup, ch chan int) {
	tick := time.Tick(time.Second)
	terminate := time.After(10 * time.Second)

	for {
		select {
		case <-tick:
			fmt.Println("Tick") // 1초 간격으로 실행
		case <-terminate:
			fmt.Println("Terminated!") // 10초 후에는 종료
			wg.Done()
		case n := <-ch:
			fmt.Println("Square:", n*n) // 1초마다 ch에서 메세지 수신
			time.Sleep(time.Second)
		}
	}
}
```

### 채널로 생산자/소비자 패턴 구현
``` go
type Car struct {
	Body  string
	Tire  string
	Color string
}

var wg sync.WaitGroup
var startTime = time.Now()

func main() {
	tireCh := make(chan *Car)  // 차체 생산 후 타이어 공정으로 송신
	paintCh := make(chan *Car) // 타이어 조립 후 도색 공정으로 송신

	fmt.Printf("Start Factory\n")

	wg.Add(3)
	go MakeBody(tireCh)
	go InstallTire(tireCh, paintCh)
	go PaintCar(paintCh)

	wg.Wait()
	fmt.Println("Close the Factory")
}

func MakeBody(tireCh chan *Car) {
	tick := time.Tick(time.Second)
	after := time.After(10 * time.Second)
	for {
		select {
		case <-tick:
			// make a body
			car := &Car{}
			car.Body = "Sports Car"
			tireCh <- car
		case <-after:
			close(tireCh)
			wg.Done()
			return
		}
	}
}

func InstallTire(tireCh, paintCh chan *Car) {
	for car := range tireCh {
		time.Sleep(time.Second)
		car.Tire = "Winter tire"
		paintCh <- car
	}
	wg.Done()
	close(paintCh)
}

func PaintCar(paintCh chan *Car) {
	for car := range paintCh {
		time.Sleep(time.Second)
		car.Color = "Red"
		duration := time.Since(startTime)
		fmt.Printf("%.2f Complete Car : %s - %s - %s \n", duration.Seconds(), car.Body, car.Tire, car.Color)
	}
	wg.Done()
}
```

결과 (컨베이어 벨트 시스템이라 첫 생산은 3초, 이후 1초마다 생산)
```
Start Factory
3.00 Complete Car : Sports Car - Winter tire - Red 
4.00 Complete Car : Sports Car - Winter tire - Red 
5.00 Complete Car : Sports Car - Winter tire - Red 
6.00 Complete Car : Sports Car - Winter tire - Red 
7.00 Complete Car : Sports Car - Winter tire - Red 
8.00 Complete Car : Sports Car - Winter tire - Red 
9.00 Complete Car : Sports Car - Winter tire - Red 
10.00 Complete Car : Sports Car - Winter tire - Red 
11.00 Complete Car : Sports Car - Winter tire - Red 
12.00 Complete Car : Sports Car - Winter tire - Red 
Close the Factory
```
---

### 컨텍스트(Context)
작업을 지시할 때 작업가능 시간, 작업 취소 등의 조건을 지시할 수 있는 작업 명세서 역할  
``` go
var wg sync.WaitGroup

func main() {
	wg.Add(1)
	// context는 항상 background를 기본으로 한다.
	// 추가하고싶은 기능을 Wrapping하여 사용한다.
	ctx, cancel := context.WithCancel(context.Background())

	go PrintEverySecond(ctx)
	time.Sleep(5 * time.Second)
	cancel()

	wg.Wait()
}

func PrintEverySecond(ctx context.Context) {
	tick := time.Tick(time.Second)
	for {
		select {
		case <-ctx.Done():
			fmt.Println("Canceled Context")
			wg.Done()
			return
		case <-tick:
			fmt.Println("Tick")
		}
	}
}
```
context에 value도 넣을 수 있다. 
> 단, string, 기본 자료형을 넣으면 타 패키지와 중복되어 문제를 일으킬 수 있다고 에디터가 불평하니 value는 생성하여 사용한다.

``` go
var wg sync.WaitGroup

type val struct {
	ID int
}

func main() {
	wg.Add(1)
	// context는 항상 background를 기본으로 한다.
	// 추가하고싶은 기능을 Wrapping하여 사용한다.
    ctx, cancel := context.WithCancel(context.Background())
    ctx = context.WithValue(ctx, val{}, 9)

	go square(ctx) // 9를 그냥 넣어도 되지만 컨텍스트 사용이 된다.
	time.Sleep(1 * time.Second)

	wg.Wait()
}

func square(ctx context.Context) {
	if v := ctx.Value(val{}); v != nil {
		n := v.(int)
		fmt.Printf("Square : %d", n*n)
	}
	wg.Done()
}
```
---

### 채널로 구독, 발행 패턴 
![](/assets/img/YY-MM/2023-12-12-01-43-00.png)  
Publisher가 
- TopicA에 대해 이벤트 발생하면 구독자 1,2에 알림  
- TopicB에 대해 이벤트 발생하면 구독자 2,3에 알림  

``` go
// publisher.go 
package main

import (
	"context"
)

type Publisher struct {
	ctx         context.Context
	subscribeCh chan chan<- string // 단방향 채널(넣기만 가능) 타입의 채널 (채널자체를 메세지로 받는다)
	publishCh   chan string
	subscribers []chan<- string // 단방향 채널(읽기만 가능)
}

func NewPublisher(ctx context.Context) *Publisher {
	return &Publisher{
		ctx:         ctx,
		subscribeCh: make(chan chan<- string),
		publishCh:   make(chan string),
		subscribers: make([]chan<- string, 0),
	}
}

// 구독자 증가
func (p *Publisher) Subscribe(sub chan<- string) {
	p.subscribeCh <- sub
}

// 이벤트 발행
func (p *Publisher) Publish(msg string) {
	p.publishCh <- msg
}

func (p *Publisher) Update() {
	for {
		select {
		case sub := <-p.subscribeCh:

			p.subscribers = append(p.subscribers, sub)
		case msg := <-p.publishCh:
			// 발행자의 이벤트에 대해 구독자들에게 전달
			for _, subscriber := range p.subscribers {
				subscriber <- msg
			}
		case <-p.ctx.Done():
			wg.Done()
			return
		}
	}
}
```

``` go 
// subscriber.go
package main

import (
	"context"
	"fmt"
)

type Subscriber struct {
	ctx   context.Context
	name  string
	msgCh chan string
}

func NewSubscriber(name string, ctx context.Context) *Subscriber {
	return &Subscriber{
		ctx:   ctx,
		name:  name,
		msgCh: make(chan string),
	}
}

func (s *Subscriber) Subscribe(pub *Publisher) {
	pub.Subscribe(s.msgCh)
}

func (s *Subscriber) Update() {
	for {
		select {
		case msg := <-s.msgCh:
			fmt.Printf("%s got Message:%s\n", s.name, msg)
		case <-s.ctx.Done():
			wg.Done()
			return
		}
	}
}
```
``` go 
// main.go 
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

var wg sync.WaitGroup

func main() {
	ctx, cancel := context.WithCancel(context.Background())

	wg.Add(4)
	publisher := NewPublisher(ctx)
	subscriber1 := NewSubscriber("AAA", ctx)
	subscriber2 := NewSubscriber("BBB", ctx)

	go publisher.Update()

	subscriber1.Subscribe(publisher)
	subscriber2.Subscribe(publisher)

	go subscriber1.Update()
	go subscriber2.Update()

	go func() {
		tick := time.Tick(time.Second * 2)
		for {
			select {
			case <-tick:
				publisher.Publish("Hello Message")
			case <-ctx.Done():
				wg.Done()
				return
			}
		}
	}()

	fmt.Scanln()
	cancel()

	wg.Wait()
}
```

위 코드는 Publisher가 구독자들에게 끊임없이 "Hello Message"를 보내고, 구독자는 해당 메세지를 받아 콘솔에 출력한다. 콘솔에 무언가 입력하면 context를 취소하고, waitGroup이 Done되어 프로세스가 종료된다.  

💡 고루틴과 채널을 이용해 무궁무진한 방법으로 동시성 프로그래밍을 설계할 수 있다.  

