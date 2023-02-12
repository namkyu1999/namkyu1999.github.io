---
title: "Go 동시성 프로그래밍"
summary: Go 동시성 프로그래밍에서 새로 알게된 것들
date: 2023-02-06
tags: ["golang"]
author: "Namkyu Park"
draft: false
categories: ["book"]
---

## 시작 전에
> 해당 포스트에서는 ‘Go 동시성 프로그래밍’ 라는 도서를 읽고, 새롭게 알게된 지식들을 작성한 것입니다.
저는 한번 공부해본 분야의 서적을 읽을 경우, 하나하나 세세하게 읽기 보다는 저자의 노하우나 팁을 위주로 읽는 편입니다.
---

## 동시성이 어려운 이유
- 대부분의 데이터 레이스는 개발자가 문제를 순차적으로 생각하기 때문에 나타납니다. 개발자들은 어떤 한 줄의 코드가 다른 코드보다 먼저 나타나기 때문에 먼저 실행될 것이라고 가정합니다.
- 프로그래밍 언어에서 제공하는 sleep 함수는 데이터 레이스의 확률을 줄일 뿐 근본적으로 해결하는 것이 아닙니다.
- 무언가가 원자적 이라는 것은, 동작하는 컨텍스트 내에서 나누어지거나 중단되지 않는 다는 것을 의미합니다.
- 어떤 컨텍스트에서 원자적인 것이 다른 컨텍스트에서는 아닐 수도 있습니다.
- 불가분(indivisible)과 중단 불가(uninterruptible)은 모두 해당 컨텍스트 내에서는 해당 요소 외에 어떤 것도 동시에 이루어지지 않는다는 것을 의미합니다.
- 무언가가 원자적이라면 암묵적으로 동시에 실행되는 컨텍스트들 내에서는 안전하다는 것을 의미합니다.

## 코드 모델링
- 동시성은 코드의 속성이고, 병렬 처리는 실행 중인 프로그램의 속성입니다.
- 코어가 하나인 기기에서 병렬로 작성한 코드를 실행하면, 해당 코드들이 병렬로 실행되는 것 같지만 사실은 cpu 컨텍스트가 전환되며 시간을 잘게 쪼개어 각 작업에 분배됩니다.
- 많은 언어는 보통 os 스레드 및 메모리 접근 동기화 수준에서 언어의 추상화 체인을 끝냅니다. Go는 다른 방식을 사용해 고루틴 및 채널의 개념으로 이를 대체합니다.
- go의 많은 부분이 CSP를 중심으로 설계되었습니다.
> CSP는 다른 포스트에서 집중적으로 리뷰할 예정입니다.
- 동시성에 대한 go의 철학은 다음과 같이 요약 가능합니다. '단순화를 목표로 하고, 가능하면 채널을 사용하며, 고루틴을 무한정 쓸 수 있는 자원처럼 다루어야 한다'

## Go에서 동시성을 다루기 위한 기본 구조
- 고루틴은 os스레드가 아닙니다. 언어의 런타임에 의해 관리되는 스레드인 그린 스레드도 아닙니다. 고루틴은 코루틴이라 불리는 더 높은 수준의 추상화입니다.코루틴은 단순히 동시에 실행되는 서브루틴으로서, 비선점적(인터럽트 불가)합니다. (os 가 작업을 중단시키지 않음)
- 고루틴은 자신의 일시 중단 지점이나 재진입 지점을 정의하지 않습니다. go의 런타임은 고루틴의 실행 시 동작을 관찰 해, 고루틴이 멈춰서 대기중일 때 자동으로 일시중단 시키고, 대기가 끝나면 다시 시작시킵니다. 런타임이 이런 식으로 고루팀을 선점가능하게 하지만, 고루틴이 멈춰있는 지점에서만 선점 가능합니다.
- 고루틴을 호스팅하는 go의 메커니즘은 M:N 스케줄러를 구현한 것으로, M개의 그린 스레드를 N개의 os 스레드에 매핑한다는 의미입니다. 이후 고루틴은 스레드에 스케줄링 됩니다. 
- go언어는 fork-join모델이라는 동시성 모델을 따릅니다. 이는 부모의 실행지점 어디서든 fork를 통해 자식 분기를 만들 수 있고, 이후 자식 분기에서 부모로 다시 join될 수 있음을 의미합니다.
- 아래와 같이 고루틴은 자신이 생성된 곳과 동일한 주소 공간에서 실행되기 때문에 원본을 이용한다.
```go
func main(){
	var wg sync.WaitGroup
	salutation := "hello"
	wg.Add(1)
	go func(){
	    defer wg.Done()
		salutation = "welcome"
    }
	wg.Wait()
	fmt.Println(salutation) // welcome이 출력된다.
}
```
- 아래 예제에서 고루틴은 문자열 타입을 갖는 반복문의 변수 salutation에 대해 닫혀있는 클로저를 실행하는 중입니다. 
  - 일반적인 실행환경에서는 고루틴을 실행 하기 전에 해당 for문이 종료될 확률이 높습니다. 즉 salutation함수가 for문을 벗어났음을 의미합니다.
  - Go의 메모리 방식을 알 수 있는데, Go 런타임에서 salutation에 대한 참조가 여전히 이루어지고 있음을 파악하고 고루틴이 계속 접근할 수 있도록 메모리를 heap으로 옮깁니다
  - 이를 해결하고 싶다면 값의 사본을 전달하면 됩니다. (go언어는 값에 의한 참조를 채택하고 있습니다.)
```go
func main(){
	var wg sync.WaitGroup
	for _, salutation := range []string{"hello", "world", "nice"}{
	    wg.Add(1)
		go func(){
			defer wg.Done()
			fmt.Println(salutation) // nice가 세번 출력된다.
        }()
        wg.Add(1)
		go func(salutation string){
		    defer wg.Done()
			fmt.Println(salutation) // 우리가 기대하는데로 한번씩 출력된다.
        }(salutation)
    }
	wg.Wait()
}
```
- 새롭게 만들어진 고루틴에는 몇 kb의 메모리가 주어지며, 충분치 않을 경우 런타임에서 스택을 저장하기 위한 메모리를 늘리거나 줄입니다.
- 동시에 실행되는 프로세스가 너무 많으면 프로세스 사이의 컨텍스트 스위칭에 모든 cpu시간을 소모하느라 실제 작업을 수행하지 못할 수 있습니다. os 수준에서 스레드를 사용하면 이로 인해 비용이 발생됩니다. 하지만 소프트웨어에서의 컨텍스트 스위칭은 훨씬 더 저렴하며 go는 이방식을 채택합니다.
- WaitGroup은 동시에 수행된 연산의 결과를 신경 쓰지 않거나, 결과를 수집할 다른 방법이 있는 경우 동시에 수행될 연산 집합을 기다릴 때 유용합니다.
- 고루틴은 언제 스케줄링 될지 확신할 수 없기 때문에, wg.Add는 외부에서 선언해야 한다.
- Cond: 고루틴들이 대기하거나, 어떤 이벤트의 발생을 알리는 집결 지점(rendezvous point)
```go
func main() {
    // sync.Cond could useful in situations where multiple readers wait for the shared resources to be available.
    c := sync.NewCond(&sync.Mutex{})
    queue := make([]interface{}, 0, 10)
    removeFromQueue := func(delay time.Duration) {
        time.Sleep(delay)
        c.L.Lock()
        queue = queue[1:]
        fmt.Println("removed from queue")
        c.L.Unlock()
        c.Signal()
    }
    for i := 0; i < 10; i++ {
        c.L.Lock()
    for len(queue) == 2 {
        c.Wait()
    }
    fmt.Println("adding to queue")
    queue = append(queue, struct{}{})
    go removeFromQueue(1 * time.Second)
        c.L.Unlock()
    }
}

// 이런식으로 pool을 생성해두면 사전 로딩을 통해 다른 객체에 대한 참조를 가져오는 데 걸리는 시간을 아낄 수 있다.
// 높은 처리 성능의 네트워크 서버를 이용하는 경우 많이 사용한다.
func main() {
    myPool := &sync.Pool{
    New: func() interface{} {
        fmt.Println("creating new instance")
            return struct{}{}
        },
    }
    instance := myPool.Get()
    myPool.Put(instance) // 인스턴스를 다시 풀로 되돌려놓음. 사용가능한 인스턴스 1개 증가
    myPool.Get()
}
```
- '<-' 해당 연산자에서 두개의 값을 받을 수 있는데, 이경우 두번째 파라미터는 '해당 채널이 열려있는지'여부입니다.
- 닫힌 채널에서도 메세지를 수신 할 수 있습니다.
- 채널과 range를 사용하게 된다면, 채널이 닫힐 때 자동으로 for 문을 탈출합니다.
```go
func main() {
	intStream := make(chan int)
	go func() {
		defer close(intStream)
		for i := 1; i <= 5; i++ {
			intStream <- i
		}
	}()
	for integer := range intStream {
		fmt.Println(integer)
	}
}
```
- 채널을 소유한 고루틴은 반드시 다음을 수행해야 합니다.
  - 채널을 인스턴스화 한다.
  - 쓰기를 수행하거나 다른 고루틴으로 소유권을 넘긴다.
  - 채널을 닫는다(일기 전용으로 만든다)
  - 이 목록에 있는 항목들을 캡슐화 하고 이를 읽기 채널을 통해 노출시킨다.
- 해당 원칙에 따라 코드를 작성하면 시스템에 대해 추론하기가 훨씬 쉬워진다.
- select 블록은 채널 중 하나가 준비됐는지 확인하기 위해 모든 채널 읽기와 쓰기를 동시에 고려한다.
- select의 case는 무작위로 선택된다.

## Go의 동시선 패턴
- for-select 패턴
```go
func main() {
	for { // 무한 반복 또는 특정 범위에 대한 반복
		select {
		// 채널에 대한 작업
		}
	}
	for _, s := range []string{"a","b","c"}{
		select{
		case <-done:
			return 
		case stringStream <-s:
		}
	}
	for {
		select{
		case <-done:
			return
		default:
		    // 선점 불가능한 작업 수행
		}
	}
}
```
- 일반적으로 동시에 실행되는 프로세스 들은 프로그램의상태에 대해 완전한 정보를 가지고있는 프로그램의 다른 부분으로 에러를 보내야 하며, 그래야 보다 많은 정보를 바탕으로 무엇을 해야할 지 지정할 수 있습니다.
- context는 done 처럼 시스템 전체를 흐른다.
```go
func main() {
	generator := func(done <-chan interface{}, integers ...int) <-chan int {
		intStream := make(chan int, len(integers))
		go func() {
			defer close(intStream)
			for _, i := range integers {
				select {
				case <-done:
					return
				case intStream <- i:
				}
			}
		}()
		return intStream
	}

	multiply := func(done <-chan interface{}, intStream <-chan int, multiplier int) <-chan int {
		multipliedStream := make(chan int)
		go func() {
			defer close(multipliedStream)
			for i := range intStream {
				select {
				case <-done:
					return
				case multipliedStream <- i * multiplier:
				}
			}
		}()
		return multipliedStream
	}

	add := func(done <-chan interface{}, intStream <-chan int, additive int) <-chan int {
		addedStream := make(chan int)
		go func() {
			defer close(addedStream)
			for i := range intStream {
				select {
				case <-done:
					return
				case addedStream <- i + additive:
				}
			}
		}()
		return addedStream
	}

	done := make(chan interface{})
	defer close(done) // 해당 작업을 통해 고루틴이 반환되어 메모리 누수를 방지한다.

	intStream := generator(done, 1, 2, 3, 4)
	pipeline := multiply(done, // 파이프라인의 각 단계가 동시에 실행된다.
		add(done,
			multiply(done, intStream, 2),
			1),
		2)

	for v := range pipeline {
		fmt.Println(v)
	}
}
```
