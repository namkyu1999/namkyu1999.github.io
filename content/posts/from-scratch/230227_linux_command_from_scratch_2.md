---
title: "밑바닥부터 만드는 리눅스 커맨드 라인 - 2. 동시성 프로그래밍으로 리팩토링하기"
summary: 리눅스 커맨드 라인 중 하나인 grep을 구현 합니다.
date: 2023-02-27
tags: ["golang", "shell", "linux"]
author: "Namkyu Park"
draft: false
categories: ["밑바닥부터 만드는 리눅스 커맨드 라인"]
---

## 시작 전에

해당 시리즈는 두 포스트로 구성 됩니다.
- [밑바닥부터 만드는 리눅스 커맨드 라인 - 1. 분석 및 기본 구현](https://namkyu1999.github.io/posts/from-scratch/230211_linux_command_from_scratch_1/)
- 밑바닥부터 만드는 리눅스 커맨드 라인 - 2. 동시성 프로그래밍으로 리팩토링하기
  - 결과출력 로직(ResultHandler)을 고루틴으로 분리하기
  - ResultHandler를 객체지향 코드로 수정하기
  - 여러 파일 검색 시 고루틴으로 작업하기


이번 시간에는 지난번에 만들었던 gorep을 동시성 & 객체지향 관점에서 리팩토링 해보겠습니다.
> 해당 포스트는 go언어로 구현된 grep의 성능 향상 버전인 [sift](https://github.com/svent/sift)를 참고하여 만들었습니다.

---
## 결과출력 로직(ResultHandler)을 고루틴으로 분리하기

이전에 gorep은 모든 파일들을 순차적으로 검색하여 pattern에 맞는 문자열이 있는지 검사하였습니다. 파일의 수가 많지 않다면 문제가 없지만, 검색해야하는 파일의 수가 많다면 성능이 좋지 않을 것입니다.

그러므로 검색을 하는 로직과 결과를 처리(화면에 출력)하는 로직을 분리하고 각각의 과정을 고루팅으로 처리하여 동시성 프로그래밍으로 구현해보겠습니다. 결과를 처리하는 로직은 앞으로 ResultHandler 라고 칭하겠습니다. 검색 로직에서 pattern에 맞는 문자열 발견 시 channel로 데이터를 넘겨줍니다. ResultHandler는 해당 채널에서 데이터를 수신하여 결과를 출력해줍니다.

기존에 작성했던 `main.go`함수를 다음과 같이 수정합니다.

```go
func main() {
	config, err := setup()
	if err != nil {
		fmt.Errorf(err.Error())
		os.Exit(2)
	}

	// 새롭게 추가 
	resultsChannel := make(chan *Result)
	wg := sync.WaitGroup{}

	wg.Add(1)
	go resultHandler(resultsChannel, &wg)
	//

	/*
	기존 로직
	*/

	// 마지막줄에 추가
	close(resultsChannel)
	wg.Wait()
	//
}
```

`resultsChannel` 이라는 이름으로 결과를 수신하는 channel을 생성하였습니다. ResultHandler가 모든 동작을 하기까지 기다리기 위한 wait group을 생성합니다. 이후 resultHandler함수를 실행합니다. 당연하게도 ide에 에러가 나올 것입니다. 이제 관련함수를 작성해보겠습니다. `cmd/gorep/output.go` 파일에 다음의 내용을 작성합니다.

```go
type Result struct {
	fileName string
	matches  []Match
}

type Match struct {
	lineNumber int
	line       string
}

func resultHandler(results chan *Result, wg *sync.WaitGroup) {
	defer wg.Done()
	for result := range results {
		sort.Slice(result.matches, func(i, j int) bool {
			return result.matches[i].lineNumber < result.matches[j].lineNumber
		})

		for _, match := range result.matches {
			fmt.Printf("%s:%d|%s\n", result.fileName, match.lineNumber+1, match.line)
		}
	}
}
```
resultHandler 함수입니다. go 언어에서는 channel을 range로 for문을 사용할 시 channel의 데이터가 생길 때마다 for문을 동작시킬 수 있습니다[[1]](https://gobyexample.com/range-over-channels). 문자 검색 시 검색된 line 하나를 `Match` 라 하였고 결과를 묶은 것을 `Result`라 하였습니다. 하나의 파일에 대한 검색 완료 시 `Result` 를 송신하게 되고 이를 수신하여 화면에 출력합니다. channel을 for loop으로 사용 할때 데이터 수신 전까지 다음 동작을 기다립니다. 또한 channel을 close() 함수를 통해 닫으면 더이상 데이터가 송신되지 않기 때문에 for 문을 벗어날 수 있습니다. 이는 다른 로직에서 해당 channel을 닫아줘야 함을 의미합니다.

이제 `search.go`파일을 변경해보겠습니다. 우선 search 함수만 변경해보겠습니다.

```go
func search(filename, pattern string, resultsChannel chan *Result) {
	file, err := os.Open(filename)
	if err != nil {
		log.Fatalf(cannotReadFile, err)
	}

	defer file.Close()
	fileScanner := bufio.NewScanner(file)

	lineNumber := 0
	
	result := &Result{
		fileName: file.Name(),
		matches:  make([]Match, 0),
	}
	for fileScanner.Scan() {
		line := fileScanner.Text()
		if index := strings.Index(line, pattern); index > -1 {
			result.matches = append(result.matches, Match{lineNumber: lineNumber, line: line})
		}
		lineNumber++
	}
	resultsChannel <- result
	if err := fileScanner.Err(); err != nil {
		log.Fatalf(errorWhileReadFile, err)
	}
}
```

이전에 pattern 을 찾았을 때 바로 출력하는 것과 구분되어 이제는 `resultsChannel`로 데이터를 송신합니다. 이외에는 달라지는 것이 없습니다. 메인함수에서 함수의 인자값만 바꾸어주면 프로그램은 정상적으로 동작합니다.

```go
// main 함수
search(fileName, config.pattern, resultsChannel)
```

저희는 현재 검색을 search와 search count로 구분하고있습니다. 둘의 차이는 결과를 출력하는 것에 있습니다. 즉 검색 부분은 동일하게 가져가고 result handler만 적절히 바꿔주면 좀더 개선된 코드를 작성할 수 있습니다. 다음 챕터에서 기술하겠습니다.

## ResultHandler를 객체지향 코드로 수정하기
go언어에서는 interface 자료형을 제공합니다. 흔히 사용하는 java와 유사하게 객체지향 프로그래밍을 할 수 있습니다. java의 interface는 명시적입니다. 즉 누구든 interface를 구현하려면 명시적으로 `implements 인터페이스 이름` 을 추가해줘야 합니다. 하지만 go언어에서는 별도로 해당 인터페이스를 구현했다는 명시를 하지 않고 인터페이스가 가지고 있는 메서드를 모두 구현하면 해당 인터페이스를 구현했다고 할 수 있습니다. 이를 참고하여 `output.go`에 다음 내용을 작성합니다.

```go
const normalResultFormat = "%s:%d|%s\n"
const countResultFormat = "%s|%d\n"

type Result struct {
	fileName string
	matches  []Match
}

type Match struct {
	lineNumber int
	line       string
}

type ResultHandler interface {
	handle(results chan *Result, wg *sync.WaitGroup)
}

type NormalResultHandler struct{}

func (n NormalResultHandler) handle(results chan *Result, wg *sync.WaitGroup) {
	defer wg.Done()
	for result := range results {
		sort.Slice(result.matches, func(i, j int) bool {
			return result.matches[i].lineNumber < result.matches[j].lineNumber
		})

		for _, match := range result.matches {
			fmt.Printf(normalResultFormat, result.fileName, match.lineNumber+1, match.line)
		}
	}
}

type CountResultHandler struct{}

func (c CountResultHandler) handle(results chan *Result, wg *sync.WaitGroup) {
	defer wg.Done()
	for result := range results {
		count := len(result.matches)
		fmt.Printf(countResultFormat, result.fileName, count)
	}
}

func NewResultHandler(isCount bool) ResultHandler {
	if isCount {
		return CountResultHandler{}
	} else {
		return NormalResultHandler{}
	}
}
```
`ResultHandler` 인터페이스는 `handle` 메서드를 통해 결과를 처리합니다. `NormalResultHandler`는 일반 문자 검색 시 문자열 그대로를 출력해주는 객체입니다. `CountResultHandler`는 match된 count의 수 요청시 결과값을 처리해주는 객체입니다. `NewResultHandler` 를 통해 객체 생성 시, 리턴 자료형이 인터페이스 이며 인자값에 따라 반환되는 객체가 달라지게 구현하였습니다. 즉 관심사를 분리하여 객체지향의 여러 요소를 만족시켰습니다. 이제 `main.go`를 수정해보겠습니다.

```go
func main(){
	now := time.Now()
	config, err := setup()
	if err != nil {
		fmt.Errorf(err.Error())
		os.Exit(2)
	}

	resultsChannel := make(chan *Result)
	resultHandler := NewResultHandler(config.count)

	resultWaitGroup := sync.WaitGroup{}
	resultWaitGroup.Add(1)

	go resultHandler.handle(resultsChannel, &resultWaitGroup)

	// 이후 동일
}
```
외부 환경 설정에 따라 resultHandler가 다르게 생성됩니다. 하지만 main함수에서는 이를 인지할 필요가 없으며 `NormalResultHandler`나 `CountResultHandler`의 세부 구현이 바뀌어도, 아니면 다른 resultHandler가 생기더라도 main함수에서는 코드를 수정할 필요가 없어지게 됩니다.

이제 마지막으로 여러 파일들을 동시에 작업하도록 구현해보겠습니다.

## 여러 파일 검색 시 고루틴으로 작업하기

 `main.go` 함수를 수정하겠습니다.

```go
func main() {
	// 이전과 동일
	
	// 변경 시작 부분
	fileWaitGroup := sync.WaitGroup{}
	for _, fileName := range config.fileName {
		fileWaitGroup.Add(1)
		go search(fileName, config.pattern, resultsChannel, &fileWaitGroup)
	}
	fileWaitGroup.Wait()

	close(resultsChannel)
	
	resultWaitGroup.Wait()
}
```
아직 `search` 함수를 고루틴이 가능하도록 구현하진 않았지만 큰그림을 보겠습니다. 이전에 `search` 함수와 `searchCount` 함수로 구분지어 실행하였는데, 이제는 객체지향적인 리팩토링을 통해 두가지를 하나로 합칠 수 있습니다.

고루틴의 경우 메인함수가 종료하게 되면 다른 모든 고루틴들도 강제로 종료됩니다. 이를 방지하고자 wait group 을 사용합니다. fileWaitGroup을 생성하여 모든 검색이 종료될 때까지 작업을 기다립니다. 모든 작업이 완료되면 resultsChannel을 close함수를 통해 닫아줍니다. resultWaitGroup은 이전에 wg로 기술하였던 resultHandler를 기다려주는 wait group입니다.

`search.go` 파일을 열어 다음을 수정합니다. 첫줄에 한가지만 추가하면 됩니다.
```go
func search(filename, pattern string, resultsChannel chan *Result, wg *sync.WaitGroup) {
	defer wg.Done()

	// 이하 동일
}
```

마무리 되었습니다. 모든 동작이 이전과 같이 동작하는 것을 보실 수 있습니다.

## 마무리

두번의 포스트를 통해서 리눅스 커맨드라인 중 하나인 grep을 구현해 보았습니다. 다음 시간이 언제가 될 지는 모르겠지만 더 좋은 포스트로 찾아뵙겠습니다.

모든 [코드는 제 깃허브](https://github.com/namkyu1999/gorep)에서 보실 수 있습니다.

## reference

[1] [Range over Channels](https://gobyexample.com/range-over-channels)
