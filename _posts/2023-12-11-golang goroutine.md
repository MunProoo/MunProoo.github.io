---
title: Golang Go Routine
date : 2023-12-11 20:00
category : [Study, Golang]
tags : [golang, goroutine, thread]
---

Golang의 문법중 부족했던 부분들만 따로 요약하여 포스팅하려한다.  
이번 포스팅의 주제는 Go Routine이다.

## 👉Thread
- 스레드는 프로세스의 흐름을 의미한다.  
- 하나의 프로세스에 여러 개의 스레드를 가지면 멀티 스레드라 한다. 
  - CPU 코어 1개에 스레드를 빠르게 전환해가며 연산하는 것.(커널 레벨)  
- 스레드 전환에는 비용이 발생한다. (Context Switching)  

---
## Go Routine 
고루틴은 OS 스레드를 사용하는 경량 스레드이다.  
메인함수도 고루틴이다. (메인 고루틴)  
![](/assets/img/YY-MM/2023-12-11-23-11-50.png)  
OS 스레드 1개당 고루틴이 할당되고, 할당된 고루틴이 대기상태(`시스템콜`이 일어나는 경우)가 되면, OS 스레드에서 고루틴을 전환하며 동시성으로 동작한다.(큐 형식)  
OS단에서 생기는 컨텍스트 스위칭보다, **스레드에서 고루틴을 전환하며 생기는 컨텍스트 스위칭 비용이 현저히 적다.**

> **시스템 콜** : 네트워크, 파일 읽기/쓰기 등 OS 자원 사용하는 기능 호출  
> **컨텍스트 스위칭**이 일어날 때 현재 스레드의 스택 메모리를 OS 가상 메모리에 복사해온다. 고루틴은 기본적인 스택 메모리가 적다. 또한 스택 메모리를 복사해올 필요가 없어서(이미 스레드에 담겨있으니까) 컨텍스트 스위칭 비용이 적은듯!  

``` go
// go 루틴 생성은 함수 앞에 go를 붙인다.
go funcName
```

### 서브 고루틴이 종료될 때까지 대기
``` go
var wg sync.WaitGroup
wg.Add(3) // 작업 개수 설정
wg.Done() // 작업이 완료되면 호출 (고루틴안에서)
wg.Wait() // Add된 수만큼 Done이 호출될 때까지 대기
```

### 💡동시성 프로그래밍의 주의점
동일한 메모리 자원을 여러 고루틴에서 접근할 때 동시성 문제 발생.  
이전에 포스팅한 **Mutual Exclusion** **(Mutex)** 를 이용하여 제어한다.  

Mutex 문제점
1. 동시성 프로그래밍으로 인한 **성능향상을 얻을 수 없다.**  
과도한 Lock으로 인해 성능이 하락하기도 한다.  
2. 고루틴을 완전히 잠들게 하는 **데드락** 문제 발생  
어디서 발생한건지 알아채기도 힘듦......  

``` go
var wg sync.WaitGroup

func main() {
	rand.New(rand.NewSource(time.Now().UnixNano()))

	wg.Add(2)
	fork := &sync.Mutex{}
	spoon := &sync.Mutex{}

	go diningProblem("A", fork, spoon, "포크", "수저")
	go diningProblem("B", spoon, fork, "수저", "포크")
	wg.Wait()
}

func diningProblem(name string, first, second *sync.Mutex, firstName, secondName string) {
	for i := 0; i < 100; i++ {
		fmt.Printf("%s 밥을 먹으려 합니다\n", name)
		first.Lock()
		fmt.Printf("%s %s 획득 \n", name, firstName)
		second.Lock()
		fmt.Printf("%s %s 획득 \n", name, secondName)

		fmt.Printf("%s 밥을 먹는다 \n", name)
		time.Sleep(time.Duration(rand.Intn(1000)) * time.Millisecond)
	}
	wg.Done()
}
```
결과 
``` 
A 밥을 먹으려 합니다
A 포크 획득 
A 수저 획득 
A 밥을 먹는다 
B 밥을 먹으려 합니다
A 밥을 먹으려 합니다
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [semacquire]:
```
위의 예제에선 모든 고루틴이 대기상태에 빠져 프로그램이 종료되었지만, 다른 고루틴이 하나라도 살아있으면 프로그램은 종료되지 않는다.😱  

따라서 뮤텍스는 `작은 범위 내에서` 매우매우 조심히 사용해야 한다.  

### 또 다른 자원관리 기법
1. 영역을 나누는 방법  
   1. 칠판을 4등분하여 각각의 영역에서만 고루틴이 돌도록 제어
2. 역할을 나누는 방법  
   1. 차체생산, 바퀴조립, 도색 등 역할을 나누어 제어 (채널)




