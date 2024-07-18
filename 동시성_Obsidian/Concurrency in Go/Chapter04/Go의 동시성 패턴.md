

지금까지 Go의 **동시성 기본 요소**(concurrency primitive)의 기반을 탐구하고, 이들 요소를 올바르게 사용하는 방법을 논의했다. 4장에서는 이 기본 요소를 패턴으로 작성해 시스템을 확장 가능하고 유지보수가 용이하도록 하는 방법에 대해 심도 깊게 다룰 것이다.


시작하기 전에 4장에 포함된 일부 패턴의 형식을 알아야 한다. 많은 예제에서 빈 인터페이스(interaface{})를 전달하는 채널을 사용할 것이다. Go에서 빈 인터페이스를 사용하는 것은 논란의 여지가 있다. 그러나 이렇게 해온 이유가 있다. 

첫 번째 이유는, 이 책의 뒷부분에 나오는 간결한 예제들을 더 쉽게 작성할 수 있다는 점이다. 두 번째 이유는, 경우에 따라서 패턴을 얻고자 하는 것을 더 잘 표현했다고 믿기 때문이다. 이 점에 관해서는 147 페이지의 "파이프라인"에서 자세히 논의할 것이다.

빈 인터페이스를 사용하는 것이 문제가 된다고 생각한다면, ==빈 인터페이스 대신 Go 생성자(generator)를 통해 필요로 하는 타입을 생성하고 활용하는 패턴을 사용할 수 있다는 점을 기억하자.==

이제 앞서 이야기한 대로 Go에서 동시성에 대한 몇 가지 패턴을 배우겠다.


---

# 제한

^3ef919

동시성 코드로 작성 할 때, 안정한 작동을 위한 몇 가지 옵션이 있다. 이 중에 둘은 벌써 다루었다.

- 메모리 공유를 위한 동기화 기본 요소
>[!예]
>sync.Mutex

- 통신을 통한 동기화
>[!예]
>채널


이 밖에서도 여러 개의 동시 프로세스에서 **암시적으로 안전한 몇 가지 옵션**이 있다.

- 변경 불가능한 데이터
- 제한(confinement)에 의해 보호되는 데이터

어떤 의미에서, 변경 불가능한 데이터는 암시적으로 동시에 실행해도 안전하기 때문에 이상적이다. 동시에 실행되는 각 프로세스는 동일한 데이터에서 동작할 수 있지만 이를 수정할 수는 없다. 새 데이터는 만들려면 원하는 대로 수정할 수 있는 새로운 복사본을 만들어야 한다.

이를 통해 개발자가 알아야만 하는 사항을 줄여줄 뿐만 아니라, ==임계 영역의 크기를 줄여 프로그램을 더 빠르게 만들어주기도 한다. Go에서는 메모리의 값을 가리키는 포인터 대신 값의 복사본을 사용하는 코드를 작성해 이 효과를 얻을 수 있다.==

일부 언어는 명시적으로 변경할 수 없는 값에 대한 포인터의 활용을 지원한다. 그러나 Go는 이러한 언어에 속하지 않는다.

==**제한**은 또한 개발자의 인지 부하를 줄여주고, 임계 영역의 크기도 줄여줄 수 있다.== 동시성 값을 제한하는 기법은 단순히 값의 복사본을 전달하는 것보다는 조금 복잡하므로 4장에서 이러한 제한 기법을 살펴보겠다.

제한은 하나의 동시 프로세스에서만 정보를 사용할 수 있도록 하는 간단하면서도 강력한 아이디어이다. 이를 달성하면 동시 프로그램은 암묵적으로 안전하며, 동기화도 필요하지 않다. 제한은 **에드 혹**(ad hoc)과 **어휘적**(lexical)이라는 두 가지 방식으로 가능하다.


**에드 혹 제한**이란, 언어의 커뮤니티나 근무하는 그룹 또는 작업하는 코드베이스에서 설정된 관례에 의해 제한이 이루어지는 경우다. ==개인적인 생각으로 누군가 코드를 작성할 때마다 정적 분석을 수행해주는 도구가 없다면 이러한 규약을 고수하기가 어렵다.==


``` go
package main

  

import (
    "fmt"
)

  

func main() {

    data := make([]int, 4)

    loopData := func(handleData chan<- int) {
        defer close(handleData)
        for i := range data {
            handleData <- data[i]
        }
    }

  

    handleData := make(chan int)
    go loopData(handleData)

  

    for num := range handleData {
        fmt.Println(num)
    }
}
```


정수 데이터 handleData 채널을 통해 loopData 함수와 루프 모두에서 사용할 수 있지만, 관례적으로 loopData 함수에서만 접근한다. 그러나 많은 사람이 코드를 건드리는 경우도 있고, 마감 시간이 닥쳐오면 실수가 일어나 제한이 깨지면서 문제가 발생할 수 있다. ==앞서 언급했듯이 정적 분석 도구는 이러한 종류의 문제를 파악할 수 있지만, Go 코드 베이스에 대한 정적 분석은 대다수 팀이 도달하지 못한 성숙도 수준을 요구한다.== 

이것이 이 책의 필자는 어휘적인 제한을 선호하는 이유이다. 어휘적인 제한은 컴파일러가 제한을 시행하도록 한다.

**어휘적 제한**은 올바른 데이터만 노출하기 위한 어휘 범위 및 이를 사용하는 여러 동시 프로세스를 위한 동시성 기본 요소와 관련이 있다. 어휘적 제한이 있다면 잘못된 작업을 일어날 수 없다. 사실 3장에서 이 주제를 이미 다뤘다. "채널"을 떠올려보자. "채널"에서는 채널의 읽기 또는 쓰기 측면만을 필요로 하는 동시 프로세스에게 노출하는 것에 대해 설명했다. 다시 예제를 살펴보자.

``` go
package main

  

import (
    "fmt"
)

  

func main() {

    chanOwner := func() <-chan int {
        results := make(chan int, 5) // #1
  

        go func() {
            defer close(results)
            for i := 0; i <= 5; i++ {
                results <- i
            }

        }()

        return results

    }

  

    consumer := func(results <-chan int) { // #2
        for result := range results {
            fmt.Printf("Received: %d\n", result)
        }
        fmt.Println("Done receiving!")
    }

    results := chanOwner() // #3
    consumer(results)

}
```

1. 여기서는 chanOwner의 어휘 범위 내부에서 채널을 인스턴스화한다. 이는 results 채널의 쓰기 측면의 범위를 그 아래에 정의된 클로저(Closure)로 제한한다. ==다시 말해, 이 채널의 쓰기 측면을 제한해 다른 고루틴들이 채널에 쓰는 것을 방지한다.==
2. 여기서는 ==채널의 읽기 측면을 받는데, 이를 consumer 내부로 전달해 consumer가 읽기 위에는 다른 작업을 하지 못하도록 할 수 있다.== 이런 제한으로 인해 main 고루틴은 이 채널의 읽기 전용 뷰로 또 다시 제한한다.
3. 여기서는 int 채널의 읽기 전용 복사본을 받는다. 읽기 접근으로만 사용하겠다고 선언해, 채널이 consume 함수 내에서 읽기용으로만 사용될 수 있도록 제한한다.


이런 식으로 설정하면 이 작은 예제에서 채널을 활용하는 것은 불가능하다. 이 예제는 제한을 소개하는 괜찮은 방법이긴 하지만, 채널들은 동시에 실행해도 안전하기 때문에 크게 흥미로운 예제는 아니다. ==동시 실행에 안전하지 않는 bytes.buffer 데이터 구조체의 인스턴스를 사용한 제한의 예를 살펴보자.==

``` go
package main

import (
    "bytes"
    "fmt"
    "sync"

)

func main() {

    printData := func(wg *sync.WaitGroup, data []byte) {
        defer wg.Done()

        var buff bytes.Buffer
        for _, b := range data {
            fmt.Fprintf(&buff, "%c", b)
        }

        fmt.Println(buff.String())

    }

  

    var wg sync.WaitGroup

    wg.Add(2)
    data := []byte("golang")
    go printData(&wg, data[:3]) // #1
    go printData(&wg, data[3:]) // #2
    wg.Wait()

  

}
```

1. 여기서는 data 구조체의 첫 3 바이트를 포함하는 슬라이스를 전달한다.
2. 여기서는 data 구조체의 마지막 3 바이트를 포함하는 슬라이스를 전달한다.

이 에제에서 printData는 data 슬라이스와 같은 클로저 내에 있지 않기 때문에 data 슬라이스에 접근할 수 있으며, 작업을 수행하기 위해 byte의 슬라이스를 인자로 받아야 함을 알 수 있다. ==여기서는 슬라이스의 서로 다른 부분을 전달해, 시작하는 고루틴들을 우리가 전달하는 각 슬라이스의 부분들로 제한한다.== 어휘 범위로 인해 잘못된 일을 할 수 없게 됐으며[^1], 따라서 메모리 접근을 동기화하거나 통신을 위해 데이터를 공유할 필요가 없다.

그래서 요점이 뭘까? 동기화를 사용할 수 있다면 왜 제한을 추구해야 하는가? **그 이유는 성능을 향상시키고 개발자의 인지 부하를 줄이기 위해서이다.** ==동기화에서는 비용이 들며, 이를 피할 수 있다면 임계 영역이 없기 때문에 동기화 비용을 지불할 필요가 없다.== 또한 ==동기화로 인해 발생 가능한 모든 문제도 막을 수 있으므로, 개발자는 이러한 문제들에 대해 전혀 걱정할 필요가 없다.==

또한 **어휘적 제한을 사용하는 동시성 코드**는 어휘적으로 제한된 변수를 사용하지 않은 동시성 코드보다 더 이해하기 쉽다는 이점이 있다. ==이는 어휘 범위의 컨텍스트 내에서 동기식 코드를 작성할 수 있기 때문이다.==


---

# for-select 루프

for-select 루프는 Go 프로그램에서 반복적으로 나타난다. 다음을 살펴보자.

``` go
for{// 무한 반복 또는 특정 범위에 대한 루프
select{
// 채널에 대한 작업
}
}
```

이 패턴이 나타날 수 있는 몇 가지 시나리오가 있다.


#### 채널에서 반복 변수 보내기

순회(iterator)할 수 있는 것을 채널의 값으로 변환하려고 하는 경우가 종종 있다. 이것은 그다지 복잡하지 않으며, 대개 다음과 같은 모양이다.

``` go
package main

  

import (
    "bytes"
    "fmt"
    "sync"
)

  

func main() {

  

    for _, sync := range []string{"a","b","c"} {
        select{
        case <-done:
            return
        case stringStream <-s:
        }
    }

  

}
```


#### 멈추기를 기다리면서 무한히 대기

멈출 때까지 무한 루프에 빠져 있는 고루틴을 생성하는 경우는 흔하게 발생한다.

이 작업은 여러 유형으로 수행할 수 있다. 어느 것을 선택하든 이는 순전히 선호하는 스타일의 차이일 뿐이다.

첫 번째 변형은 select 구문을 가능한 짧게 유지한다.

``` go
for{
select{
case <- done:
return
default:

}
// 선점 불가능한(non-preemptable) 작업 수행
}
```

done 채널이 닫히지 않는다면, select 구문을 빠져나갈 것이고, ==루프 본문의 나머지 부분을 진행할 것이다.==

두 번째 변형은 select 구문의 default 절에 작업을 포함시킨다.
``` go
for{
select{
case <- done:
return
default:
// 선점 불가능한 작업 수
}
}
```

select 구문에 들어갔을 때 done 채널이 닫히지 않았다면, default 구문을 대시 실행할 것이다. 

이 패턴은 이게 전부지만 많은 곳에서 나타나기 때문에 언급할 가치가 있다.


---
# 고루틴 누수 방지

^5a8639

[[Go의 동시성 구성 요소#^9ccee2]]고루틴에서 배웠듯이, 고루틴은 저렴하며, 손쉽게 만들 수 있다. 이런 이유로 Go는 생산적인 언어라 할 수 있겠다.

런타임은 고루틴을 ==임의의 수의 운영체제 스레드로 다중화==하며, 많은 경우에 해당 추상화 수준에 대해 염려할 필요가 없도록 처리한다. 그러나 고루틴들은 자원을 필요로 하며, 런타임에 의해 가비지 컬렉션되지 않으므로, 그들이 남기는 메모리 흔적이 얼마나 작든 그 흔적들을 우리의 프로세스에 남겨두고 싶지는 않다. 그렇다면 메모리 상의 흔적들이 정리됐는지 확인하려면 어떻게 해야 할까?

처음부터 단계별로 생각해보자. 고루틴은 왜 존재하는가? 2장에서는 고루틴이 서로 병렬로 실행될 수 있고, 그렇지 않을 수도 있는 작업의 단위를 나타낸다고 했다. 고루틴이 종료되는데 몇 가지 경로가 있다.

- 작업이 완료됐을 때
- 복구할 수 없는 에러로 인해 더 이상 작업을 계속할 수 없을 때
- 작업을 중단하라는는 요청을 받았을 때

처음 두 경로는 사용자의 알고리즘이므로 별 다른 노력 없이 도달할 수 있다. 그러나 작업 취소는 어떻게 해야 하는가? 작업 취소에는 네트워크 효과가 있기 때문에, 이 세 가지 경로 중에는 작업 취소가 가장 중요한 것으로 밝혀졌다. 
고루틴을 시작했다면, ==고루틴은 일련의 조직화된 방식으로 다른 몇몇 고루틴들과 협력할 가능성이 크다.== 심지어 이러한 상호 연관성을 그래프로 나타낼 수도 있다. ==자식 고루틴이 계속 실행해야 하는지 여부는 많은 다른 고루틴들의 상태에 대한 지식을 근거해 정할 수 있다.== 이러한 전체 문맥에 대한 지식을 가진 부모 고루틴(많은 경우 main 고루틴)은 그 자식 고루틴에게 종료하라고 말할 수 있어야 한다.

5장에서 대규모의 고루틴 상호 의존성을 계속 살펴보겠지만, 지금은 하나의 자식 고루틴이 정리되도록 고려해보겠다. 고루틴 누수의 간단한 예부터 시작해보자.


``` go
package main

  

import (
    "fmt"
)

  

func main() {

  

    doWork := func(strings <-chan string) <-chan interface{} {

        completed := make(chan interface{})

        go func() {
            defer fmt.Println("doWork exited")
            defer close(completed)
            for s := range strings {
                // 원하는 작업을 수행
                fmt.Println(s)
            }

        }()

        return completed

    }

  

    doWork(nil)
    // 여기서 추가적인 작업이 이루어질 수 있다.
    fmt.Println("Done.")
}
```

여기서는 main 고루틴이 nil 채널을 doWork로 전달하는 것을 볼 수 있다. 따라서 strings 채널은 실제로 어떠한 문자열로 쓰지 않으며, doWork를 포함하는 고루틴은 이 프로세스가 지속되는 동한 메모리가 남아 있다(심지어 doWork 및 main 고루틴 내에서 고루틴을(join)하면 데드락 상태가 발생한다).

이 예제에서 프로세스의 수명은 매우 짧지만, 실제 프로그램에서는 수명이 긴 프로그램이 시작되는 부분에서 고루틴을 쉽게 시작할 수 있다. ==최악의 경우, main 고루틴이 평생 동안 고루틴들을 계속 돌려 메모리 사용량에 영향을 미칠 것이다.==

이를 성공적으로 완하는 방법은, ==부모 고루틴이 자식 고루틴에게 취소(cancallation) 신호를 보낼 수 있도록 부모와 자식 고루틴들 사이에 신호를 설정하는 것이다.== 일반적으로 이 신호는 일반적으로 done이라는 읽기 전용 채널이다. 부모 고루틴은 이 채널을 자식 고루틴으로 전달한 다음, 자식 고루틴을 취소하려고 할 때 이 채널을 닫는다. 예제를 살펴보자.


``` go
package main

  

import (
    "fmt"
    "time"
)

  

func main() {

    doWork := func(done <-chan interface{}, strings <-chan string) <-chan interface{} { // #1

        terminated := make(chan interface{})

        go func() {
            defer fmt.Println("doWork exited.")
            defer close(terminated)
            for {
                select {
                case s := <-strings:
                    // 원하는 작업을 수행
                    fmt.Println(s)
                case <-done: // #2
                    return
                }
            }
        }()
        return terminated
    }

    done := make(chan interface{})
    terminated := doWork(done, nil)

  

    go func() { // #3
        // 1초 후에 작업을 취소한다.
        time.Sleep(1 * time.Second)
        fmt.Println("Canceling doWork goroutine...")
        close(done)
    }()

    <-terminated // #4
    fmt.Println("Done.")

}
```

1. 여기서 doWork 함수로 done 채널을 전달한다. 일반적으로 이 채널이 첫 번째 매개변수이다.
2. 여기서는 어디서든 쓰이는 for-select 패턴을 사용하고 있음을 볼 수 있다. ==case 구문 중 하나는 done 채널이 신호를 받았는지 여부를 확인하는 것이다.== ==신호를 받았다면 이 고루틴에서 리턴한다.==
3. 여기서는 1초 이상 지나면 doWork에서 생성된 고루틴을 취소할 또 다른 고루틴을 생성한다.
4. 여기서는 doWork에서 생성된 고루틴과 main 고루틴을 조인한다.

출력 결과는 다음과 같다.

``` go
PS C:\Users\kgm09\goproject\src> go run main.go
Canceling doWork goroutine... // doWork 고루틴을 취소한다...
doWork exited. // doWork 종료
Done. // 완료
```

strings  채널에 nil을 전달했음에도 불구하고, 고루틴은 여전히 성공적으로 종료됨을 알 수 있다. 앞의 예제와 달리 이 예제에서는 두 개의 고루틴을 조인하지만 데드락 상태는 일어나지 않는다. 그 이유는 두 개의 고루틴을 조인하기 전에, 세 번째 고루틴을 생성해 1초 후에 doWork 내에서 고루틴을 취소하기 때문이다. 성공적으로 고루틴 누수를 제거했다!


앞의 예제는 채널을 잘 수신하는 고루틴 경우를 다루지만 그렇지 않은 경우, 즉 ==채널에 값을 쓰려는 시도를 차단하는 고루틴의 경우는 어떨까?== 다음은 이 문제를 보여주는 간단한 예제다.

``` go
package main

  

import (
    "fmt"
    "math/rand"
)

  

func main() {

  

    newRandStream := func() <-chan int {

        randStream := make(chan int)

        go func() {
            defer fmt.Println("newRandStream closure exited.") // #1
            defer close(randStream)

            for {
                randStream <- rand.Int()
            }
        }()
        return randStream

    }
    randStream := newRandStream()
    fmt.Println("3 random ints:")

    for i := 1; i <= 3; i++ {
        fmt.Printf("%d: %d\n", i, <-randStream)
    }

}
```

1. 고루틴이 성공적으로 끝난 경우에 여기서 메시지를 출력한다.

이 코드의 실행 결과는 다음과 같다.

``` go
PS C:\Users\kgm09\goproject\src> go run main.go
3 random ints:
1: 2192360936233020603
2: 5902303455957584128
3: 6385952305647238041
```

출력 결과로부터 지연된(defer) fmt.Println 구문이 절대 실행되지 않는다는 것을 알 수 있다.

루프의 세 번째 반복 이후에, 고루틴 블록은 더 이상 읽을 수 없는 채널에 다음번 랜덤 정수를 보내려고 시도한다. 예제에서는 생산자에게 멈춰도 된다고 말할 방법이 없다. ==수신의 경우와 마찬가지로, 해결책은 생산자 고루틴에게 종료를 알리는 채널을 제공하는 것이다.==

``` go
package main

  

import (

    "fmt"
    "math/rand"
    "time"
)

  

func main() {

  

    newRandStream := func(done <-chan interface{}) <-chan int {
        randStream := make(chan int)

        go func() {
            defer fmt.Println("newRandStream closure exited.")
            defer close(randStream)
            for {
                select {
                case randStream <- rand.Int():
                case <-done:
                    return
                }
            }
        }()
        return randStream
    }

  

    done := make(chan interface{})
    randStream := newRandStream(done)
    fmt.Println("3 random ints:")

    for i := 1; i <= 3; i++ {
        fmt.Printf("%d: %d\n", i, <-randStream)
    }

    close(done)
    //진행중인 작업을 시뮬레이션한다.
    time.Sleep(1 * time.Second)

}
```

이 코드는 다음과 같이 출력된다.

``` go
PS C:\Users\kgm09\goproject\src> go run main.go
3 random ints:
1: 4714018836851891106
2: 5067401626120439271
3: 3422553270202962245
newRandStream closure exited. // newRandStream 클로저가 종료된다.
```






이번에는 고루틴이 제대로 정리됐음을 볼 수 있다.


고루틴이 누수되지 않도록 하는 방법을 알았으므로 규약을 명시할 수 있다. 다른 고루틴을 생성한 책임이 있는 고루틴은 해당 고루틴을 중지시킬 책임도 있다.

이 규약은 프로그램이 성장함에 따라 구성 가능하고 확장 가능하도록 도와준다. 147페이지의 "파이프라인" 및 187페이지의 "context패키지"에서 이 기법을 다시 살펴본다.
고루틴의 중지를 보장하는 방법은 고루틴의 타입과 목적에 따라 다를 수 있지만, 모두 done 채널을 전달하는 것을 바탕으로 구축한다.

---
# or 채널

^6dfa66

때로는 하나의 done 채널로 결합해, 그 구성 요소 중 하나의 채널이 닫힐 때 결합된 채널이 닫히도록 해야 할 경우도 있을 것이다. ==이 경우에는 이러한 결합을 수행하는 select문을 작성하는 것이 좋다. 그러나 때때로 런타임에는 작업중인 done 채널의 수를 알 수 없다.== 이러한 경우나 간단한 한 줄의 코드를 선호한다면 or 채널 패턴을 사용해 채널을 결합할 수 있다.

이 패턴은 재귀 및 고루틴들을 통해 복합 done 채널을 만든다.


``` go

package main

  

func main() {

  

    var or func(channels ...<-chan interface{}) <-chan interface{}

    or = func(channels ...<-chan interface{}) <-chan interface{} { // #1

        switch len(channels) {
        case 0: // #2
            return nil
        case 1: // #3
            return channels[0]
        }

  

        orDone := make(chan interface{})

        go func() { // #4
            defer close(orDone)
            switch len(channels) {
            case 2: // #5
                select {
                case <-channels[0]:
                case <-channels[1]:
                }
            default: // #6
                select {
                case <-channels[0]:
                case <-channels[1]:
                case <-channels[2]:
                case <-or(append(channels[3:], orDone)...): // #6
                }
            }
        }()
        return orDone
    }
}

```

1. 여기에 가변(variadic) 채널 슬라이스를 받아 하나의 채널을 리턴하는 or 함수가 있다.
2. 이것은 재귀 함수이므로 종료 기준을 설정해야 한다. 첫 번째 종료 기준은 가변 슬라이스가 비어 있으면 단순히 nil을 리턴하는 것이다. ==이것은 채널을 전달하지 않는 것과 관련이 있다.== 예제에서는 복합 채널이 무엇이든 할 것이라고 기대하지 않는다.
3. 두 번째 종료 기준은 가변 슬라이스에 하나의 요소만 있으면 해당 요소를 리턴하는 것이다.
4. 다음은 함수의 핵심 부분이며 재귀가 발생하는 곳이다. 채널들에서 차단 없이 메시지를 기다릴 수 있도록 고루틴을 생성한다.
5. 재귀 방식을 사용하고 있기 때문에, or에 대한 모든 재귀 호출은 적어도 두 개의 채널을 가지고 있다. 고루틴 수를 제한하는 최적화를 위해, 두 개의 채널에 대한 호출 또는 두 개의 채널을 가지고 있는 특별한 case를 배치한다.
6. 여기서는 슬라이스의 세 번째 인덱스 이후에 위치한 모든 채널에서 재귀적으로 or 채널을 만든 다음, 이 중에서 select를 수행한다. ==이 반복 관계는 첫 번째 신호가 리턴되는 것으로부터 트리를 형성하기 위해 나머지 **슬라이스를 or 채널들로 분해**한다.== 또한 고루틴들이 트리를 위쪽과 아래쪽 모두로 빠져나올 수 있도록 orDone 채널을 전달한다.

이는 ==여러 개의 채널을 한 개의 채널로 결합==해, ==여러 채널 중에 하나라도 닫히거나 채널에 데이터가 쓰여지면 모든 채널이 닫히도록 할 수 있는 매우 간결한 함수다.== 이 함수를 어떻게 사용할 수 있는지 살펴보자. 다음은 설정된 지속 시간 후에는 닫히는 여러 채널들을 받아서 이들을 or 함수를 사용해 단일 채널로 결합하고 이를 닫는 간단한 예제다.


``` go
package main

  

import (
    "fmt"
    "time"
)

  

func main() {

  

    var or func(channels ...<-chan interface{}) <-chan interface{}
    or = func(channels ...<-chan interface{}) <-chan interface{} { // #1

  

        switch len(channels) {
        case 0: // #2
            return nil
        case 1: // #3
            return channels[0]
        }

  

        orDone := make(chan interface{})

        go func() { // #4

            defer close(orDone)

            switch len(channels) {
            case 2: // #5
                select {
                case <-channels[0]:
                case <-channels[1]:
                }

  

            default: // #6
                select {
                case <-channels[0]:
                case <-channels[1]:
                case <-channels[2]:
                case <-or(append(channels[3:], orDone)...): // #6
                }
            }
        }()
        return orDone
    }

  

    sig := func(after time.Duration) <-chan interface{} { // #1

  

        c := make(chan interface{})

        go func() {

            defer close(c)

            time.Sleep(after)

        }()

        return c

  

    }

  

    start := time.Now() // #2

  

    <-or(sig(2*time.Hour),

        sig(5*time.Minute),

        sig(1*time.Second),

        sig(1*time.Hour),

        sig(1*time.Minute))

  

    fmt.Printf("done after %v", time.Since(start)) // #3

  

}
```

1. 이 함수는 단순히 after에 명시된 시간이 지나면 닫히는 채널은 생성한다.
2. 여기서는 or 함수로부터의 채널이 차단되는 시점을 대략적으로 추적한다.
3. 그리고 여기서 읽기가 발생하기까지 걸린 시간을 출력한다.

``` shell
PS C:\Users\kgm09\goproject\src> go run main.go
done after 1.0070629s
```


호출이 닫히기까지 서로 다른 시간이 소요되는 여러 개의 채널을 배치함에도 불구하고, 1초 후에 닫히는 채널로 인해 or 호출에 의해 생성된 전체 채널이 닫힌다는 점에 주목하라. 이는 ==or 함수가 구성한 트리 상의 위치에도 불구하고 항상 이 채널이 처음으로 닫히기 때문에, 이것의 클로저에 의존하는 채널도 역시 닫히기 때문이다.==

추가적인 고루틴들의 비용인 $f(x)=\left\lfloor x/2 \right\rfloor$의 비용으로 이 간결함을 얻을 수 있다. 여기서 x는 고루틴들의 수이다. ==그러나 Go의 강점 중 하나는 고루틴을 신속하게 생성하고 스케줄링 및 실행할 수 있다는 것이다.== Go 언어는 고루틴을 사용해 문제를 올바르게 모델링하도록 적극 권장한다. 여기에서 생성되는 고루틴의 수에 대해 걱정하는 것은 성급한 최적화일 것이다. 또한 컴파일하는 시점에 작업에 필요한 채널의 수를 모를 경우에는 done 채널을 결합할 수 있는 다른 방법이 없다.

이 패턴은 시스템의 **모듈들이 교차하는 지점에서 사용할 때 유용**하다. ==이런 교차점에서는 콜 스택을 통해 고루틴 트리를 취소할 수 있는 여러 개의 조건이 존재하곤 한다.== or 함수를 사용해 간단히 이 조건들을 결합하고 스택 아래쪽으로 전달할 수 있다. 131 페이지의 "context 패키지"에서 이 작업을 수행하는 다른 방법도 살펴볼 것이다. context 패키지는 매우 휼륭하며, 조금 더 자세한 설명을 제공한다.

또한 "복제된 요청"에서는 이 패턴의 변형을 사용해 복잡한 패턴을 형성하는 방법을 살펴보겠다.

---
# 에러처리

동시성 프로그램에서 에러 처리를 올바르게 진행하는 일은 어려울 수 있다. ==때로는 다양한 프로세스가 정보를 공유하고 조정하는 방법을 생각하는 데 많은 시간을 할애하느라 에러가 발생한 상태를 우아하게 처리하는 것을 잊어버리기도 한다.== Go는 널리 알려진 에러의 예외 모델을 피하면서 에러 처리가 중요하다는 사실을 천명했고, 프로그램을 개발할 대 알고리즘에 주의를 기울이는 것과 동일한 수준으로 에러 경로(error path)에 주의 해야 한다고 말했다. 이 개념을 바탕으로 동시에 실행되는 여러 개의 프로세스 작업할 때 어떤 식으로 에러를 처리해야 하는지 살펴보겠다.

에러 처리에 관해 생각할 때 가장 근본적인 질문은 "에러 처리의 책임자는 누구인가?"이다. 어떤 시점에서는 스택을 따라 에러를 전달하는 것을 멈추고 실제로 뭔가를 수행해야 한다. 누가 이를 책임지는가?

동시에 실행되는 프로세스라면 이 질문은 좀 더 복잡해진다. 동시에 실행되는 프로세스들은 부모 형제 프로세스와 독립적으로 작동하기 때문에 에러와 관련해 제대로 된 일이 무엇인지 추론하기가 어려울 수 있다. 이 문제로 인한 예제로 다음 코드를 살펴보자.

``` go
package main

  

import (
    "fmt"
    "net/http"

)
func main() {

    checkStatus := func(done <-chan interface{}, urls ...string) <-chan *http.Response {

        respones := make(chan *http.Response)

        go func() {
           defer close(respones)
            for _, url := range urls {

                resp, err := http.Get(url)

                if err != nil {
                    fmt.Println(err) // #1
                    continue
                }

                select {
                case <-done:
                   return
                case respones <- resp:
                }
            }
        }()
        return respones
    }

    done := make(chan interface{})
    defer close(done)

    urls := []string{"https://www.google.com", "https://badhost"}

    for Response := range checkStatus(done, urls...) {
        fmt.Printf("Response:%v\n", Response.Status)
    }
  

}
```

1. 여기서는 고루틴이 에러가 발생했음을 알리기 위해 최선을 다하고 있음을 볼 수 있다. 달리 뭘 할 수 있겠는가? 에러를 다시 돌려줄 수는 없다! 에러가 얼마나 많아야 지나치게 많은 것일까? 계속해서 요청을 하는가?

이 코드를 실행하면 다음과 같은 결과를 얻는다.

``` shell
PS C:\Users\kgm09\goproject\src> go run main.go
Response:200 OK
Get "https://badhost": dial tcp: lookup badhost: no such host
```

여기서는 고루틴이 이 문제에 대해 선택의 여지가 없다는 것을 알 수 있다. 단순히 에러를 삼켜버릴 수는 없으므로 그저 합리적으로 행동한다. 에러를 출력하고 누군가 주목해 주기를 바랄 뿐이다. 고루틴들을 이 어색한 상황에 빠뜨리지 말라. ==관심 사항을 분리할 것을 권한다.== 일반적으로 ==동시에 실행되는 프로세스들은 프로그램의 상태에 대해 완전한 정보를 가지고 있는 프로그램의 다른 부분으로 에러를 보내야 하며==, 그래야 보다 많은 정보를 바탕으로 무엇을 해야 할지 결정할 수 있다. 다음 예제는 이 문제에 대한 정확한 해결책을 보여준다.

``` go
package main

  

import (

    "fmt"

    "net/http"

)

  

type Result struct { // #1
    Error    error
    Response *http.Response
}

  

func main() {

    checkStatus := func(done <-chan interface{}, urls ...string) <-chan Result { // #2

        results := make(chan Result)
        go func() {
            defer close(results)

  

            for _, url := range urls {
                var result Result
                resp, err := http.Get(url)
                result = Result{Error: err, Response: resp} // #3

                select {
                case <-done:
                    return
                case results <- result: // #4
                }
            }
        }()
        return results
    }

  

    done := make(chan interface{})

    defer close(done)

  

    urls := []string{"https://www.google.com", "https://badhost"}

    for result := range checkStatus(done, urls...) {
        if result.Error != nil { // #5
            fmt.Printf("error: %v", result.Error)
            continue
        }

        fmt.Printf("Response %v\n", result.Response.Status)

    }

  

}
```

1. 여기서는 `*http.Response` 와 고루틴 내의 루프 반복 시에 발생할 수 있는 error 모두를 포함하는 타입을 생성한다.
2. 이 행은 ==루프 반복의 결과를 조회==하기 위해 읽어올 수 있는 채널을 리턴한다.
3. 여기서는 Error 및 Response 필드가 설정된 Result 인스턴스를 만든다.
4. 여기가 채널에 Result를 쓰는 곳이다.
5. 프로그램의 main 고루틴 내부인 바로 이곳에서, checkStatus에 의해 영리하게 시작된 고루틴에서 발생한 에러를 더 큰 프로그램의 전체 컨텍스트 내에서 처리할 수 있다.

이 코드의 출력은 다음과 같다.

``` shell
PS C:\Users\kgm09\goproject\src> go run main.go
Response 200 OK
error: Get "https://badhost": dial tcp: lookup badhost: no such host
```

여기서는 잠재적인 출력을 잠재적인 에러와 연결한 방법이 핵심이다. ==이는 checkStatus 고루틴에서 생성 가능한 결과의 전체 집합을 나타내며==, ==에러가 발생했을 때 main 고루틴이 수행할 작업을 결정할 수 있게 해 준다.== 넓은 의미에서 에러 처리의 문제를 생산자 고루틴과 성공적으로 분리했다. 이를 통해 생산자 고루틴을 생성한 고루틴(이 경우 main 고루틴)이 실행 중인 프로그램에 대해 더 많은 컨텍스트를 가지고 있으며 에러와 관련해 더 현명한 결정을 내릴 수 있게 됐다.

이전 예제에서는 단순히 에러를 표준 출력에 썼지만, 다른 작업을 할 수도 있다. ==프로그램을 조금 변경해 세 가지 이상의 에러가 발생하면== 상태를 확인하는 것을 멈추게 하자.

``` go
package main

  

import (
    "fmt"
    "net/http"
)

  

type Result struct { // #1
    Error    error
    Response *http.Response
}

  

func main() {

    checkStatus := func(done <-chan interface{}, urls ...string) <-chan Result { // #2
        results := make(chan Result)

        go func() {
            defer close(results)

  

            for _, url := range urls {
                var result Result
                resp, err := http.Get(url)
                result = Result{Error: err, Response: resp} // #3

                select {
                case <-done:
                    return
                case results <- result: // #4
                }
            }
        }()
        return results
    }

  

    done := make(chan interface{})
    defer close(done)

  

    errCount := 0
    urls := []string{"a", "https://www.google.com", "b", "c", "d"}

    for result := range checkStatus(done, urls...) {
        if result.Error != nil { // #5
            fmt.Printf("error: %v", result.Error)
            errCount++
            
            if errCount >= 3 {
                fmt.Println("Too many errors, breaking!")
                break
            }

            continue
        }
        fmt.Printf("Response %v\n", result.Response.Status)
    }

  

}
```

이 코드는 다음과 같이 출력된다.

``` shell
PS C:\Users\kgm09\goproject\src> go run main.go
error: Get "a": unsupported protocol scheme ""Response 200 OK
error: Get "b": unsupported protocol scheme ""error: Get "c": unsupported protocol scheme ""Too many errors, breaking!
```
에러가 checkStatus에서 리턴되고 해당 고루틴에서 내부적으로 처리되지 않기 때문에, 에러 처리가 익숙한 Go 패턴을 따른다는 것을 알 수 있다. 이 간단하 예제를 통해 main 고루틴이 여러 고루틴들의 결과를 조정하고 자식 고루틴을 계속하거나 취소하기 위한 보다 복잡한 규칙을 작성하는 상황을 쉽게 상상해볼 수 있다. ==여기서 다시 주목할 점은 고루틴들에서 리턴될 값을 구성할 때 에러가 일급 객체로 간주돼야 하는 것이다.== 고루틴 에러를 발생시킬 수 있는 겨우, ==이러한 에러는 결과 타입과 밀접하게 결합돼야 하며, 일반적인 동기 함수처럼 동일한 통신 회선을 통해 전달해야 한다.==

---
# 파이프라인

프로그램을 작성할 때, 무작정 않아서 하나의 긴 함수를 작성하지 않을 것이다. 부디 그러지 않기를 바란다. 당신은 함수, 구조체, 메서드 등 형태로 추상화를 구성할 것이다. 왜 그렇게 할까? 더 큰 흐름에는 중요하지 않은 세부 정보를 추상화하기 위한 목적도 있을 것이고, 다른 영역에 영향을 주지 않고 코드의 한 영역에서 작업하기 위한 목적도 있을 것이다.

어떤 시스템을 변경하고 하나의 논리적인 변화를 반영하기 위해 여러 영역을 건드려야 했던 적이 있는가? 그렇다면 그 시스템은 불완전한 추상화로 인한 어려움을 겪고 있는 것일 수도 있다.


**파이프라인**은 시스템에서 추상화를 구성하는데 사용할 수 있는 또 다른 도구다. 특히 프로그램이 스트림이나 데이터에 대한 일괄 처리(batch)작업들을 처리해야 할 대 사용하는 매우 강력한 도구이다. 파이프라인이라는 용어는 1856년에 처음으로 사용됐다고 알려있다.

컴퓨터과학에서는 이 용어를 빌려 쓰는데, 역시 한 곳에서 다른 곳으로 뭔가를 운반하기 때문이다. 파이프라인은 데이터를 가져와서, 그 데이터를 대상으로 작업을 수행하고, 결과 데이터를 다시 전달하는 일련의 작업에 불과하다. 이러한 각각의 작업을 파이프라인상의 단계(stage)라고 부른다.

파이프라인을 사용하면 각 단계의 관심사를 분리할 수 있어 많은 이점을 얻을 수 있다. 상호 독립적으로 각 단계를 수정할 수 있으며, 각 단계의 수정과 무관하게 단계들의 결합 방식을 짜맞출 수 있다. 도한 데이터 흐름상의 이전 단계 또는 다음 단계의 작업을 동시에 처리할 수 있고, 일부분을 팬 아웃하거나 속도를 제한할 수 있다. 팬 아웃은 "팬 아웃, 팬 인"에서 다루고, 속도 제한에 대해서는 5장에서 다룬다. 지금은 이 용어들이 의미하는 바를 걱정하지 않아도 된다. 우선 간단하게 시작하여 파이프라인의 단계를 생성해보자.


==이전에 언급했듯이 하나의 단계는 데이터를 가져와서 변환을 수행하고 데이터를 다시 전송하는 것이다==. 다음은 파이프라인상의 단계로 간주될 수 있는 함수이다.

``` go
    multiply := func(values []int, multiplier int) []int {
        multipliedValues := make([]int, len(values))
        for i, v := range values {
            multipliedValues[i] = v * multiplier
        }
        return multipliedValues
    }

```

이 함수는 정수의 슬라이스와 승수를 인자로 받아서 반복문을 통해 이들을 곱하고, 변환된 새 슬라이스를 리턴한다. 지루한 함수처럼 보인다. 또 다른 단계를 만들어보자.

``` go
    add := func(values []int, additive int) []int {

        addedValues := make([]int, len(values))

        for i, v := range values {
            addedValues[i] = v + additive
        }

        return addedValues

    }
```

또 하나의 지루한 함수다! 이번 함수는 그저 새로운 슬라이스를 만들고, 각 요소에 값을 더해준다. 이 시점에서 무엇이 이 두 함수를 단순한 함수가 아닌 파이프라인의 단계로 만드는 것인지 궁금할 것이다. 이 둘을 조합해보자.

``` go
    ints := []int{1, 2, 3, 4}

    for _, v := range add(multiply(ints, 2), 1) {

        fmt.Println(v)

    }
```

이 코드는 다음과 같이 출력된다.

``` go
PS C:\Users\kgm09\goproject\src> go run main.go
3
5
7
9
```

Range 절 내에서 add와 multiply를 조합하는 방식을 살펴보자. 이들은 일상에서 매일 접할 만한 함수지만, 파이프라인 단계의 속성을 갖도록 구성했기 때문에 이를 결합해 파이프라인을 구성할 수 있다.

파이프라인 단계의 특성은 무엇인가? ^4c1df8

- 각 단계는 동일한 타입을 소비하고 리턴한다.
- 각 단계는 전달될 수 있도록 언어에 의해 구체화[^2]되어야 한다. Go의 함수들은 구체화돼 있으며, 이러한 목적에 잘 맞는다.


함수형 프로그래밍에 익숙한 사람이라면 머리를 끄덕이면서 **고차 함수**와 **모나드**(monad) 같은 용어에 대해 생각할 것이다. 실제로 파이프라인의 단계는 함수형 프로그래밍과 매우 밀접하게 관련돼 있으며, 모나드의 부분 집합으로 간주될 수 있다.
여기에서 명시적으로 모나드나 함수형 프로그래밍을 깊이 다루지는 않는다. 모나드와 함수형 프로그래밍이 그 자체로 흥미롭긴 하지만, 두 주제에 대한 실무적인 지식들은 파이프라인을 이해하는데 유용할 수는 있어도 꼭 필요한 것은 아니다.

여기에서 add와 multiply 단계는 파이프라인 단계의 모든 속성을 만족시킼다. 둘 다 int의 슬라이스를 소비하고 int의 슬라이스를 리턴한다. Go는 함수들을 구체화했으므로 add와 multiply를 전달할 수 있다. 이런 특성들로 인해 파이프라인의 단계가 가지는 흥미로운 특성, 즉, 각 단계 자체를 수정하지 않고도 높은 수준에서 단계들을 쉽게 결합할 수 있다는 특성이 나타난다.

예를 들어 파이프라인에 2를 곱하는 새로운 스테이지를 추가하고자 한다면, 앞의 파이프라인을 다음과 같이 새로운 multiply 단계로 감싸면 된다.

``` go
    ints := []int{1, 2, 3, 4}

    for _, v := range add(multiply(ints, 2), 1) {
        fmt.Println(v)
    }
```

이 코드를 실행하면 다음과 같이 출력된다.

``` go
PS C:\Users\kgm09\goproject\src> go run main.go
4
6
8
10
```

새로운 함수를 작성하거나, 기존의 함수를 수정하거나, ==파이프라인 상의 결과를 수정하지 않고 어떻게 이 작업을 수행할 수 있었는지 확인해보자.== 어쩌면 파이프라인 패턴을 사용할 때는 이점을 알 수 있을 것이다. 물론 이 코드를 절차적으로도 작성할 수 있다.


``` go
package main

  

import (
    "fmt"
)

  

func main() {
    ints:=[]int{1,2,3,4}
    for _, v := range ints {
        fmt.Println(2*(v*2+1))
    }
}
```

각 단계에서 데이터 슬라이스를 가져와서 데이터 슬라이스를 리턴하는 방법을 살펴보자. 이러한 단계들은 **일괄 처리**라고 하는 것을 수행한다. ==여기서 일괄 처리란, 단지 한 번에 하나씩이산(discrete) 값을 처리하는 대신에 모든 데이터 덩어리를 한 번에 처리한다는 것을 의미한다.== **스트림 처리**를 수행하는 타입의 파이프라인 단계가 하나 더 있다. ==이는 각 단계가 한 번에 하나의 요소를 수신하고 방출한다는 것을 의미한다.==

일괄 처리와 스트림 처리에는 장단점이 있다. 하지만 지금은 원본 데이터가 변경되지 않은 상태로 남아 있기 때문에, ==각 단계에서 계산 결과를 저장하기 위해 동일한 길이의 새 슬라이스를 만들어야 한다는 것만 유의==하자. ==즉, 어떤 경우에도 프로그램의 메모리 사용량은 파이프라인을 시작할 때 사용하는 슬라이스의 크기보다 크다는 의미이다.== 작업 단계를 스트림 지향으로 변환하면 어떤 모습일지 살펴보자.

``` go
package main

  

import (
    "fmt"
)

  

func main() {

  

    multiply := func(value, multiplier int) int {
        return value * multiplier
    }

    add := func(value, additive int) int {
        return value + additive
    }

  

    ints := []int{1, 2, 3, 4}

    for _, v := range ints {
        fmt.Println(multiply(add(multiply(v, 2), 1), 2))
    }

  

}
```

각 단계가 이산 값을 수신해 방출하고, 프로그램의 메모리 사용량은 파이프라인의 입력 크기만큼으로 다시 줄어든다. 그러나 파이프라인을 for 루프를 내부로 넣고 range가 파이프라인에 데이터를 공급하는 무거운 작업을 수행하도록 만들어야 했다.

이것은 ==파이프라인에 데이터를 공급하는 방식의 재사용을 제한할 뿐만 아니라==, ==뒷부분에서 살펴보겠지만 확장성 역시 제한한다.== 또 다른 문제도 있다. ==실실적으로 루프가 반복될 때마다 파이프라인을 인스턴스화 한다.== 함수를 호출하는 것에서 큰 비용이 발생하진 않지만, 어쨌든 루프의 반복마다 세 가지 함수 호출을 수행한다. 그리고 동시성 측면에서는 어떨까? 앞서 파이프라인 활용의 이점 중 하나가 개별 단계의 동시에 처리할 수 있는 능력이었고, 팬 아웃에 대해서도 언급했다. 이것은 어디에서 비롯되는 것일까?

이러한 개념을 소개하기 위해 multiply 및 add 함수를 조금 더 확장할 수도 있지만, 이 함수들은 파이프라인의 개념을 소개하는 역할을 수행했다. 이제는 Go에서 파이프라인을 구성하기 위한 최상의 방법이 무엇인지 배우고, Go 채널의 기본 요소를 시작할 시간이다.


### 파이프라인 구축의 모범 사례

채널은 Go의 모든 기본 요구 사항을 충족하기 때문에, Go에서 파이프라인을 구성하는데 적합하다. 채널은 값을 받고 방출할 수 있고, 동시에 실행해도 안전하며, 여러 가지를 아우르고, 언어에 의해 구체화한다. 잠깐 시간을 내서 채널을 이용하도록 앞의 예제를 변경해보자.


``` go
package main

  

import (
    "fmt"
)

  

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

  

    multiply := func(
        done <-chan interface{},
        intStream <-chan int,
        multiplier int,
    ) <-chan int {

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

    add := func(
        done <-chan interface{},
        intStream <-chan int,
        additive int,
    ) <-chan int {
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
    defer close(done)

    intStream := generator(done, 1, 2, 3, 4)
    pipeline := multiply(done, add(done, multiply(done, intStream, 2), 1), 2)

    for v := range pipeline {
        fmt.Println(v)
    }

  

}
```

이 코드는 다음과 같이 출력된다.

``` go
PS C:\Users\kgm09\goproject\src> go run main.go
6
10
14
18
```

원하는 출력 값을 그대로 복사한 것처럼 보이겠지만, 코드가 훨씬 더 많이 필요하다. 우리가 얻은 것은 정확히 무엇인가? 먼저 작성한 것을 살펴보자. 우선 이제는 두 개가 아닌 세 개 함수를 가진다. ==이들 함수는 몸체 안에서 하나의 고루틴을 시작하는 것처럼 보인다.==

"[[Go의 동시성 패턴#^5a8639]]고루틴 누수 방지"에서 확인했듯이, ==채널에서 고루틴이 종료돼야 한다는 신호를 구독하는 패턴을 사용한다.== 함수들은 모두 채널을 리턴하는 것처럼 보이며, 일부는 추가 채널을 사용하는 것처럼 보인다. 흥미롭다! 더 자세히 살펴보자.


```go
done:=make(chan interface{})
defer close(done)
```

프로그램이 수행하는 첫 번째 작업은, done 채널을 만들고 defer문에서 close 호출하는 것이다. 이전에 논의했듯이, 이 작업을 통해 프로그램이 깨끗하게 종료되고 결코 고루틴이 누수되지 않도록 예방한다. 새로울 것이 없다. 다음으로 generator 함수를 살펴보자.

``` go
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
```

generator 함수는 ==가변 정수 슬라이스를 인자로 받아 입력된 정수 슬라이스와 동일한 길이를 가지는 버퍼링된 정수 채널을 생성==하고,==고루틴을 시작하고 생성된 채널을 리턴한다.== 그런 다음 생성된 고루틴에서, generator는 전달된 ==가변 슬라이스를 순회하면서 **자신이 생성한 채널로 슬라이스의 값들을 보낸다.**==

이 채널로 데이터를 전송하는 부분은 done 채널에 대한 case와 동일한 select문 안에 있다. 이 패턴은 고루틴의 누수를 막기 위해 134페이지 "[[Go의 동시성 패턴#^5a8639]] 고루틴 누수 방지"에서 배운다.



그러니까 간단히 말해서 ==generator 함수는 이산 값들의 집합을 채널의 데이터 스트림으로 변환한다==. 이런 유형의 함수를 **생성기**(generator)라고 부른다. 파이프라인을 시작할 때는 항상 채널로 변환해야 하는 데이터 뭉치가 있기 때문에, 파이프라인을 사용해 작업할 때 이 생성기를 자주 볼 수 있다. 뒤에서 재미있는 생성기 몇 가지를 살펴보겠지만, 먼저 이 프로그램에 대한 분석을 마치도록 하겠다. 다음으로 파이프라인을 생성한다.

```go
    pipeline := multiply(done, add(done, multiply(done, intStream, 2), 1), 2)

```

이 코드는 이전에 작업한 파이프라인과 동일하다. 숫자들이 스트림에 2를 곱하고 1을 더하고 그 결과에 2를 곱한다. ==이 파이프라인은 이전 예제의 함수들을 활용한 파이프라인과 유사하지만 결정적인 차이점이 있다.==

먼저 채널들을 사용하고 있다. 뻔한 차이점이지만 다음과 같은 두 가지 이유로 결정적인 차이점이라고 할 수 있다. **첫째, ==파이프라인에서 range 구문을 사용해 값을 추출할 수 있으며, 입력 및 출력이 동시에 실행되는 컨텍스트에서 안전하기 때문에 각 단계에서 안전하게 동시에 실행할 수 있다는 점이다.==**

둘째, ==파이프라인의 각 단계가 동시에 실행한다는 점==이다. 즉, 모든 단계는 입력만을 기다리며, 출력을 보낼 수 있어야 한다. 이 차이는 165페이지의 "팬 아웃, 팬 인"에서 배우겠지만, 중대한 파급 효과가 있는 것으로 밝혀졌다. 그러나 지금은 이것이 각 단계들로 하여금 특정 타임 슬라이스에서 상호 독립적으로 실행될 수 있도록 허용한다는 점에만 주목한다.

이 예에서는 range를 통해 파이프라인의 범위를 순회하면서 시스템을 통해 값을 가져온다.

``` go
for v:=range pipeline{
fmt.Println(v)
}

```

다음은 시스템의 각 없이 각 채널에 입력되는 방식과 채널이 닫히는 시점을 보여주는 표다. 반복 횟수는 수행 중인 for 루프의 반복 횟수를 0부터 카운트한 값이고, 각 열의 값은 파이프라인 단계에서의 값이다.


| 반복 횟수 | Generator | Multiply | Add  | Multiply | 값   |
| ----- | --------- | -------- | ---- | -------- | --- |
| 0     | 1         |          |      |          |     |
| 0     |           | 1        |      |          |     |
| 0     | 2         |          | 2    |          |     |
| 0     |           | 2        |      | 3        |     |
| 0     | 3         |          | 4    |          | 6   |
| 1     |           | 3        |      | 5        |     |
| 1     | 4         |          | 6    |          | 10  |
| 2     | (닫힘)      | 4        |      | 7        |     |
| 2     |           | (닫힘)     | 8    |          | 14  |
| 3     |           |          | (닫힘) | 8        |     |
| 3     |           |          |      | (닫힘)     | 18  |


고루틴들을 종료하기 위해 신호를 보내는 패턴을 사용하는 것에 대해서도 더 자세히 살펴보자. 여러 상호 의존적인 고루틴을 다룰 때 이 패턴을 어떻게 작동할까? 프로그램 실행이 끝나기 전에 done 채널에 대한 close를 호출하면 어떻게 될까?


이 질문들에 대답하기 위해서는 파이프라인의 구성을 다시 살펴보아야 한다.



``` go
    pipeline := multiply(done, add(done, multiply(done, intStream, 2), 1), 2)
```

==각 단계는 **공통의 done 채널** 및 **파이프라인의 후속 단계로 전달되는 채널**이라는 두 가지 방식으로 상호 연결된다.== 즉, multiply 함수에 의해 생성된 채널은 add 함수에 따라 전달되는 식이다. 앞의 표를 다시 살펴보고, 완료되기 전에 done 채널에 대해 close를 호출한 후 닫은 무슨 일이 일어나는지 살펴보자.



| 반복횟수        | Generator | Multiply | Add  | Multiply | 값          |
| ----------- | --------- | -------- | ---- | -------- | ---------- |
| 0           | 1         |          |      |          |            |
| 0           |           | 1        |      |          |            |
| 0           | 2         |          | 2    |          |            |
| 0           |           | 2        |      | 3        |            |
| 1           | 3         |          | 4    |          | 6          |
| close(done) | (닫힘)      | 3        |      | 5        |            |
|             |           | (닫힘)     | 6    |          |            |
|             |           |          | (닫힘) | 7        |            |
|             |           |          |      | (닫힘)     |            |
|             |           |          |      |          | (range 탈출) |

done 채널을 닫는 것이 파이프라인을 통해 어떻게 연결되는지 보이는가? 이를 통해 파이프라인의 각 단계에서 다음 두 가지가 가능해진다.

- **입력**(incoming) 채널에 대한 순회. 입력 채널이 닫히면 range가 종료된다.
- done 채널과 select 문을 공유하는 전송.


파이프라인 단계의 상태에 상관없이, 입력 채널에서 대기 중이거나, 전송 대기 중이거나, done 채널을 닫으면 파이프라인 단계는 강제로 종료될 것이다.

여기에서 재귀 관계가 발생한다. 파이프라인의 시작 부분에서 이산 값을 채널로 변환해야 한다는 점을 확고히 했다. 이 프로세스에서는 반드시 선점 가능해야 하는 두 가지 지점이 있다.

- 즉각적으로 이루어지지 않는 **이산 값의 생성**
- 해당 채널로 **이산 값 전송**


첫 번째는 당신에게 달렸다. 예제의 generator 함수에서 이산 값들은 가변 슬라이스를 아우르는 범위에서 생성된다. 즉, 선점할 필요가 없을 만큼 즉각적이다. 두 번째는 select문과 done 채널을 통해 처리되며, 이로 인해 inStream에 쓰기를 시도하는 것이 차단 된 경우에도 generator가 선점 가능하도록 한다.

파이프라인의 ==다른 쪽 끝에서는, 마지막 단계에서 선점 가능성이 보장된다는 사실을 추론을 통해 알 수 있다==. 선점 당하게 되면 range 문으로 읽고 있는 채널이 닫힐 것이고, 그 결과 range를 빠져나오게 되기 때문에 선점 가능하다. 필요한 스트림이 선점 가능하기 때문에 마지막 단계 역시 선점이 가능하다.

==파이프라인의 시작과 끝 사이에서 코드는 항상 for-range 루프로 한 채널의 내용을 순회하여==, ==done 채널을 포함하는 select 구문 내에서 다른 채널로 전송==된다.


==입력 채널에서 값을 가져오기 위해 어떤 단계가 대기 상태가 되면, 해당 채널이 닫힐 때 그 단계가 대기 해제 된다.== ==이때 우리가 속한 단계처럼 작성되거나, 우리가 세운 파이프라인의 시작이 선점 가능하기 때문에 채널이 닫힐 것이라는 점을 추론할 수 있다.== 값을 전송하다가 단계가 대기 상태가 되는 경우, **select 문을 사용한다면 선점 가능하다.**

따라서 전체 파이프라인은 done 채널을 닫음으로써 항상 선점 가능하다. 멋지지 않은가?

#### 유용한 생성기들


이전에 인기 있는 생성기에 대해 설명할 예정이라고 언급했다. 한 번 더 이야기하지면, ==파이프라인의 생성기는 이산 값의 집합을 채널의 값 스트림으로 변환하는 함수이다.==

``` go
package main

  



  

    repeat := func(done <-chan interface{},
        values ...interface{},
    ) <-chan interface{} {

        valueStream := make(chan interface{})

        go func() {
            defer close(valueStream)
            for {
                for _, v := range values {
                    select {
                    case <-done:
                        return
                    case valueStream <- v:
                    }
                }
            }
        }()

        return valueStream
}
```


이 함수는 사용자가 멈추라고 말할 때까지 사용자가 전달한 값을 무한 반복한다.

reqeat와 함께 사용할 때 도움이 되는 take라는 또 다른 범용 파이프라인 단계를 살펴보자.

``` go
    take := func(
        done <-chan interface{},
        valueStream <-chan interface{},
        num int,

    ) <-chan interface{} {

        takeStream := make(chan interface{})

        go func() {
            defer close(takeStream)

            for i := 0; i < num; i++ {
                select {
                case <-done:
                    return
                case takeStream <-<- valueStream:
                }
            }
        }()
        return takeStream
    }
```

이 파이프라인 단계는 입력 valueStream에서 첫 번째 항목을 취한 다음 종료한다. 이 둘을 조합하면 매우 강력해질 수 있다.

``` go
    done := make(chan interface{})
    defer close(done)

  

    for num := range take(done, repeat(done, 1), 10) {
        fmt.Printf("%d ", num)

    }
```

이 코드는 다음과 같이 출력된다.

``` go
PS C:\Users\kgm09\goproject\src> go run main.go
1 1 1 1 1 1 1 1 1 1 
```

매우 기본적인 이 예제에서, 1을 무한히 반복해서 생성하기 위해 repeat 생성기를 만들었고, 그 중에 처음 10개만을 가져왔다. repeat 생성기의 송신 블록이 take 단계에서 수신되므로 repeat 생성기를 매우 효율적이다. 1의 무한 스트림을 생성할 수 있지만, ==take 단계로 숫자 N를 전달하면 N+1개의 인스턴스만 생성한다.==

###### 전체 파일

``` go
package main

import "fmt"


func main() {

  

    repeat := func(done <-chan interface{},
        values ...interface{},
    ) <-chan interface{} {

        valueStream := make(chan interface{})

        go func() {
            defer close(valueStream)
            for {
                for _, v := range values {
                    select {
                    case <-done:
                        return
                    case valueStream <- v:
                    }
                }
            }
        }()
       return valueStream
    }

  

    take := func(
        done <-chan interface{},
        valueStream <-chan interface{},
        num int,
    ) <-chan interface{} {
        takeStream := make(chan interface{})
        go func() {
            defer close(takeStream)
            for i := 0; i < num; i++ {
                
                select {
                case <-done:
                    return
                case takeStream <- <-valueStream:
                }
            }
        }()
        return takeStream
    }

    done := make(chan interface{})
    defer close(done)

    for num := range take(done, repeat(done, 1), 10) {
        fmt.Printf("%d ", num)
    }

  

}
```




이것을 확장할 수도 있다. 이번에는 또 다른 반복 생성기이지만, 반복적으로 함수를 호출하는 생성기를 만들어보자. 바로 repeatFn이라는 생성기이다.

``` go
    repeatFn:=func (
        done <-chan interface{},
        fn func ()  interface{},
    ) <-chan interface{} {

        valueStream:=make(chan interface{})

        go func() {
            defer close(valueStream)
            for{
                select{
                case <-done:
                    return
                case valueStream <-fn():
                }
            }
        }()
        return valueStream
    }
```


이를 사용해 10개의 랜덤한 숫자를 만들어보자.

``` go

    done := make(chan interface{})
    defer close(done)

    rand := func() interface{} {
        return rand.Int()
    }

    for num := range take(done, repeatFn(done, rand), 10) {
        fmt.Println(num)
    }

```

이 코드의 실행 결과는 다음과 같다.

``` go
PS C:\Users\kgm09\goproject\src> go run main.go
7116806733379453580
4957167530285191153
8105702628185508691
7848924151151124412
2570936068002913861
1094568048971577240
1459689394203524675
6490013014581608528
5925618321766816136
8310962285428531463
```

멋지다. 필요한 만큼 랜덤한 정수가 생성되는 무한한 채널이다.

이 생성기와 단계 모두가 interface{} 채널을 수신하고 보내는 이유가 궁금할 수 있다. 특정 타입에 맞춰 이런 함수들을 직접 작성하거나 Go 생성자를 작성할 수 있었다.

빈 인터페이스는 Go에서 다소 금기시되지만 개인적으로 파이프라인의 단계에서는 파이프라인의 패턴의 표준 라이프러리를 사용하기 위해 interface{}의 채널을 처리해도 좋다고 생각한다. ==앞에서 설명한 것처럼, 파이프라인의 유용성은 재사용 가능한 단계에서 비롯된다.== ==이것은 각 단계가 그 자체에 알맞는 특수성 수준에서 작동할 때 가장 잘 이루어진다.== repeat 및 repeatFn 생성기는 목록 또는 연산자를 반복해 데이터 스트림을 생성하는 것이 주된 관심사다. take 단계는 파이프라인을 제한하는 것이 주된 관심사다. 이러한 연산들은 작업 중인 타입에 대한 정보가 필요하지 않으며, 대신 매개 변수에 대한 지식만 필요하다.

특정 타입을 처리해야 할 경우, 타입 단정문(assertion)를 수행하는 단계를 배치할 수 있다. 추가적인 파이프라인 단계(그리고 고루틴)와 타입 단정문을 더하는 것으로 인한 성능상의 부담은 무시할 만 하다. toString 파이프라인 단계를 추가하는 간단한 예제를 살펴보자.

``` go
toString := func(done <-chan interface{},
        valueStream <-chan interface{},
    ) <-chan string {

  

        stringStream := make(chan string)

        go func() {
            defer close(stringStream)
            for v := range valueStream {
                select {
                case <-done:
                    return
                case stringStream <- v.(string):
                }
            }
        }()
        return stringStream
    }
```

그리고 이를 사용하는 예제를 살펴보자.

``` go
    done := make(chan interface{})
    defer close(done)

  

    var message string
    for token := range toString(done, take(done, repeat(done, "I", "am."), 5)) {
        message += token
    }
    fmt.Printf("message: %s ...", message)
```

실행 결과는 다음과 같다.

``` shell
PS C:\Users\kgm09\goproject\src> go run main.go
message: Iam.Iam.I ...
```
파이프라인을 일반화하는 부분의 성능상 비용은 무시할 수 있음을 증명해보겠다. 두 가지 성능 측정 함수, 즉 일반적인 단계를 테스트하는 함수와 타입에 특화된 단계를 테스트하는 함수를 작성한다.


``` go
Running tool: C:\Users\kgm09\go\bin\go.exe test -benchmem -run=^$ -bench ^BenchmarkGenericc$ src/pipelines

  

goos: windows

goarch: amd64

pkg: src/pipelines

cpu: AMD Ryzen 5 4500U with Radeon Graphics        

BenchmarkGenericc-6       504788          2385 ns/op           0 B/op          0 allocs/op

PASS

ok      src/pipelines   2.586s



Running tool: C:\Users\kgm09\go\bin\go.exe test -benchmem -run=^$ -bench ^BenchmarkTyped$ src/pipelines

  

goos: windows

goarch: amd64

pkg: src/pipelines

cpu: AMD Ryzen 5 4500U with Radeon Graphics        

BenchmarkTyped-6     1000000          1434 ns/op           0 B/op          0 allocs/op

PASS

ok      src/pipelines   1.917s
```

특정 타입에 특화된 단계가 2배지만, 크게 의미 있는 정도는 아니다. 일반적으로 파이프라인의 성능상 제한 요소는 생성기 또는 계산 집약적인 단계 중 하나다. 만약 생성기가 repeat나 repeatFn 생성기처럼 메모리에서 스트림을 생성하지 않으면, 아마도 입출력이 성능에 큰 영향을 미칠 것이다. 디스크나 네트워크에서 읽어오는 경우 여기서 보여지는 성능상의 부하가 큰 의미가 없어질 것이다.

작업 단계 중 하나에 계산량을 많이 필요하다면, 이것 역시 이러한 성능상의 부하를 퇴색시킬 것이다. 이 기법을 쓰는 것이 여전히 마음에 걸린다면, 생성기 단계를 생성하기 위한 Go 생성자를 작성할 수 있다. 계산적으로 비용이 많이 드는 단계를 어떻게 줄일 수 있을까? 이것이 전체 파이프라인의 속도를 제한하지 않을까?

이를 완화하는 방법인 "팬 아웃, 팬 인" 기법에 대해 알아보겠다.


---
# 팬 아웃, 팬 인

^7b5465


이제 파이프라인을 설치했다. 데이터는 시스템을 통해 아름답게 흘러 들어가며, 함께 연결된 단계를 거치면서 변형된다. 이는 아름다운 흐름(stream) 같다. 아름답고 느린 흐름, 그런데 왜 그렇게 오래 걸리는 건가?

때로는 파이프라인의 단계에서 특별하게 많은 계산을 필요할 수 있다. ==이 경우, 많은 계산을 필요로 하는 단계가 완료되기를 기다리는 동안 파이프라인상의 앞쪽 단계들이 대기 상태가 될 수 있다.== ==또한 파이프라인 자체가 전체적으로 실행하는 데 오랜 시간이 걸릴 수도 있다. ==이 문제를 어떻게 해결할 수 있을까?

파이프라인에는 흥미로운 능력이 있다. ==바로 개별 단계를 조합해 데이터 스트림에서 연산할 수 있다는 점이다. 파이프라인의 단계들을 여러 번 재사용할 수 있다.== 여러 개의 고루틴을 통해 파이프라인의 상류 단계로부터 데이터를 가져오는 것을 **병렬화** 하면서, 파이프라인상의 한 단계를 재사용하면 흥미롭지 않겠는가? 이는 파이프라인의 성능을 향상시키는 데 도움을 줄 수 있다.



사실, 이 패턴은 **팬 아웃, 팬 인**이라는 이름이 있다.

**팬 아웃**은 파이프라인의 입력을 처리하기 위해 여러 개의 고루틴들을 시작하는 프로세스를 나타내는 용어이고, 팬 인은 여러 결과를 하나의 채널로 결합하는 프로세스를 설명하는 용어이다.

그렇다면이 패턴이 활용하기에 적합한 파이프라인의 단계는 무엇일까? 다음 두 가지 사항이 모두 적용되는 경우, 해당 단계에 팬 아웃을 수행하는 것을 생각해볼 수 있다.

- 단계가 이전에 계산한 값에 의존하지 않는다.
- 단계를 실행하는데 **시간이 오래 걸린다.**

순서 독립성(order-independence)은 중요하다. 왜냐하면 동시에 실행되는 해당 단계의 복사본이 어떤 순서로 실행되는지, 어떤 순서로 리턴할지에 대한 보장이 없기 때문이다.

예제를 살펴보자. 다음 예제에서는 소수를 찾는 매우 비효율적인 방법을 구현했다. "[[Go의 동시성 패턴#파이프라인|파이프라인]]"에서 작성한 많은 단계를 사용한다.

``` go
package main

  

import (
    "fmt"
    "math/rand"
    "runtime"
    "sync"
    "time"
)

  

func main() {

  

    repeatFn := func(
        done <-chan interface{},
        fn func() interface{},
    ) <-chan interface{} {

        valueStream := make(chan interface{})

  

        go func() {
            defer close(valueStream)
            for {
                select {
                case <-done:
                    return
                case valueStream <- fn():
                }
            }
        }()
        return valueStream
    }

  

    take := func(
        done <-chan interface{},
        valueStream <-chan interface{},
        num int,
    ) <-chan interface{} {

        takeStream := make(chan interface{})

        go func() {
            defer close(takeStream)
            for i := 0; i < num; i++ {
                select {
                case <-done:
                    return
                case takeStream <- <-valueStream:
                }
            }
        }()
        return takeStream
    }

  

    toInt := func(done <-chan interface{}, valueStream <-chan interface{}) <-chan int {

        intStream := make(chan int)

        go func() {
            defer close(intStream)
            for v := range valueStream {
                select {
                case <-done:
                    return
                case intStream <- v.(int):
                }
            }
        }()
        return intStream
    }

  

    primeFinder := func(done <-chan interface{}, intStream <-chan int) <-chan interface{} {

        primeStream := make(chan interface{})

        go func() {
            defer close(primeStream)

            for integer := range intStream {
                integer -= 1
                prime := true

                for divisor := integer - 1; divisor > 1; divisor-- {
                    if integer%divisor == 0 {
                        prime = false
                        break
                    }
                }

  

                if prime {
                    select {
                    case <-done:
                        return
                    case primeStream <- integer:
                    }
                }
            }
        }()
        return primeStream
    }

  

    fanIn := func(done <-chan interface{},
        channels ...<-chan interface{}) <-chan interface{} { // #1

        var wg sync.WaitGroup // #2
        multiplexedStream := make(chan interface{})

  

        multiplex := func(c <-chan interface{}) { // #3

            defer wg.Done()

            for i := range c {

                select {
                case <-done:
                    return
                case multiplexedStream <- i:
                }
            }
        }

        wg.Add(len(channels)) // #4
        for _, c := range channels {
            go multiplex(c)
        }

        go func() { // #5
            wg.Wait()
            close(multiplexedStream)
        }()
        return multiplexedStream
    }

  

    rand := func() interface{} {
        return rand.Intn(50000000)
    }

    done := make(chan interface{})
    defer close(done)
    
    start := time.Now()
    randIntStream := toInt(done, repeatFn(done, rand))

    numFinders := runtime.NumCPU()
    fmt.Printf("Spinning up %d prime finders.\n", numFinders)

    finders := make([]<-chan interface{}, numFinders)
    fmt.Println("Primes")

    for i := 0; i < numFinders; i++ {
        finders[i] = primeFinder(done, randIntStream)
    }

    for prime := range take(done, fanIn(done, finders...), 10) {
        fmt.Printf("\t%d\n", prime)
    }

    fmt.Printf("Search took: %v", time.Since(start))
}
```

이 코드의 실행 결과는 다음과 같다.


``` shell
PS C:\Users\kgm09\goproject\src> go run main.go
Spinning up 6 prime finders.
Primes
        91237
        43236397
        31862627
        47282077
        31931623
        43205887
        6911603
        39417887
        45872573
        10234727
Search took: 2.2005173s
```

스트림을 50,000,000보다 작은 난수의 스트림을 생성하고, 이를 정수 스트림으로 변환한 다음 primeFinder 단계로 전달한다. primeFinder는 단순히 입력 스트림에 제공된 숫자를 그보다 작은 숫자로 나누기 시작한다. 성공하지 못하면 그 값을 다음 단계로 전달한다. 이는 소수를 찾으려고 시도하는 끔찍한 방법이지만 오랜 시간이 걸려야 한다는 요구를 확실히 충족한다.

for 루프에서는 발견된 소수들을 range로 순회하면서 들어오는 대로 출력하며, take 단계에 의해 10개의 소수가 발견된 후에는 파이프라인을 닫는다. ==그런 다음 검색이 얼마나 오래 걸렸는지를 출력한다.== ==done 채널은 defer 구문으로 인해 닫히고 파이프라인은 해체된다.== 결과에서 중복을 피하기 위해, 파이프라인에 또 다른 단계를 도입해 이미 발견된 소수를 집합에 저장해둘 수 있지만, 간단한 구현을 위해 이 부분은 무시한다.

10개의 소수를 찾는 데 대략 23초가 걸렸다. 이는 좋지 않다. ==일반적으로 알고리즘 자체를 먼저 살펴보고, 알고리즘 참고서를 들고 각 단계에서 개선할 수 있는 부분이 있는지 확인한다.== 그러나 여기서 이 단계의 목적이 느린 연산이므로, 이 느린 연산을 보다 빠르게 소화할 수 있도록 단계 중 하나 이상을 팬 아웃할 방법이 있는지 살펴보겠다.

이 예제는 비교적 간단하다. 예제에는 난수를 생성하는 단계와 소수를 걸러내는 단계만 있다. 더 큰 프로그램에서 파이프라인이 더 많은 단계로 구성될 수 있다. 어떤 단계를 팬 아웃 해야 할지 어떻게 할 수 있을까? ==앞에서 언급한 순서 독립성과 지속 시간이라는 기준을 기억하라.== 난수 생성기를 순서에 독립적이지만, 실행하는 데 특별히 시간이 필요하지 않다. primeFinder 단계도 역시 순서에 상관없으며, 숫자는 소수이거나 소수가 아니다. 그리고 단순한 알고리즘으로 인해 실행하는데 오랜 시간이 걸린다. 이것은 팬 아웃하기 좋은 후보인 것 같다.

다행이도 파이프라인에서 단계를 펼치는 절차는 매우 쉽다. 이제 해야 할 일은 그 단계의 여러 버전을 시작하는 것 뿐이다. 따라서 다음과 같이 코드를 그 다음 코드로 바꿔주면 된다.

``` go
    primeStream:=primeFinder(done,randIntStream)
```

이 코드를 이렇게 수정한다.

```go

numFinders:=runtime.NumCPU()
finders:=make([]<-chan int,numFinders)
for i:=0;i<numFinders;i++{
finders[i]=primeFinder(done,randIntStream)
}
```

여기서는 이 단계의 복사본을 가지고 있는 CPU만큼 실행한다. 내 컴퓨터 중 하나에서는 runtime.NumCPU()이 8을 리턴하므로 이 숫자를 계속 사용하겠다. ==실제 운영 환경에서는 최적의 CPU 수를 결정하기 위해 약간의 경험적인 테스트를 수행할 것이다.== 그러나 여기에서는 예제를 간단하게 유지하고 CPU가 findPrimes 단계의 복사본 하나면 실행중이라고 가정하자.

이게 전부다! 이제 8개의 고루틴이 난수 생서기에서 데이터를 끌고와서, 그 수가 소수인지 판단하려고 시도한다. 난수를 생성하는데 시간이 많이 걸리지 않아야 findPrimes 단계의 각 고루틴이 해당 숫자가 소수인지 여부를 판단하고 바로 또 다른 가용 숫자를 사용할 수 있다.

그렇지만 문제가 있다. ==8개의 고루틴을 가지고 있고 8개의 채널을 가지고 있지만, range 에서 소수를 가져올 때는 하나의 채널만을 기대한다.== 이 때문에 이 패턴의 팬인 부분을 사용한다.


``` go
fanIn := func(done <-chan interface{},

        channels ...<-chan interface{}) <-chan interface{} { // #1
        var wg sync.WaitGroup // #2
        multiplexedStream := make(chan interface{})


        multiplex := func(c <-chan interface{}) { // #3
            defer wg.Done()

            for i := range c {
                select {
                case <-done:
                    return
                case multiplexedStream <- i:
                }
            }
        }

        // 모든 채널들로부터 select

        wg.Add(len(channels)) // #4
        for _, c := range channels {
            go multiplex(c)
        }

        go func() { // #5
            wg.Wait()
            close(multiplexedStream)
        }()
        return multiplexedStream
    }
```


1. 여기서는 고루틴들을 종료시키기 위해 표준 done 채널을 사용하며, interface{} 채널들의 가변 슬라이스를 사용한다.
2. 이 줄에서는 sync.WaitGroup을 생성하여 모든 채널의 데이터가 소진될 때까지 기다릴 수 있다.
3. 여기서는 채널을 전달하면 채널에서 읽은 값을 multiplexedStream 채널로 전달하는 multiplex 함수를 만든다.
4. 이 줄은 다중화하는 채널 수에 따라 sync.WaitGroup을 증가시킨다.
5. 여기서는 다중화하고 있는 모든 채널이 비워질 때까지 기다리는 고루틴을 만들어 multiplexedStream 채널을 닫을 수 있도록 해 준다.

간단히 말해서, 팬 인은 여러 소바지가 읽어 들이는 하나의 다중화된 채널을 생성하는 것, ==그리고 하나의 입력 채널마다 하나의 고루틴을 동작시키는 것, 입력 채널이 모두 닫혔을 때 다중화된 채널을을 닫는 하나의 고루틴과 관련돼어 있다.== N개의 다른 고루틴들이 완료되기를 기다리고 있는 고루틴을 만들 예정이므로, 조정을 위해 sync.WaitGrou을 만드는 것이 합리적이다. multiplex 함수는 WaitGroup에 작업이 완료됐음을 알린다.




>[!추가 알림]
>팬 인, 팬 아웃 알고리즘의 단순한 구현은 ==결과가 도착하는 순서가 중요하지 않은 경우에만 제대로 작동한다.== randIntStream에서 읽은 항목의 순서가 소수임을 판단하는 체를 통과할 때 보존된다는 것을 보장하는 어떠한 조치도 취하지 않았다. 나중에 이 순서를 유지하는 방법의 예를 살펴보겠다.

이 모든 것들을 하나호 합친 후, 실행 시간이 얼마나 감소하는지 살펴보자.

>[!important] 
> 코드 전체를 출력하도록 하겠다.

``` go
package main

  

import (

    "fmt"
    "math/rand"
    "runtime"
    "sync"
    "time"

)

  

func main() {

    repeatFn := func(
       done <-chan interface{},
        fn func() interface{},
    ) <-chan interface{} {

        valueStream := make(chan interface{})
        go func() {
            defer close(valueStream)
            for {
                select {
                case <-done:
                    return
                case valueStream <- fn():
                }
            }
        }()
        return valueStream
    }

  

    take := func(
        done <-chan interface{},
        valueStream <-chan interface{},
        num int,
    ) <-chan interface{} {

        takeStream := make(chan interface{})

        go func() {
            defer close(takeStream)

            for i := 0; i < num; i++ {
                select {
                case <-done:
                    return
                case takeStream <- <-valueStream:
                }
            }
        }()
        return takeStream
    }

  

    toInt := func(done <-chan interface{}, valueStream <-chan interface{}) <-chan int {

        intStream := make(chan int)

        go func() {
            defer close(intStream)
            for v := range valueStream {
                select {
                case <-done:
                    return
                case intStream <- v.(int):
                }
            }
        }()
        return intStream
    }

  

    primeFinder := func(done <-chan interface{}, intStream <-chan int) <-chan interface{} {

        primeStream := make(chan interface{})

        go func() {
            defer close(primeStream)

            for integer := range intStream {
                integer -= 1
                prime := true

  

                for divisor := integer - 1; divisor > 1; divisor-- {
                    if integer%divisor == 0 {
                        prime = false
                        break
                    }
                }

  

                if prime {
                    select {
                    case <-done:
                        return
                    case primeStream <- integer:
                   }
                }
            }
        }()
        return primeStream
    }

  

    fanIn := func(done <-chan interface{},

        channels ...<-chan interface{}) <-chan interface{} { // #1
        var wg sync.WaitGroup // #2
        multiplexedStream := make(chan interface{})

  

        multiplex := func(c <-chan interface{}) { // #3
         defer wg.Done()

            for i := range c {
                select {
                case <-done:
                    return
                case multiplexedStream <- i:
                }
            }
        }

        // 모든 채널들로부터 select

        wg.Add(len(channels)) // #4
        for _, c := range channels {
            go multiplex(c)
        }

        go func() { // #5
            wg.Wait()
            close(multiplexedStream)
        }()
        return multiplexedStream
    }

  

    done := make(chan interface{})
    defer close(done)

  

    start := time.Now()

    rand := func() interface{} {
        return rand.Intn(50000000)
    }

    randIntStream := toInt(done, repeatFn(done, rand))

    numFinders := runtime.NumCPU()
    fmt.Printf("Spinning up %d prime finders.\n", numFinders)
    finders := make([]<-chan interface{}, numFinders)

    fmt.Println("Primes:")

    for i := 0; i < numFinders; i++ {
        finders[i] = primeFinder(done, randIntStream)
    }

  

    for prime := range take(done, fanIn(done, finders...), 10) {
        fmt.Printf("\t%d\n", prime)
    }

    fmt.Printf("Search took: %v", time.Since(start))

}
```


결과는 다음과 같다.

``` go
PS C:\Users\kgm09\goproject\src> go run main.go
Spinning up 6 prime finders.
Primes:
        36288451
        46141883
        2236139
        49375763
        39018953
        7897321
        49730227
        23742259
        28929809
        25260341
Search took: 3.7445275s
```

대략 4초 가까이 줄어들었으니 나쁘지 않다! 이것은 팬 아웃, 팬인 패턴의 이점을 분명히 보여주며, 파이프라인의 유용성을 반복해서 보여준다. 우리는 프로그램의 구조를 크게 바꾸지 않고 실행시간을 크게 단축했다.

---
# or-done 채널

^5a946e

때로는 시스템에서 서로 다른 부분의 채널들로 작업하게 되는 경우가 있다. ==파이프라인과 달리 작업 중인 코드가 done 채널을 통해 취소될 대 채널이 어떻게 동작할지 단언할 수 없다.== 즉, ==고루틴이 취소됐다는 것이 읽어오는 채널 역시 취소됐음을 의미하는지는 알 수 없다.==

이러한 이유 때문에 "[[Go의 동시성 패턴#^5a8639|고루틴 누수 방지]]"에서 설명했듯이, 채널에서 읽어오는 부분은 done 채널에서 select하는 selec구문으로 감싸야 한다. 이 방식은 완벽하지만, 다음과 같이 채널에서 쉽게 읽어올 수 있는 코드가 필요하다.

``` go
for val:=range myChan{
// val로 무언가 작업을 한다.
}
```

그리고 이 코드는 다음과 같이 확장된다.

```go
loop:
for{
select{
case <-done:
break loop
case maybeVal, ok:=<-myChan:
if ok==false{
return // 혹은 break로 for 문을 벗어난다.
}
// val로 무언가 작업을 한다.
}
}
```

 이 코드는 순식간에 바빠질 수 있다. 특히나 중첩된 루프물을 사용하는 경우에는 더욱 그러하다. 보 명확한 동시성 코드를 작성하기 위해 고루틴을 사용하고, 성급한 최적화를 수행하지 않는다는 주제를 계속하면서, 하나의 고루틴으로 이 문제를 해결할 수 있다.

``` go
orDone:=func(done, c<-chan interface{})<-chan interface{}{
valStream:=make(chan interface{})
go func(){
defer close(valStream)
for{
select{
case <-done:
	return
case v,ok:=<-c:
	if ok==false{
	return
	}
	select{
	case valStream <-v:
	case <-done:
	}
}
}
}()
return valStream
}
```
이렇게 하면 다시 다음과 같은 단순한 반복문으로 돌아갈 수 있다.

```go
for val:=range orDone(done,myChan){
// var로 무언가 작업한다.
}
```


운영상의 몇몇 특정한 경우에서는 일련의 select 구문을 활용한 딱 맞는 반복문이 필요하다고 생각할 수 있다.

---
# tee 채널


때로는 채널에서 들어오는 값을 분리해 코드베이스의 별개의 두 영역으로 보내고자 할 수도 있다. 사용자 명령 채널을 생각해보자. 채널에서 사용자 명령 스트림을 가져와서 이를 실행해줄 누군가에게 이 명령을 보내고, 또 나중에 감사를 위해 명령을 기록할 누군가에게 보낼 수 있다.

유닉스 계열 시스템의 tee 명령에서 그 이름을 따온 **tee 채널**이 바로 이 역할을 한다. 읽어들일 채널을 전달할 수 있으며, 동일한 값을 얻어오는 별개의 채널 두 개를 리턴한다.

```go

tee:=func(
done <-chan interface{},
in <-chan interface{},

)(_,_ <-chan interface{}){
out1:=make(chan interface{})
out2:=make(chan interface{})

go func(){

defer close(out1)
defer close(out2)

for val:=range orDone(done,in){
 var out1, out=out1,out2 // #1
 for i:=0;i<2;i++{ // #2
 select{
 case <-done:
 case out1<-val:
out1=nil         // #3
case out2<-val:
out2=nil         // #3
 }
 }
}

}()
return out1,out2
}
```

1. 로컬 버전의 out1과 out2를 사용하고자 하므로, 지역 변수를 선언해 이들을 가릴 것이다.
2. 하나의 select문을 사용해 out1과 out2에 대한 쓰기가 서로를 차단하지 않도록 할 것이다. 두 채널 모두에 쓸 수 있도록 하기 위해 selct문의 루프를 두 번, 아웃바운드 채널마다 한 번씩 반복한다.
3. 한 채널에 쓴 이후에는 로컬 복사본을 nil을 설정해, ==그 채널에 대한 추가적인 기록이 차단되도록 하고, 다른 채널에서는 계속 쓸 수 있도록 한다.==

out1과 out2에 대한 쓰기는 강하게 결합돼 있다. out1과 out2가 모두 쓰여진 다음에 in 채널에서 다음 항목을 가져온다. 어쨌거나 일반적으로 각 채널에서 읽어가는 프로세스의 처리 속도 tee 커맨드의 관심사가 아니기 때문에 문제가 되지는 않지만, 주목할 만한 가치가 있다.

다음은 간단한 예시다.

``` go
done:=make(chan interface{})
defer close(done)

out1,out2:=tee(done,take(done,repeat(done,1,2),4))

for val1:=range out1{
fmt.Println("out1: %v, out2: %v\n",val1,<-out2)
}

```


이 패턴을 활용하면 채널을 시스템의 합류 지점으로 계속 사용할 수 있다.

---
# bridge 채널

연속된 채널로부터 값을 사용하고 싶을 때도 있다.

``` go
<-chan <-chan interface{}
```

139페이지의 "[[Go의 동시성 패턴#^6dfa66|or 채널]]"이나 165페이지의 "[[Go의 동시성 패턴#^7b5465|팬 아웃, 팬 인]]"에서 배웠듯이, 채널들의 슬라이스를 하나의 채널로 병합하는 것과는 약간 다르다. ==연속된 채널은 서로 다른 출처에서부터 순서대로 쓴다는 점을 시사한다.== 간헐적으로 발생하는 파이프라인 단계가 하나의 예일 수 있다.

128페이지의 "[[Go의 동시성 패턴#^3ef919|제한]]"에서 만든 패턴을 따르고, 채널에 대한 소유권을 해당 채널에 쓰는 고루틴이 가진다면, 새로운 고루틴에서 파이프라인 단계가 재시작될 때마다 새로운 채널이 생성될 것이다. 이 점은 실질적으로 연속된 채널을 가진다는 것을 의미한다. 이 시나리오는 263페이지의 "비정상 고루틴의 치료"에서 더 자세히 살펴본다.

소비자라면 코드가 그 값이 연속된 채널로부터 온다는 사실에 신경을 쓰지 않을 수도 있다. 이 경우 채널들의 채널을 처리하는 것이 어려울 수 있다. ==채널들의 채널을 채널 브리징(bridging)이라고 하는 기법을 통해 단순한 채널로 변형시킬 수 있는 함수를 정의한다면, 소비자는 훨씬 더 쉽게 당면한 문제에 집중할 수 있다.== 이를 위해 다음과 같은 코드를 사용할 수 있다.

``` go
    bridge := func(done <-chan interface{},
        chanStream <-chan <-chan interface{}) <-chan interface{} {

  

        valStream := make(chan interface{})

        go func() {
            defer close(valStream)
            for {
                var stream <-chan interface{}
                select {
                case maybeStream, ok := <-chanStream:
                    if ok == false {
                        return
                    }

                    stream = maybeStream

                case <-done:
                    return
                }

                for val := range orDone(done, stream) {

                    select {
                    case valStream <- val:
                    case <-done:
                    }
                }
            }
        }()
        return valStream
    }
```

1. 이것은 bridge로부터의 모든 값을 리턴하는 채널이다.
2. 이 루프는 chanStream에서 채널들을 가지고 오며, 채널을 사용할 수 있도록 내부의 루프물을 제공한다.
3. 이 루프는 주어진 채널의 값을 읽고, 해당 값을 valStream에 반복한다. ==현재 반복하고 있는 스트림이 닫히면 이 채널에서 읽기를 수행하는 루프에서 빠져나가고, 루프의 다음 반복이 이어지며, 다음으로 읽을 채널을 선택한다.== 이를 통해 끊임없는 값을 스트림을 얻을 수 있다.

앞의 코드는 매우 직관적이다. 이제는 채널들의 채널을 하나의 채널 같은 외관(facade)으로 나타내기 위해 bridge를 사용할 수 있다. 다음은 하나의 요소가 쓰여진 연속된 10개의 채널을 생성하고 이 채널들을 bridge 함수로 전달하는 예제이다.

``` go
genVals := func() <-chan <-chan interface{} {

        chanStream := make(chan (<-chan interface{}))

        go func() {
           defer close(chanStream)
            for i := 0; i < 10; i++ {
                stream := make(chan interface{}, 1)
                stream <- i
                close(stream)
                chanStream <- stream
            }
        }()
        return chanStream
    }

    for v := range bridge(nil, genVals()) {
        fmt.Printf("%v ", v)
    }
```

이 코드의 실행 결과는 다음과 같다.

``` go
PS C:\Users\kgm09\goproject\src> go run main.go
0 1 2 3 4 5 6 7 8 9 
```
bridge 덕분에 하나의 range 구문에서 채널들의 채널을 사용할 수 있게 됐으며, 반복문의 로직에 집중할 수 있게 됐다. 채널들의 채널을 파괴하는 것은 이 문제와 관련된 코드에서 처리해야 한다.



###### 전체 코드
``` go
package main

import "fmt"

func main() {

    orDone := func(done, c <-chan interface{}) <-chan interface{} {

        valStream := make(chan interface{})
        go func() {
            defer close(valStream)
            for {
                select {
                case <-done:
                    return
                case v, ok := <-c:
                    if ok == false {
                        return
                    }

                    select {
                    case valStream <- v:
                    case <-done:
                    }
                }
            }
        }()
        return valStream
    }

  

    bridge := func(done <-chan interface{},
        chanStream <-chan <-chan interface{}) <-chan interface{} {

        valStream := make(chan interface{})
        go func() {
            defer close(valStream)
            for {
                var stream <-chan interface{}
                select {
                case maybeStream, ok := <-chanStream:
                    if ok == false {
                        return
                    }
                   stream = maybeStream
                case <-done:
                    return
                }

                for val := range orDone(done, stream) {

                    select {
                    case valStream <- val:
                    case <-done:
                    }
                }
            }
        }()
        return valStream
    }

  

    genVals := func() <-chan <-chan interface{} {

        chanStream := make(chan (<-chan interface{}))

        go func() {
            defer close(chanStream)

            for i := 0; i < 10; i++ {
                stream := make(chan interface{}, 1)
                stream <- i
                close(stream)
                chanStream <- stream
            }
        }()
        return chanStream
    }
    for v := range bridge(nil, genVals()) {
        fmt.Printf("%v ", v)
    }
}
```








---

# 대기열 사용

파이프라인이 아직 준비되지 않았더라도, 파이프라인에 대한 작업을 받아들이는 것이 유용할 때가 있다. 이 절차를 **대기열 사용**(queuing)이라고 한다.

대기열 사용이란, ==어떤 단계가 일부 작업을 완료하면 이를 메모리의 임시 위치에 저장해 다른 단계에서 나중에 조회할 수 있으며==, ==작업을 완료한 단계는 작업 결과에 대한 참조를 저장할 필요가 없다는 의미이다.== 101페이지의 "[[Go의 동시성 구성 요소#채널|채널]]"에서 대기열의 일종인 **버퍼링된 채널**에 대해 이야기했지만, 그 이후로 그다지 많이 사용하지는 않았다. 여기에는 정당한 이유가 있다.

시스템에 대기열을 도입하는 것은 매우 유용하지만, 일반적으로 프로그램을 최적화할 대 사용하려는 마지막 기술 중 하나다. 대기열을 성급하게 도입하면 데드락이나 라이브락과 같은 동기화 문제가 드러나지 않을 수 있으며, 프로그램이 정확해짐에 따라 대기열이 얼마나 필요한지 알게 될 수도 있다.

그래서 대기열을 사용하면 어떤 점이 좋을까? 시스템 성능을 조정하려고 할 때 사람들이 범하는 일반적인 실수 중 하나로, 성능 문제를 해결하려고 대기열 도입을 고민하는 것에 대해 이야기하면서 이 질문에 대한 답을 해보자. ==**대기열 사용**은 프로그램의 총 실행 속도를 높여주지 않는다. 단지 다른 프로그램이 다른 방식으로 동작하도록 허용할 뿐이다.==


그 이유를 이해하기 위해 간단한 파이프라인을 살펴보자.

```go

done:=make(chan interface{})
defer close(done)

zeros:=take(done,3,repeat(done,0))
short:=sleep(done,1*time.Second,zeros)
long:=sleep(done,4*time.Second,short)
pipeline:=long
```

이 파이프라인은 4개의 단계를 하나로 연결한다.

1. 끊임없이 0의 스트림을 생성하는 반복단계 
2. 3개의 아이템을 받으면 그 이전 단계를 취소하는 단계
3. 1초간 슬립하는 "짧은(short)" 단계
4. 4초간 슬립하는 "긴(long)" 단계

이 예제의 목적상 1단계와 2단계는 즉각적인 것으로 가정하고, 슬립 단계들이 파이프라인의 실행 시간에 어떻게 영향을 미치는지 초점을 맞춰보자.

다음은 시간 t, 반복 횟수 i, long 및 short 단계가 다음 값으로 넘어가는 데 걸린 시간을 측정한 표이다.




| 시간(t) | i   | long 단계 | Short 단계 |
| ----- | --- | ------- | -------- |
| 0     | 0   |         | 1초       |
| 1     | 0   | 4초      | 1초       |
| 2     | 0   | 3초      | (대기)     |
| 3     | 0   | 2초      | (대기)     |
| 4     | 0   | 1초      | (대기)     |
| 5     | 1   | 4초      | 1초       |
| 6     | 1   | 3초      | (대기)     |
| 7     | 1   | 2초      | (대기)     |
| 8     | 1   | 1초      | (대기)     |
| 9     | 2   | 4초      | (닫힘)     |
| 10    | 2   | 3초      |          |
| 11    | 2   | 2초      |          |
| 12    | 2   | 1초      |          |
| 13    | 3   | (닫힘)    |          |

이 파이프라인을 실행하는 데 대략 13초 정도 걸린다는 것을 알 수 있다. short 단계는 완료하는데 9초 정도 걸린다. 버퍼를 포함하도록 파이프라인을 수정하면 어떻게 될까? 동일한 파이프라인에서 long 단계와 short 단계 사이에 크기가 2인 버퍼를 도입한 후 측정해보자.

```go
done:=make
defer close(done)

zeros:=take(done,3,repeat(done,0))
short:=sleep(done,1*time.Second,zeros)
buffer:=buffer(done,2,short)
long:=sleep(done,4*time.Second,buffer)
pipeline:=long
```

실행 시간은 다음과 같다.


| Time(t) | i   | Long 단계 | 버퍼  | Short 단계 |
| ------- | --- | ------- | --- | -------- |
| 0       | 0   |         | 0/2 | 1초       |
| 1       | 0   | 4초      | 0/2 | (닫)      |
| 2       | 0   | 3초      | 1/2 |          |
| 3       | 0   | 2초      | 2/2 |          |
| 4       | 0   | 1초      | 2/2 |          |
| 5       | 1   | 4초      | 1/2 |          |
| 6       | 1   | 3초      | 1/2 |          |
| 7       | 1   | 2초      | 1/2 |          |
| 8       | 1   | 1초      | 1/2 |          |
| 9       | 2   | 4초      | 0/2 |          |
| 10      | 2   | 3초      | 0/2 |          |
| 11      | 2   | 2초      | 0/2 |          |
| 12      | 2   | 1초      | 0/2 |          |
| 13      | 3   | (닫힘)    |     |          |

전체 파이프라인은 여전히 13초가 걸린다! 하지만 short 단계의 실행 시간을 확인해보자. 이전에는 9초가 걸렸지만 이번에는 3초만에 완료됐다. 이 단계의 실행 시간을 2/3로 줄였다! 그러나 전체 파이프라인에 여전히 13초의 실행 시간이 필요하다면, 이게 어떤 식으로 도움이 될까?

다음과 같은 파이프라인을 생각해보자.

``` go
p:=processRequest(done,acceptConnection(done,httpHandler))
```

여기서 ==파이프라인은 취소될 때까지 종료되지 않으며, 연결을 수락하는 단계는 파이프라인이 취소될 때까지 계속해서 연결을 수락한다.== 이 시나리오에서 processRequest 단계가 acceptConnection 단계를 차단하는 것으로 인해, 프로그램에 대한 연결이 시간 초과되는 것을 원하지는 않을 것이다. 또한 acceptConnection 단계가 차단되는 상황을 가능한 한 피하고 싶을 것이다. 그렇지 않으면 프로그램 사용자는 자신의 요청이 모두 거부된 것을 보게 된다.

그러니까 대기열 도입의 유용성에 대한 질문의 답은, ==단계 중 하나의 실행 시간이 줄어드는 것이 아니라 그 단계가 차단 상태에 있는 시간이 줄어든다는 것이다.== 이렇게 하면 해당 단계에서 작업을 계속할 수 있다. 이 예에서 사용자의 요청이 지연될 가능성은 있지만, 서비스를 완전히 거부하지는 않는다.

대기열은 이러한 식으로 단계를 분리해 한 단계의 실행 시간이 다른 단계의 실행 시간에 영향을 미치지 않도록 한다는 점에 유용하다. 이 방식으로 단계를 분리하면 체크 시스템의 런타임 동작이 단계적으로 변경되는데, 이는 시스템에 따라 좋거나 나쁠 수 있다.

그 다음으로는 대기열을 조정하는 문제가 발생한다. 대기열을 어디에 배치해야 할까? 버퍼 크기는 어느 정도로 해야 할까? 이러한 질문에 대한 대답은 파이프라인의 특성에 따라 다르다.

대기열의 사용이 시스템의 전반적인 성능을 향상시킬 수 있는 상황을 분석해보자. 가능한 상황은 다음과 같다.

- 특정 단계에서 일==괄 처리 요청이 시간을 절약==하는 경우
- 특정 단계의 지연으로 인해 시스템에 피드백 루프가 생성되는 경우.

첫 번째 상황의 한 예로 입력을 전솧하기로 설계되 있는 것(예: 디스크)보다 상대적으로 빠른 곳(예: 메모리)으로 버퍼링 하는 단계가 있다. 물론 이 자체가 Go의 bufio 패키지의 목적이다. 다음은 대기열에 버퍼링된 쓰기와 버퍼링되지 않은 쓰기를 간단히 비교하는 예제이다.


``` go

package file

  

import (

    "bufio"
    "io"
    "log"
    "os"
    "testing"

)

  

func BenchmarkUnbufferedWrite(b *testing.B) {
    performedWrite(b, tmpFileOrFatal())
}

  

func BenchmarkBufferedWrite(b *testing.B) {
    bufferredFile := bufio.NewWriter(tmpFileOrFatal())
    performedWrite(b, bufio.NewWriter(bufferredFile))
}

  

func tmpFileOrFatal() *os.File {
    file, err := os.CreateTemp("", "tmp")
    if err != nil {
        log.Fatalf("error: %v", err)
    }
    return file
}

  

func performedWrite(b *testing.B, writer io.Writer) {

    repeat := func(
        done <-chan interface{},
        values ...interface{},
    ) <-chan interface{} {

        valuesStream := make(chan interface{})

        go func() {
            defer close(valuesStream)

            for {
                for _, v := range values {
                    select {
                    case <-done:
                        return
                    case valuesStream <- v:
                    }
                }
            }
        }()
        return valuesStream
    }

    take := func(done <-chan interface{},
        valueStream <-chan interface{},
        num int) <-chan interface{} {

        takeStream := make(chan interface{})

        go func() {
            defer close(takeStream)

            for i := 0; i < num; i++ {
                select {
                case <-done:
                    return
                case takeStream <- <-valueStream:
                }
            }
        }()
        return takeStream
    }

    done := make(chan interface{})
    defer close(done)


    b.ResetTimer()

    for bt := range take(done, repeat(done, byte(0)), b.N) {
        writer.Write([]byte{bt.(byte)})
    }
}

```

그리고 다음은 이 성능 측정의 결과이다.

``` shell
Running tool: C:\Users\kgm09\go\bin\go.exe test -benchmem -run=^$ -bench ^(BenchmarkUnbufferedWrite|BenchmarkBufferedWrite)$ src/pipelines

  

goos: windows

goarch: amd64

pkg: src/pipelines

cpu: AMD Ryzen 5 4500U with Radeon Graphics        

BenchmarkUnbufferedWrite-6        106975         10382 ns/op           1 B/op          1 allocs/op

BenchmarkBufferedWrite-6          474832          2401 ns/op           1 B/op          1 allocs/op

PASS

ok      src/pipelines   4.155s
```


버퍼링된 쓰기가 버퍼링되지 않는 쓰기보다 빠르다. 그 이유는 bufio.Writer에서는 충분한 데이터 덩어리(청크,chunk)가 버퍼에 누적될 때까지 내부적으로 대기열에 대기한 다음, 그 덩어리를 쓰기 때문이다. 이 과정을 종종 **청킹**이라고 부르는 데는 분명한 이유가 있다.

bytes.Buffer는 자신이 저장해야 하는 바이트를 수용하기 위해 할당된 메모리를 증가시켜야 하기 때문에 청킹을 사용하는 것이 빠르다. 여러 가지 이유로 할당된 메모리를 늘리는 데는 비용이 많이 든다. 따라서 대기열ㅇ르 사용하면 시스템 전체의 성능이 향상된다. 

이는 메모리 내에서 이루어지는 청킹의 간단한 예시일 뿐이지만, 현장에서는 청킹을 자주 경험할 수 있다. ==일반적으로 하나의 연산을 수행하는 데는 오버헤드가 요구되므로 청킹을 통해 시스템 성능을 향상시킬 수 있다. 데이터베이스 트랜잭션을 열고, 메시지 체크섬을 계산하고, 연속적인 공간을 할당하는 것이 그 예이다.==

청킹뿐만 아니라 후방 참조(lookbehind)를 지원하거나, 순서를 정렬함으로써 알고리즘을 최적화하는 경우에도 대기열의 사용은 도움이 될 수 있다.

두 번째 시나리오에서는 ==한 단계에서는 지연이 파이프라인 전체에 더 많은 입력을 유발시킨다.== 이것은 좀 더 알아차리기 어렵지만, 파이프라인의 상류upstream 단계로부터의 데이터 공급 시스템이 체계적으로 붕괴될 수도 있기 때문에 더 중요하다.

이 아이디어는 흔히 **부정적인 피드백 루프**, 하향 나선형, 심지어 죽음의 나선형이라고도 한다. ==그 이유는 파이프라인과 그 상류 단계의 시스템 사이에 순환 관계가 존재하기 때문이다.== 상류 단계 또는 시스템이 새로운 요청을 제출하는 비율은 파이프라인이 얼마나 효율적인지와 관련돼 있다.

파이프라인의 효율성이 특정 임계 값 아래로 떨어지면 파이프라인 상류 단계의 시스템이 파이프라인으로의 입력을 늘리기 시작하고, 이에 따라 파이프라인의 효율이 저하되면서 죽음의 나선이 시작된다. ==실패에 대한 일종의 **안전장치**(fail-safe)가 없으면 파이프라인을 이용하는 시스템이 결코 회복되지 않는다.==

파이프라인의 입구에 대기열을 도입하면, 요청에 대한 지연 시간을 비용으로 피드백 루프를 멈출 수 있다. 파이프라인에 대한 호출자의 관점에서 요청은 처리 중인 것처럼 보이지만 시간이 매우 오래 걸린다. 호출자가 시간을 초과하지 않는 이상 파이프라인은 안정적으로 유지된다. ==호출자가 시간을 초과하면, 대기열에서 꺼내올 때 준비가 됐는지 확인할 수 있도록 지원하는지 확실하게 할 필요가 있다.== 그렇게 하지 않으면 무심코 죽은 요청을 처리하다가 피드백 루프를 생성해 파이프라인의 효율성을 떨어트릴 수 있다.


>[! 죽음의 나선을 본 적이 있는가?]
>새롭게 온라인에 등장한 인기 있는 시스템(새로운 게임 서버, 제품 출시용 웹 사이트 등)에 처음 접속하려고 했을 때, 개발자가 최선을 다했음에도 불구하고 사이트가 계속 오락가락 하면, 축하한다! 당신은 부정적인 피드백 루프 목격자다.
>누군가 개발팀에게 대기열이 필요하다는 사실을 알아차릴 때까지, 개발팀은 언제나 다른 작업을 시도하다가 결국은 급하게 대기열을 구현한다.
>그러면 고객들이 대기열의 대기 시간에 대해 불평하기 시작한다!


이 책의 예제들에서 패턴이 드러난다는 점을 눈치챘을 수도 있다. 대기열은 다음 중 하나에서 구현돼야 한다.

- 파이프라인의 입구
- 일괄 작업으로 효율이 높아지는 단계

계산적으로 비싼 단계 이후나, 또 다른 곳에 대기열을 추가하고 싶은 유혹을 느낄 수도 있지만, 유혹을 이겨내도록 하자. 앞서 배웠듯이 대기열의 사용은 겨우 몇몇 상황에서만 파이프라인의 실행 시간을 단축시킨다. 이를 피하기 위한 시도로 여기저기에 대기열을 추가하는 것은 비참한 결과를 초래할 수 있다.


이것은 처음에는 직관적이지 않다. 그 이유를 이해하려면 파이프라인의 처리량을 논의해야 한다. 하지만 걱정할 필요는 없다. 그렇게 어려운 문제는 아니며, 대기열의 크기를 결정하는 방법에 대한 질문에 대답하는 데도 도움이 될 것이다.

==대기열 사용 이론에서는 충분한 샘플링을 통해 파이프라인의 처리량을 예측하는 법칙이 잇다. 이를 **리틀의 법칙**(Little's Law)이라고 부른다==. 리틀의 법칙을 이해하고 사용하려면 몇 가지 사항만 알면 된다.

먼저 리틀의 법칙을 대수적으로 정의해보자. 이 법칙은 일반적으로 $L=\lambda W$ 같이 표현된다.


- L= 시스템에서 구성 단위의 평균적인 수
- $\lambda$= 구성 단위의 평균 도착률
- W=구성 단위가 시스템에서 보내는 평균 시간


이 방정식은 소위 안정적인 시스템에만 적용된다. ==파이프라인에서 안정적인 시스템이란, 작업이 파이프라인에 들어오는, 즉 진입 속도가 시스템에서 나가는, 즉 탈출 속도와 같다.== 진입 속도가 탈출 속도를 초과하면 시스템이 불안정해지고 죽음의 나선 상태로 들어선다. 진입 속도가 탈출 속도보다 작은 경우에도 여전히 불안정한 시스템이지만, 자원을 완정하게 활용하지 못하는 상황이 발생할 뿐이다. 가장 최악의 상황은 아니지만, 클러스터나 데이터 센터와 같은 거대한 규모에서 과소 사용이 발견되면 이 문제에 관심을 가질 수 있다.


그러면 우리 파이프라인을 안정적이라고 가정하자. 구성 단위가 시스템에서 n의 배수인 W 만큼의 시간을 보낼 때 w를 줄이고 싶다면, 주어진 선택 사항은 시스템에서 구성 단위의 평균 수(L) 줄이는 방법 $(L/n =\lambda *W/n)$ 뿐이다. 그리고 탈출 속도를 높이면 시스템상 구성 단위의 평균 수를 줄일 수 있다.

또한 각 단계에 대기열을 추가하면 L이 증가하는데, 이는 곧 구성 단위의 도착률 증가 $(nL=n\lambda * W)$또는 구성 단위가 시스템에서 소비하는 평균의 증가 $(nL=\lambda * nW)$로 이어진다. 리틀의 법칙을 통해 대기열의 사용이 시스템에서 소비되는 시간을 줄이는 데 도움이 되지 않음을 입증했다. 또한 파이프라인을 하나의 큰 덩어리로 관찰하고 있기 때문에, W를 n배 만큼 줄이는 것이 파이프라인상의 모든 단계에 걸쳐 분산된다. 이 경우 리틀의 법칙은 실제로 다음과 같이 정의돼야 한다.

$L=\lambda \sum_i W_i$

즉, 전체 파이프라인의 속도는 가장 느린 단계에 의해 결정된다. 이것저것 가리지 말고 최적화하라!

리틀의 법칙은 깔끔하다! 이 간단한 방정식을 통해 파이프라인을 분석하는 방법을 모두 얻을 수 있다. 흥미로운 예시를 위해 이 방정식을 사용해보자. 분석을 진행하는 동안 파이프라인에는 3단계가 있다고 가정한다.

파이프라인이 초당 처리할 수 있는 요청의 수를 결정해보자. 파이프라인에서 표본을 추출(sampling)할 수 있으며, 하나의 요청(r)이 파이프라인을 통과하는 데는 약 1초(s)가 걸린다고 가정한다. 이 숫자들을 서로 연결해보자!

$3r=\lambda r/s * 1s$
$3r/s=\lambda r/s$
$\lambda r/s=3r/s$


이 파이프라인의 각 단계가 요청을 처리하기 때문에 L을 3으로 설정한다. 그런 다음 W를 1초로 설정하고, 약간의 대수학을 수행하면 보일 것이다. 이 파이프라인은 초당 3개의 요청을 처리할 수 있다.

원하는 수만큼의 요청을 처리하기 위해 대기열이 얼마나 커야 할지 정하는 것은 어떨까? 리틀의 법칙이 이 질문에 답하는 데도 도움이 된다.

표본 추출 결과, 하나의 요청을 처리하는 데 1ms가 걸린다는 것이 드러났다고 가정해보자. 초당 100,000건의 요청을 처리하려면 대기열이 얼마나 커야 할까? 다시 한번 숫자들을 연결해보자.

$Lr - 3r=100,000r/s*0.0001s$
$Lr-3r=10r$
$Lr=7r$

마찬가지로 파이프라인은 3개의 단계로 이루어져 있으므로, L을 3만큼 줄일 수 있다. $\lambda$를 100,000r/s로 설정하면, 그 정도 양의 요청을 처리하려면 대기열의 용량이 7이어야 한다는 사실을 알 수 있다. 대기열의 크기를 늘리면 시스템을 통해 작업하는 데 시간이 오래 걸린다! 실질적으로 시스템의 활용성과 지연을 맞바꾸는 것이다.

리틀의 법칙이 통찰력을 제공할 수 없는 분야는 실패에 대한 처리 분야다. ==어떤 이유로든 파이프라인에 패닉이 발생하면 대기열에 있는 모든 요청을 잃게 된다는 사실을 염두해 두어야 한다.==


이는 요청을 다시 생성하는 것이 어렵거나 혹은 이루어지지 않을 경우를 방지하기 위한 것일 수도 있다. ==이러한 상황을 줄이기 위해 대기열 크기를 0으로 유지하거나, 필요에 따라 나중에 읽을 수 있도록 어딘가에 보존되는 대기열인 영구 대기열로 옮길 수 있다.==

대기열 사용은 시스템에서 유용할 수 있지만, ==그 복잡성 때문에 개인적인 생각으로는 보통 마지막에 구현할 것을 권한다.==


---

# context 패키지


지금까지 살펴봤듯이, 동시성 프로그램에서 시간 초과, 취소 또는 시스템의 다른 부분의 에러로 인해 작업을 선점해야 하는 경우가 있다. 이전에 done 채널을 만드는 관용적인 구문을 살펴보았다. ==done 채널은 프로그램을 관통해 흐르면서 동시에 수행되는 연산들을 차단하는 모든 것을 취소한다.== 이것은 잘 작동하지만 어떤 면에서는 제한적이다.

취소됐다는 단순한 아람 대신, 취소 이유가 무엇인지, 또는 함수에 완료돼야만 하는 마감 시한이 있는지 등의 추가적인 정보를 전달할 수 있다면 도움이 될 것이다.

시스템의 크기에 상관없이 일반적으로 이들 정보로 done 채널을 감쌀 필요가 없다는 점이 밝혀졌고, Go 언어의 작성자들은 이를 위한 표준 패턴을 만들기로 했다. context 패키지는 표준 라이브러리 밖의 실험에서 비롯됐지만, Go 1.7에서는 표준 라이브러리로 옮겨져, 동시에 실행되는 코드로 작업할 때 고려해야 할 표준 Go 구문이 됐다.

context 패키지를 들여다보면 패키지가 매우 간단하다는 것을 알 수 있다.

``` Go

var Canceled=errors.New("context canceled")
var DeadlineExeeded error=deadlineExceedError{}

type CancelFunc
type Context

func Background() Context
func TODO() Context
func WithCancel(parent Context)(ctx Context,cancel CancelFunc)
func WithDeadline(parent Context,deadline time.Time)(Context,CancelFunc)
func WithTimeout(parent Context,timeout time.Duration)(Context,CancelFunc)
func WithValue(parent Context,key,val interface{}) Context
```

이 타입과 함수들은 뒤에서 다시 살펴볼 것이다. 하지만 지금은 Context 타입에 초점을 맞추자. 이것은 done 채널처럼 시스템 전체를 흐르는 타입이다. context 패키지를 사용하는 경우, 최상위 단계 동시성 호출에서부터 다음 단계로 흘러내려가는 각 함수는 첫 번째 인수로 Context를 받는다. 이 타입은 다음과 같은 모양이다.

```go
type Context interface {

    // Deadline returns the time when work done on behalf of this context
    // should be canceled. Deadline returns ok==false when no deadline is
    // set. Successive calls to Deadline return the same results.
    Deadline() (deadline time.Time, ok bool)

  

    // Done returns a channel that's closed when work done on behalf of this
    // context should be canceled. Done may return nil if this context can
    // never be canceled. Successive calls to Done return the same value.
    // The close of the Done channel may happen asynchronously,
    // after the cancel function returns.

    //

    // WithCancel arranges for Done to be closed when cancel is called;
    // WithDeadline arranges for Done to be closed when the deadline
    // expires; WithTimeout arranges for Done to be closed when the timeout
    // elapses.

    //

    // Done is provided for use in select statements:

    //

    //  // Stream generates values with DoSomething and sends them to out
    //  // until DoSomething returns an error or ctx.Done is closed.
    //  func Stream(ctx context.Context, out chan<- Value) error {

    //      for {
    //          v, err := DoSomething(ctx)
    //          if err != nil {
    //              return err
    //          }

    //          select {
    //          case <-ctx.Done():
    //              return ctx.Err()
    //          case out <- v:
    //          }
    //      }
    //  }
    //

    // See https://blog.golang.org/pipelines for more examples of how to use
    // a Done channel for cancellation.
    Done() <-chan struct{}

  

    // If Done is not yet closed, Err returns nil.
    // If Done is closed, Err returns a non-nil error explaining why:
    // Canceled if the context was canceled
    // or DeadlineExceeded if the context's deadline passed.
    // After Err returns a non-nil error, successive calls to Err return the same error.

    Err() error

  

    // Value returns the value associated with this context for key, or nil
    // if no value is associated with key. Successive calls to Value with
    // the same key returns the same result.

    //

    // Use context values only for request-scoped data that transits
    // processes and API boundaries, not for passing optional parameters to
    // functions.

    //

    // A key identifies a specific value in a Context. Functions that wish
    // to store values in Context typically allocate a key in a global
    // variable then use that key as the argument to context.WithValue and
    // Context.Value. A key can be any type that supports equality;
    // packages should define keys as an unexported type to avoid
    // collisions.

    //

    // Packages that define a Context key should provide type-safe accessors
    // for the values stored using that key:

    //

    //  // Package user defines a User type that's stored in Contexts.
    //  package user

    //

    //  import "context"
    //

    //  // User is the type of value stored in the Contexts.
    //  type User struct {...}

    //

    //  // key is an unexported type for keys defined in this package.
    //  // This prevents collisions with keys defined in other packages.
    //  type key int

    //

    //  // userKey is the key for user.User values in Contexts. It is
    //  // unexported; clients use user.NewContext and user.FromContext
    //  // instead of using this key directly.
    //  var userKey key

    //

    //  // NewContext returns a new Context that carries value u.
    //  func NewContext(ctx context.Context, u *User) context.Context {
    //      return context.WithValue(ctx, userKey, u)
    //  }

    //

    //  // FromContext returns the User value stored in ctx, if any.
    //  func FromContext(ctx context.Context) (*User, bool) {
    //      u, ok := ctx.Value(userKey).(*User)
    //      return u, ok
    //  }

    Value(key any) any
}
```

이것 역시 굉장히 간단해 보인다. ==함수가 선점당했을 대 닫힌 채널을 리턴하는 Done 메서드도 있다. 또한 새롭지만 이해하기 쉬운 메서드도 있다.== 바로 얼마만큼의 시간이 지난 후에 고루틴이 취소될지를 나타내는 Deadline 함수와, 고루틴이 취소된 경우 nil이 아닌 값을 반환하는 Err 메서드다. 그러나 Value 메서드는 다소 어색해 보인다. 이 메서드의 용도는 무엇일까?

Go 언어의 작성자는 고루틴의 주된 용도 중 하나가 요청을 처리하는 프로그램이라는 것에 주목했다. ==일반적으로 이러한 프로그램들에서는 선점에 대한 정보 이외에 요청에 특화된 정보도 함께 전달해야 한다.== 이것이 Value 함수의 목적이다. 이 부분에 대해서는 조금 더 이야기하겠지만, 여기서는 context 패키지가 두 가지 주요 목적으로 사용된다는 것을 알야아 한다.

- 호출 그래프상의 분기를 취소하기 위한 API 제공
- 호출 그래프를 따라 요청 범위(request-scope) 데이터를 전송하기 위한 데이터 저장소(data-bag)의 제공

첫 번째 목적인 취소에 초점을 맞춰보자.

134페이지의 "고루틴 누수 방지"에서 배웠던 것처럼 함수에서의 취소는 세 가지 측면이 있다.

- 고루틴의 부모가 해당 고루틴을 취소하고자 할 수 있다.
- 고루틴이 자신의 자식을 취소하고 할 수 있다.
- 고루틴 내의 모든 대기 중인 작업을 취소될 수 있도록 선점 가능할 필요가 있다.


context 패키지를 사용하면 이 세 가지를 모두 관리할 수 있다.


앞서 언급했듯이, Context 타입이 함수의 첫 번째 인수가 된다. ==Context 인터페이스의 메서드를 살펴보면, 하부의 구조체 상태를 변경할 수 있는 것이 없다는 것을 알 수 있다. ==게다가 Context를 받아들이는 함수에 그것을 취소할 수 있도록 해주는 것은 없다. ==이 때문에 컨텍스트를 취소하는 자식으로부터 호출 스택 위쪽의 함수가 보호된다.== done 채널을 제공하는 Done 메서드와 이것을 함께 사용하면, 부모들로부터의 취소를 Context 타입에서 안전하게 관리할 수 있다.


여기서 한 가지 궁금증이 생길 수 있다. Context 불변(immutable)이라면, 어떻게 호출 스택에서 현재 함수 아래에 있는 함수들이 취소 동작에 영향을 미칠까?

여기서 context 패키지의 함수들이 중요해진다. 기억을 되살리기 위해 그 함수들 중 몇 가지를 다시 한번 살펴보자.

``` go
func WithCancel(parent Context)(ctx Context,cancel CancelFunc)
func WithDeadline(parent Content,deadline time.Time)(Context, CancelFunc)
func WithTimeout(parent Context,timeout time.Duration)(Context,CancelFunc)
```

이 함수들을 모두 Context를 인자로 받고 Context를 리턴한다는 점에 유의하자. 이들 중 일부는 데드라인 및 시간 초과와 같은 다른 인수도 받는다. 이 함수들은 모두 각 함수와 관련된 옵션들로 Context의 새 인스턴스를 생성한다.

WithCancel는 리턴된 cancel 함수가 호출될 때 done 채널을 닫는 새 Context를 리턴한다. WithDeadline은, 기기의 시계가 deadline을 넘었을 때 done 채널을 닫는 새로운 Context를 리턴한다. WithTimeout은 주어진 timeout 지속 시간 후, done 채널을 닫는 새로운 Context 를 리턴한다.


함수가 호출 그래프에서 아래쪽에 있는 함수들을 모종의 방법으로 취소해야 하는 경우, ==함수는 이러한 함수들 중 하나를 호출해 주어진 Context를 전달하고, 리턴된 Context를 자식들에게 전달한다.== 함수가 취소 동작을 수정하지 않아도 되는 경우, 함수는 단순히 주어진 Context를 전달만 한다.

이런 방식으로 호출 그래프의 연속적인 레이어는 부모에게 영향을 주지 않으면서 자신의 요구사항에 부합하는 Context를 생성할 수 있다. 이것은 호출 그래프의 분기를 관리하는 방법에 대한 구성 가능하고 우아한 해결책을 제공한다.

이러한 의미에서, ==Context의 인스턴스는 프로그램의 호출을 통해 전달된다.== 객체 지향 패러다임에서는 자주 사용되는 데이터에 대한 참조를 멤버 변수로 저장하는 것이 일반적이지만, context.Context의 인스턴스에서는 이렇게 하지 않는 것이 중요하다. ==context.Context의 인스턴스는 겉에서 보기에는 똑같아 보일 수 있지만, 내부적으로는 모든 스택 프레임에서 변경될 수 있다.== ==이러한 이유로 Context의 인스턴스를 항상 함수들로 전달하는 것이 중요하다.== 이런 식으로 함수들은 스택에 N 레벨만큼 쌓아올린 스택 프레임을 위한 컨텍스트가 아닌, 자신들을 위한 켄텍스트를 가지게 된다.


비동기 호출 그래프의 맨 위에서는 코드가 컨텍스트를 전달받지 못했을 것이다. 체인을 시작하기 위해 context 패키지는 Context의 빈 인스턴스를 만드는 함수를 두 가지 제공한다.

``` go
func Background() Context
func TODO() Context
```

Background는 단순히 빈 Context를 리턴한다. TODO는 운영 환경에서 사용하기 위한 것은 아니지만 마찬가지로 빈 Context를 리턴한다. TODO의 의도된 목적은 활용할 Context를 알지 못하거나, Context를 제공받기는 기대하지만 상류 코드가 아직 코드를 제공하지 않았을 때 **이러한 자리를 표시하는 역할**(placeholder)을 하는 것이다.

그럼 이 모든 것을 사용해보자. done 채널 패턴을 사용하는 예를 살펴보고, context 패키지를 사용하도록 전환했을 때 얻을 수 있는 이점을 살펴보겠다.

```go

package main

  

import (
    "fmt"
    "sync"
    "time"
)

  

func main() {

    var wg sync.WaitGroup
    done := make(chan interface{})
    defer close(done)

  

    wg.Add(1)

    go func() {
        defer wg.Done()
        if err := printGreeting(done); err != nil {
            fmt.Printf("%v", err)
            return
        }
    }()

  

    wg.Add(1)

    go func() {
        defer wg.Done()

        if err := printFarewell(done); err != nil {
            fmt.Printf("%v", err)
            return
        }
    }()

    wg.Wait()
}

  

func printGreeting(done <-chan interface{}) error {
    greeting, err := genGreeting(done)

    if err != nil {
        return err
    }
    fmt.Printf("%s world! \n", greeting)
    return nil

}

  

func printFarewell(done <-chan interface{}) error {

    farewell, err := genFarewell(done)

    if err != nil {
        return err
    }
    fmt.Printf("%s world!\n", farewell)
    return nil
}

  

func genGreeting(done <-chan interface{}) (string, error) {

    switch locale, err := locale(done); {
    case err != nil:
        return "", err
    case locale == "EN/US":
        return "hello", nil
    }
    return "", fmt.Errorf("unsupported locale")
}

  

func genFarewell(done <-chan interface{}) (string, error) {

    switch locale, err := locale(done); {
    case err != nil:
        return "", err
    case locale == "EN/US":
        return "goodbye", nil
    }
    return "", fmt.Errorf("unsupported locale")
}

  

func locale(done <-chan interface{}) (string, error) {

    select {
    case <-done:
        return "", fmt.Errorf("canceled")
    case <-time.After(1 * time.Minute):
    }
    return "EN/US", nil

}
```
이 코드를 실행한 결과는 다음과 같다. 


``` go
PS C:\Users\kgm09\goproject\src> go run main.go
hello world! 
goodbye world!
```

레이스 컨디션은 무시하고(안부 인사를 하기 전에 작별 인사를 받을 수도 있다!). 이 프로그램에 동시에 실행되는 두 개의 분기가 있다는 사실을 알 수 있다. done 채널을 만들고 이를 호출 그래프를 통해 전달함으로써 표준 선점 방법을 설정했다. 

main의 어느 지점에서든 done 채널을 닫으면 두 채널이 취소된다. main에 고루틴을 도입함으로써, 이 프로그램을 몇 가지 흥미롭고 색다른 방식으로 제어할 수 있는 가능성을 열었다. genGreeting이 너무 오래 걸리는 경우 시간 초과되기를 원할 수도 있다. 부모가 곧 취소될 것이라는 것을 알고 있다면, 아마도 genFarewell이 locale을 호출하지 않기를 바랄 수도 있다. 각 스택 프레임에서 함수는 그 아래에 호출 스택 전체에 영향을 줄 수 있다.


done 채널 패턴을 사용하는 경우, 전달받은 done 채널을 다른 done 채널로 감싼 다음, 그 중 하나가 실행되면 리턴하는 식으로 동일하게 이를 구현할 수 있다. ==하지만 이 경우에는 Context에서 제공하는 마감 시한이나 에러에 대한 추가 정보는 얻을 수 없다.==


done 채널 패턴을 사용하는 것과 context 패키지를 사용하는 것을 보다 쉽게 비교해 보기 위해 이 프로그램을 트리로 나타내 보겠다. 트리의 각 노드는 함수의 호출을 나타낸다.

``` mermaid

flowchart TD

node((시작))-->main(main)
locale(locale)
locale2(locale)
main-->pgreet(printGreeting)-->ggreet(genGreeting)-->locale
main-->pgreet2(printGreeting)-->ggreet2(genGreeting)-->locale2
```


이 프로그램을 done 채널 대신 context 패키지를 사용하도록 수정해보자.

이제 context.Context의 유연성을 가지고 있기 때문에 재미있는 시나리오를 도입할 수 있다.

genGreeting이 locale에 대한 호출을 포기하기 전에 1초만 기다리고 싶다고 가정해 보자. 즉, 1초의 타임 아웃이다. 또한 main에 스마트한 로직을 구축하려고 한다. ==만약 printGreeting이 성공적이지 못했다면, printFarewell에 대한 호출도 취소하려는 것이다.== 어쨌거나 안부 인사를 건내지도 않았는데 작별 인사를 하는 것은 말이 안 된다.

다음은 context 패키지를 이용해 구현하는 것이다.



``` go

package main

  

import (
    "context"
    "fmt"
    "sync"
    "time"
)

  

func main() {

    var wg sync.WaitGroup
    ctx, cancel := context.WithCancel(context.Background()) // #1
    defer cancel()

  

    wg.Add(1)
    go func() {
        defer wg.Done()

        if err := printGreeting(ctx); err != nil {
            fmt.Printf("cannot print greeting: %v\n", err)
            cancel() // #2
        }
    }()

  

    wg.Add(1)

    go func() {
        defer wg.Done()
        if err := printFarewell(ctx); err != nil {
            fmt.Printf("cannot print farewell:%v\n", err)
        }
    }()
    wg.Wait()
}

  

func printGreeting(ctx context.Context) error {

    greeting, err := genGreeting(ctx)
    if err != nil {
        return err
    }

    fmt.Printf("%s world! \n", greeting)
    return nil

}

  

func printFarewell(ctx context.Context) error {

    farewell, err := genFarewell(ctx)

    if err != nil {
        return err
    }
    fmt.Printf("%s world! \n", farewell)
    return nil
}

  

func genGreeting(ctx context.Context) (string, error) {

    ctx, cancel := context.WithTimeout(ctx, 1*time.Second) // #3
    defer cancel()

    switch locale, err := locale(ctx); {
    case err != nil:
        return "", err

    case locale == "EN/US":
        return "hello", nil
    }
    return "", fmt.Errorf("unsupported locale")
}

  

func genFarewell(ctx context.Context) (string, error) {

    switch locale, err := locale(ctx); {

    case err != nil:
        return "", err

    case locale == "EN/US":
        return "goodbye", nil
    }
    return "", fmt.Errorf("unsupported locale")
}

  

func locale(ctx context.Context) (string, error) {

    select {
    case <-ctx.Done():
        return "", ctx.Err() // #4
    case <-time.After(1 * time.Minute):
    }
    return "EN/US", nil
}
```

1. 여기서 main은 context.Background()로 새로운 Context를 만들고 취소를 허용하기 위해 이를 context.WithCancel로 감싼다.
2. printGreeting에서 리턴된 에러가 있을 경우에 main은 이 줄에서 Context를 취소한다.
3. 여기서 genGreeting은 자신의 Context를 context.WithTimeout으로 감싼다. 이렇게 하면 리턴된 Context가 1초 후에 자동적으로 취소될 것이고, 이에 따라 이 함수가 Context를 전달한 모든 함수, 다시 말해 locale 함수 역시 취소될 것이다.
4. 여기서는 Context가 취소될 이유를 리턴한다. ==이 에러는 위쪽으로 전달되어 2의 취소를 발생시킨 main까지 올라갈 것이다.==


다음은 이 코드의 실행 결과이다.

``` go

PS C:\Users\kgm09\goproject\src> go run main.go
cannot print greeting: context deadline exceeded
cannot print farewell:context canceled

// 컨텍스트 제한 시간 초과
 // 컨텍스트 취소
```

무슨 일이 일어난 것인지 보기 위해 호출 그래프를 그려보자. 여기의 숫자는 앞의 예제에서 코드를 설명 줄에 해당한다.

[그림]


출력을 통해 시스템이 완벽하게 작동한다는 것을 알 수 있다. locale이 실행하는 데 최소 1분이 걸리기 때문에, genGreeting의 호출은 항상 제한 시간이 초과될 것이고, 이것은 main이 항상 printFarewell 아래의 호출 그래프를 취소한다는 것을 의미한다.

genGreeting이 부모의 Context에 아무런 영향을 미치지 않고 자신만의 필요에 맞는 context.Context를 구축할 수 있었던 방법에 주목하자. ==만약 genGreeting이 성공적으로 리턴됐고, printGreeting이 한 번 더 genGreeting을 호출해야 했다면, genGreeting이 어떻게 동작했는지에 대한 정보를 유출하지 않고도 그렇게 할 수 있었을 것이다.== 이러한 구성 가능성으로 인해 호출 그래프상에서 관심사가 뒤섞이지 않으면서도 커다란 시스템을 작성할 수 있다.

이 프로그램을 더 개선할 수 있다. locale이 실행되는 데 약 1분이 걸린다는 점을 알고 있으므로 locale 내부에서 마감 시한이 주어졌는지 확인할 수 있고, 마감 시한에 걸릴지도 모른다. 이 예제는 context.Context의 Deadline 메서드를 사용해 그 작업을 수행하는 것을 보여준다.

```go
package main

  

import (
    "context"
    "fmt"
    "sync"
    "time"
)

  

func main() {

    var wg sync.WaitGroup
    ctx, cancel := context.WithCancel(context.Background()) // #1
    defer cancel()

  

    wg.Add(1)
    go func() {
        defer wg.Done()
        if err := printGreeting(ctx); err != nil {
            fmt.Printf("cannot print greeting: %v\n", err)
            cancel() // #2
        }
    }()

  

    wg.Add(1)

    go func() {
        defer wg.Done()
        if err := printFarewell(ctx); err != nil {
            fmt.Printf("cannot print farewell:%v\n", err)
        }
    }()
    wg.Wait()

  

}

  

func printGreeting(ctx context.Context) error {
    greeting, err := genGreeting(ctx)

    if err != nil {
        return err
    }

    fmt.Printf("%s world! \n", greeting)
    return nil

}

  

func printFarewell(ctx context.Context) error {
    farewell, err := genFarewell(ctx)
    if err != nil {
        return err
    }
    fmt.Printf("%s world! \n", farewell)
    return nil
}

  

func genGreeting(ctx context.Context) (string, error) {
    ctx, cancel := context.WithTimeout(ctx, 1*time.Second) // #3
    defer cancel()

    switch locale, err := locale(ctx); {

    case err != nil:
        return "", err

    case locale == "EN/US":
        return "hello", nil

    }
    return "", fmt.Errorf("unsupported locale")
}

  

func genFarewell(ctx context.Context) (string, error) {

    switch locale, err := locale(ctx); {
    case err != nil:
        return "", err

    case locale == "EN/US":
        return "goodbye", nil
    }
    return "", fmt.Errorf("unsupported locale")
}

  

func locale(ctx context.Context) (string, error) {

    if deadline, ok := ctx.Deadline(); ok { // #1
        if deadline.Sub(time.Now().Add(1*time.Minute)) <= 0 {
            return "", context.DeadlineExceeded
        }
    }

    select {

    case <-ctx.Done():
        return "", ctx.Err()
    case <-time.After(1 * time.Minute):
    }

    return "EN/US", nil
}
```

1. 여기서는 Context에 마감 시한이 주어졌는지 확인한다. 마감 시한이 주어졌고, ==시스템 클록이 마감 시한을 넘겼다면 context 패키지에 정의된 특별한 에러인 DeadlineExceeded를 리턴한다.==

프로그램에서 이 반복의 차이는 작지만, locale 함수가 빠르게 실패할 수 있다. ==기능상의 다음 부분을 호출하는데 높은 비용을 초래할 수 있는 프로그램에서는 이러한 빠른 실패를 통해 상당한 시간을 절약할 수 있지만==, ==그렇지 않은 경우라고 해도 최소한 실제 시간 초과를 기다리지 않고 즉시 실패할 수 있다.== 유일한 문제는 하위의 호출 그래프가 얼마나 오래 걸리는지 알고 있어야 하는 점인데, 이는 매우 어려울 수 있다.

이로 인해 context 패키지가 제공하는 나머지 절반, 즉, ==요청 범위 데이터를 저장하고 조회할 수 있는 Context용 데이터 저장소를 사용하게 된다.== ==함수가 고루틴과 Context를 생성할 때, 많은 경우에 요청을 처리할 프로세스를 시작하며, 스택의 아래쪽에 있는 함수는 요청에 대한 정보를 필요로 하낟는 것을 잊지 말자.== 다음은 Context에 데이터를 저장하고 조회하는 예제다.


``` go

package main

  

import (
    "context"
    "fmt"
)

  

func main() {
    ProcessRequest("jane", "abc123")
}

  

func ProcessRequest(userID, authToken string) {
    ctx := context.WithValue(context.Background(), "userID", userID)
    ctx = context.WithValue(ctx, "authToken", authToken)
    HandleResponse(ctx)

}

  

func HandleResponse(ctx context.Context) {

    fmt.Printf(
        "handling response for %v (%v)",
        ctx.Value("userID"),
        ctx.Value("authToken"),
    )
}
```

실행 결과는 다음과 같다.

``` shell
PS C:\Users\kgm09\goproject\src> go run main.go
handling response for jane (abc123)
```

매우 간단하다. 이를 위한 유일한 자격 조건은 다음과 같다.

- 사용하는 키는 Go의 **비교 가능성** 개념을 충족시켜야 한다. 즉, 항등 연산자 == 및 !=을 사용하면 올바른 결과를 리턴해야 한다.
- 리턴된 값은 여러 고루틴에서 접근할 때 안전해야 한다.


==Context의 키와 값이 interface{}로 정의되어 있기 때문에, 값을 검색할 때 Go의 타입 안전성을 잃어버리게 된다.== 키는 다른 타입이거나 우리가 제공한 키와 약간 다를 수 있다. 값도 기대하는 타입과는 다를 수도 있다. **이러한 이유로 Go 언어 작성자는 Context에서 값을 저장하고 조회할 대 몇 가지 규칙을 따를 것을 권고한다.**

**첫 번째로 패키지에 맞춤형 키 타입을 정의할 것을 추천한다.** ==다른 패키지도 똑같이 맞춤형 키 타입을 정의하기만 하면 Context 내에서의 충돌도 방지할 수 있다.== 그 이유를 알려주기 위해 맵에 타입은 다르지만 같은 값을 가지는 키를 저장하려고 시도하는 짧은 프로그램을 살펴보겠다.

``` go
package main

  

import (
    "fmt"
)

type foo int
type bar int


func main() {

    m := make(map[interface{}]int)

    m[foo(1)] = 1
    m[bar(1)] = 2

    fmt.Printf("%v", m)

}
```

실행 결과는 다음과 같다.

``` go
PS C:\Users\kgm09\goproject\src> go run main.go
map[1:2 1:1]
```

근본적인 값은 동일하지만 타입이 다른 정보는 맵 내에서 서로 구분된다. ==패키지의 키를 위해 정의한 타입은 외부로 내보내지 않으므로 다른 패키지 내에서 생성한 키와 충돌할 수 있다.==

==데이터를 저장하는데 사용하는 키를 내보내지 않을 것이기 때문에, 데이터를 검색하는 함수를 내보내야 한다.== 이 방식은 이 데이터의 소비자가 정적인, **타입-안전**(type-safe) 함수를 사용할 수 있도록 하기 때문에 멋지게 동작한다.

이 모든 것을 하나로 합치면 다음과 같은 예제를 얻을 수 있다.

``` go
package main

  

import (
    "context"
    "fmt"
)

  

func main() {
    ProcessRequest("jane", "abc123")
}

  

type ctxKey int

  

const (
    ctxUSerID ctxKey = iota
    ctxAuthToken
)

  

func UserID(c context.Context) string {
    return c.Value(ctxUSerID).(string)
}

  

func AuthToken(c context.Context) string {
    return c.Value(ctxAuthToken).(string)
}

  

func ProcessRequest(userID, authToken string) {
    ctx := context.WithValue(context.Background(), ctxUSerID, userID)
    ctx = context.WithValue(ctx, ctxAuthToken, authToken)
    HandleResponse(ctx)

}

  

func HandleResponse(ctx context.Context) {

    fmt.Printf(
        "handling response for %v (auth: %v)",
        UserID(ctx),
        AuthToken(ctx),
    )

}
```

이 코드의 실행 결과는 다음과 같다.

``` shell
PS C:\Users\kgm09\goproject\src> go run main.go
handling response for jane (auth: abc123)
```

이제 타입-안전 방식으로 Context에서 값을 조회할 수 있는 방법을 확보했다. 만약 소비자가 다른 패키지에 있다면 정보를 저장하는 데 어떤 키를 사용했는지 알지도 못하고 관심도 없을 것이다. 그러나 이 기법에는 문제가 있다.

앞의 예제에서 HandleResponse가 response라는 다른 패키지 엤다고 가정해보겠다. 그리고 process 패키지 내에 ProcessRequest 패키지가 있다고 가정해보자. process 패키지는 HandleResponse를 호출하기 위해 response 패키지를 가져와야(import) 하지만, 

==HandleResponse는 process 패키지에 정의된 접근자 함수에 접근할 수 없다. process를 임포트할 대 순환 의존성을 발생시키기 때문이다.==

==Context에 키를 저장하는 데 사용되는 타입은 process패키지에 대해 비공개(private)이기 때문에 response 패키지에는 이 데이터를 조회할 수 있는 방법이 없다!==

이로 인해 여러 위치에서 가져온 데이터 타입을 중심으로 패키지를 생성하는 아키텍쳐를 강제할 수 밖에 없다. 이 방식이 확실하게 나쁘다고 할 수는 없지만, 알아둬야 할 것이 있다. context 패키지는 꽤 깔끔하지만, 언제나 칭찬을 받지는 못했다. Go 커뮤니티 내에서 context 패키지는 다소 논란의 여지가 있다. 이 패키지의 취소 측면은 꽤 잘 받아들여졌지만, Context에 임의의 데이터를 저장할 수 있는 기능과 타입에 안전하지 않은 (type-unsafe) 방식의 데이터 저장으로 인해 논란이 발생했다. **접근자 함수**로 타입 안정선을 부분적으로 완화했지만, 잘못된 타입을 저장해 버그를 품고 있을 수도 있다.

그러나 더 큰 문제는 개발자가 Context의 인스턴스에 저장해야만 하는 특성이다.

API나 프로세스 경계를 통과하는 요청 범위(request-scoped)의 데이터에 대해서만 컨텍스트 값을 사용하고, 함수에 선택적 매개 변수를 전달하는 용도로는 사용하지 않는다.

선택적 매개 변수가 무엇인지는 명확하게 알 수 있다. Go가 선택적 매개 변수를 지원하도록 하고 싶은 은밀한 욕망을 위해 Context를 사용해서는 안 된다. 그런데 **"요청 범위 데이터"** 란 무엇인가? context 패키지 내의 주석에는 "API나 프로세스의 경계를 통과하는 요청 범위 데이터"라는 내용이 있다. ==이 주석을 보면, 요청 범위 데이터는 프로세스와 API 경계를 통과하는 것으로 보이는데, 그 상황 자체가 주석에 명확히 나타나 있지 않아서 여러 가지 상황을 의미할 수 있다. 이를 정의하는 가장 좋은 방법은, 팀 내에서 몇 가지 경험적인 방법을 통해 코드 검토에서 평가하는 것이다.== 다음은 필자의 개인적인 경험적인 규칙이다.

1. 데이터가 API나 프로세스 경계를 통과해야 한다.

프로세스의 메모리 내에서 데이터를 생성하는 경우, API 경계를 넘지 않는 한, 요청 범위의 데이터가 될 가능성이 적다.

2. 데이터는 변경 불가능(immutable)해야 한다.

불변 값이 아니라면 그 정의에 따라, 당신이 저장하고 있는 값은 요청으로부터 온 것이 아니다.

3. 데이터는 단순한 타입으로 변해야 한다.

요청 범위 데이터가 프로세스 및 API 경계를 통과하기 위한 것이라면, 패키지들의 복잡한 그래프를 임포트할 필요가 없으며, 다른 쪽에서 이 데이터를 가져올 수 있다.


4. 데이터는 메서드가 있는 타입이 아닌 데이터여야 한다.

연산은 논리이며, 이 데이터를 소비하는 것이 연산을 보유해야 한다.

5. 데이터는 연산을 주도하는 것이 아니라 꾸미는 데 도움이 되어야 한다.

Context에 무엇이 포함됐는지, 혹은 포함되지 않았는지에 따라 알고리즘이 다르게 동작하는 경우, 선택적 매개 변수의 영역으로 넘어갔을 가능성이 크다.


이들은 고정 불변의 규칙이 아닌 경험적인 규칙이다. 그==러나 Context에 저장하는 데이터가 이 가이드라인 5개를 모두를 위반하는 것으로 밝혀지면, 수행하려는 작업을 자세히 살펴봐야 할 수도 있다.==


또한, 이 데이터가 활용되기 전에 거쳐가야 할 레이어의 수도 고려해야 한다. 데이터를 받은 곳과 사용되는 곳 사이에 몇 개의 프레임워크와 수십 가지 함수가 있다면, 장황하고 코드 자체가 문서화인 함수의 시그니처들에 기대어 데이터를 매개 변수로 추가하기를 원하는가? 아니면 데이터를 Context에 배치해 보이지 않는 의존성을 만들겠는가? 각 접근 방식마다 장점이 있으며, 결국에는 스스로가 결정해야 한다.


이러한 경험적인 방법을 사용해도 어떤 값이 요청 범위에 속하는지 아닌지는 대답하기 어려운 질문이다. 다음 표를 살펴보자. 이 표는 각 타입의 데이터가 위에 나열된 다섯 가지 경험 법칙을 충족하는지 여부에 대한 개인적인 의견이다. 



| 데이터       | 1   | 2   | 3   | 4   | 5   |
| --------- | --- | --- | --- | --- | --- |
| 요청 ID     | V   | V   | V   | V   | V   |
| 사용자 ID    | V   | V   | V   | V   |     |
| URL       | V   | V   |     |     |     |
| API 서버 연결 |     |     |     |     |     |
| 인증 토큰     | V   | V   | V   | V   |     |
| 요청 토큰     | V   | V   | V   |     |     |

API 서버 연결처럼 컨텍스트에 저장해서는 안 된다는 것이 명확할 수도 있고, 그렇지 않을 수도 있다. **인증 토큰**의 경우는 어떤가? 이것은 불변값이고 byte의 슬라이스일 수 있지만, 이 데이터의 수신자가 요청의 처리 여부를 결정하는데 이 값을 사용하지 않았을까? 이 데이터는 컨텍스트에 속하는가? 더 진흙탕으로 만들자면, 한 팀에서는 받아들여지는 것이 다른 팀에서는 받아들여지지 않을 수도 있다.

여기에는 절대적으로 쉬운 대답이 존재하지 않는다. ==이 패키지는 표준 라이브러리에 포함돼 있으므로, 이를 사용하는데 있어 나름의 의견을 형성해야 하지만, 그 의견은 어떤 프로젝트를 다루고 있는지에 따라 달라질 수 있다.== 마지막으로 하고 싶은 조언은 Context에서 제공하는 취소 기능은 매우 유용하며, 데이터 저장소에 대한 당신의 생각 때문에 이를 사용하는 것을 단념해서는 안 된다는 것이다.


[^1]: unsafe 패키지를 통해 메모리를 수동으로 조작할 가능성에 대해서는 무시한다. 안전하지 않은(unsafe)것에는 다 이유가 있다.

[^2]: 언어의 관점에서 구체화란, 개발자가 어떤 개념을 대상으로 직접 작업할 수 있도록 언어에서 해당 개념을 노출시켜주는 것을 의미한다. Go 함수는 시그니처(function signature) 타입을 가지는 변수를 정의할 수 있기 때문에, Go 함수는 구체화돼 있다고 한다. 이는 또한 프로그램에서 함수를 전달할 수 있음을 의미한다.