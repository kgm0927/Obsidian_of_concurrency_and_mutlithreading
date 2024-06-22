
3장에서는 Go의 동시성을 지원하는 풍부하고 정교한 기능에 대해 설명한다. 3장을 배우고 나면 활용할 수 있는 문법과 함수, 패키지로는 무엇이 있는지 파악할 수 있다. 또한 각 기능에 대해서도 이해할 수 있다. 


---

### 고루틴

고루틴은 Go 프로그램을 구성하는 가장 기본적인 단위 중 하나다. 실제 모든 Go 프로그램에서는 적어도 하나의 고루틴이다. ==프로세스가 시작될 때 자동으로 생성되고 시작되는 main **고루틴**이 바로 그것이다.== 거의 모든 프로그램에서 문제를 해결할 때 언젠가는 고루틴을 사용하게 될 것이다. 그럼 고루틴이 무엇인가?


간단히 말하면 고루틴은 다른 코드와 함께 동시에 실행되는 함수이다. 그렇다고 고루틴이 반드시 병렬로 실행되는 것은 아니다! 함수 앞에 go 키워드를 보면 간단히 시작할 수 있다.

``` go
func main(){
go sayHello()
}

func sayHello(){
fmt.Println("hello")
}

```



익명 함수(anonymous function)도 동작한다! 다음은 앞의 예제와 같은 동작을 하는 예제이다. 그러나 함수로 고루틴을 만드는 것이 아니라 익명 함수로 고루틴을 만든다.


``` go
go func(){
fmt.Println("hello")
}()// #1
```

1. go 키워드를 사용하기 위해 익명 함수를 즉시 호출해야 한다는 점에 유의한다.

이렇게 하는 대신 함수를 변수에 할당하고, 다음과 같이 익명 함수를 호출할 수 있다.

``` go

sayHello := func()
{
fmt.Println("hello")
}

go sayHello()
// 다른 작업들을 계속한다.
```


함수 하나와 키워드 하나만 사용해 동시에 실행되는 논리 블록을 만들 수 있다! 믿거나 말거나 이것이 고루틴을 시작하기 위해 알아야 할 전부다. 고루틴을 적절하게 사용하고, 동기화하고, 구성하는 방법에 관해서는 할 말이 많지만, 실제로 고루틴을 활용하는 데는 위 내용만 알면 충분하다. 3장의 나머지 부분에서는 고루틴이란 무엇이며, 어떻게 작동하는지 자세히 설명한다.

그러면 보이지 않는 곳에서 무슨 일이 일어나고 있는지 살펴보자. 고루틴은 실제로 어떻게 동작하는가? 고루틴은 OS 스레드인가? 아니면 그린 스레드일까? 고루틴을 얼마나 많이 만들 수 있을까?

일부 다른 언어는 비슷한 동시성 기본 요소가 있기는 하지만, 고루틴은 Go에만 존재한다. 고루틴은 OS 스레드가 아니다. 언어의 런타임에 의해 관리되는 스레드인 그린 스레드(green thread)도 아니다. ==고루틴은 **코루틴**(coroutine)이라 불리는 더 높은 수준의 추상화이다. 코루틴은 단순히 동시에 실행되는 서브루틴(함수, 클로저, 또는 Go의 메서드)으로서, **비전점적**(nonpreemptive), 다시 말해 인터럽트할 수 없다.== 대신 코루틴은 잠시 중단(suspend)하거나 재진입(reentry)할 수 있는 여러 개의 지점을 가지고 있다.


고루틴을 Go만의 고유한 특징으로 만드는 것은 바로 Go 런타임과의 긴밀한 통합이다. 고루틴은 자신의 일시 중단 시점이나 재진입 지점을 정의하지 않는다. Go의 런타임은 고루틴의 실행 시 동작을 관찰해, 고루틴이 멈춰서 대기(block) 중일 때 자동으로 일시 중단시키고, 대기가 끝나면 다시 시작시킨다. ==Go 런타임이 이런 식으로 고루틴을 선점 가능하게 해 주기는 하지만, 고루틴이 멈춰 있는 지점에서만 선점 가능하다.== 이게 바로 런타임과 고루틴 로직 사이의 우아한 파트너십이다. 따라서 고루틴은 코루틴의 특별한 클래스로 간주할 수 있다.


코루틴은 동시에 실행되는 구조물이라고 여겨지지만, 그렇다고 해서 동시성이 코루틴의 속성인 것은 아니며, 고루틴은 코루틴의 일종이므로 고루틴 역시 그렇다. 즉, 누군가가 여러 코루틴이 동시에 주관(host,호스팅)하면서 각 코루틴이 실행될 수 있는 기회를 제공해야 한다. 그렇지 않으면 코루틴은 동시에 실행되지 않을 것이다! 그렇다고 이 말이 코루틴은 절대적으로 병렬적이라고 암시하는 것은 아니라는 점에 유의하자. 병렬로 처리된다는 환상을 심어주기 위해 순차적으로 실해되는 여러 개의 코루틴을 생성할 가능성이 있으며, 실제로 Go에서는 항상 이러한 일이 발생한다.

고루틴을 호스팅하는 Go의 매커니즘은 **M:N스캐줄러**를 구현한 것으로, M개의 그린 스레드를 N개의 OS스레드에 매핑한다는 의미이다. 그런 다음 고루틴은 스레드에 스케줄링 된다. 사용 가능한 그린 스레드보다 더 많은 고루틴이 있을 경우, 스케줄러는 사용 가능한 스레들로 고루틴을 분배하고 이 고루틴들이 대기 상태가 되면 다른 고루틴이 실행될 수 있도록 한다. 이 모든 것이 어떻게 작동하는지에 대해서는 6장에서 논의할 것이며, 3장에서는 Go가 동시성을 어떻게 모델링하는지 설명할 것이다.


Go는 **fork-join 모델**[^1]이라는 동시성 모델을 따른다. fork라는 단어는 프로그램의 어느 지점에서든지 실행의 자식(child) 분기를 만들어 부모와 동시에 실행할 수 있다는 사실을 나타낸다. 그리고 join이라는 단어는 미래의 어느 시점에서 이렇게 동시에 실행된 분기가 다시 합쳐진다는 사실을 나타낸다. 자식 분기가 다시 부모 분기와 합쳐지는 지점을 **합류 지점**(join point)이라고 한다. 이런 구도를 좀 더 자세히 이해하기 위해 다음 그림을 살펴보자.


[그림]


``` go
package main
import (
    "fmt"
    "sync"
)
  

func main() {
    sayHello := func() {
        fmt.Println("hello")
    }

    go sayHello()
}
```


여기서 sayHello 함수가 자체적인 고루틴에서 실행되는 동안 프로그램의 나머지 부분도 계속 실행될 것이다. 이 예제에서는 합류 지점이 없다. sayHello를 실행되는 고루틴은 일정하지 않은 시간이 지난 후에 종료될 것이고, 프로그램의 나머지 부분은 이미 계속 실행되고 있을 것이다.


그러나 이 예제에서는 한 가지 문제가 있다. 작성된 대로 sayHello 함수가 실행될지 아닐지 전혀 알 수 없다는 점이다. 이 고루틴은 Go의 런타임을 통해 실행되도록 생성 및 스케줄링되지만, 실제로는 main 고루틴이 종료되지 전에 실행될 기회를 얻지 못할 수도 있다.

실제로 이 간단한 예제를 실행할 때, 단순함을 위해 main 함수의 나머지 부분을 생략했기 때문에 sayHello에 대한 호출을 호스팅하는 고루틴이 시작되기도 전에 프로그램의 실행이 끝날 것이다. 결과적으로 표준 출력(stdout)에는 "hello"라는 단어가 출력되지 않는다. ==고루틴을 생성해 후에 time.Sleep을 둘 수도 있지만, 이는 실제로 합류 지점을 만드는 것이 아니라 단지 레이스 컨디션을 일으킬 뿐이라는 것을 기억하자.== [[동시성 소개]]인 1장을 보면, 종료되기 전에 실행될 확률이 늘어나기는 하지만 실행을 보장할 수 없다. ==**합류 지점**은 프로그램의 정확성을 보증하고 레이스 컨디션을 제거하는 요소이다.==

합류 지점을 생성하려면 main 고루틴과 sayHello 고루틴을 동기화해야 한다. 이 작업은 여러 가지 방법으로 수행할 수 있지만, 여기서는 78페이지의 "sync 페이지"에서 설명할 내용인 sync.WaitGroup을 사용한다. 이 예제가 어떻게 두 개의 고루틴 사이에 조인 지점을 만드는지 이해하는 것은 현재로서는 중요하지 않다. 다음은 앞서 언급한 문제점을 수정한 예시이다.


``` go
package main

  

import (
    "fmt"
    "sync")

  

func main() {

    var wg sync.WaitGroup

    sayHello := func() {
        defer wg.Done()
        fmt.Println("hello")
    }
    wg.Add(1)
    go sayHello()
    wg.Wait() // #1
}
```


1. 여기가 합류지점이다.

이 예제는 sayHello 함수를 호스팅하는 고루틴이 종료될 때까지 확정적으로 main 고루틴을 멈춰 놓는다. sync.WaitGroup 의 동작 방식에 대해서는 78페이지의 "sync 패키지"에서 배우겠지만, 여기서는 이 예제가 정확하게 동작하도록 만들기 위해 sync.WaitGroup 사용해 합류 지점을 만든다.

이 책에서는 빠르게 고루틴 예제를 만들기 위해 익명 함수를 많이 사용했다. 이제 클로저(closure)로 관심을 돌려보자. 클로저는 자신들이 생성돼 어휘 범위(lexical scope)를 패쇄해 변수들을 캡처한다. ==고루틴에서 클로저를 실행하면 클로저는 이들 변수의 복사본에 대해 동작하는가? 아니면 원본을 참조해 동작하는가?== 한번 시도해보자.


``` go
package main

  

import (
    "fmt"
    "sync"
)


func main() {
    var wg sync.WaitGroup
    salutation := "hello"
    wg.Add(1)
    go func() {
        defer wg.Done()
	        salutation = "welcome" //#1
    }()

    wg.Wait()

    fmt.Println(salutation)

}
```


1. 바로 여기서 고루틴은 salutation 변수의 값을 수정한다.

salutation의 값은 "hello"일까? 아니면 "welcome"일까? 그렇게 생각하는 이유는 무엇인가? 한번 실행해서 결과를 보겠다.


`welcome`

흥미롭다! 고루틴은 자신이 생성된 곳과 동일한 주소 공간에서 실행되기 때문에 프로그램은 "welcome"이라는 단어를 출력한다.

다른 예를 들어보자. 이 프로그램은 무엇을 출력할 것이라고 생각하는가?

``` go
package main
import (
    "fmt"
    "sync"
)

  

func main() {

    var wg sync.WaitGroup
    for _, salutation := range []string{
        "hello", "greetings", "good day",
    } {
        wg.Add(1)
        go func() {
            defer wg.Done()
            fmt.Println(salutation) // #1
        }()
    }
    wg.Wait()
}
```


1. 여기서는 문자열 슬라이스(slice)의 범위에 의해 성생된 반복문의 변수 salutation을 참조한다.
정답은 예상보다 훨씬 까다로우며, Go의 몇 가지 놀랄 만한 특징 중 하나를 보여준다.
대부분은 직관적으로 이 코드가 "hello","greeting","good day"라는 단어를 일련의 비결정적인 순서로 출력한 것이라고 생각할 것이다. 실제 동작을 보면 이러하다.

```
good day
good day
good day
(실제로 이것과 많이 다르기도 하다. 하지만 책에서 나온 것은 이거기 때문에 이렇게 해 놓고 설명을 하도록 하겠다.)
```

무슨 일이 일어났는지 보겠다. 이 예제에서 고루틴은 문자열 타입을 갖는 반복문의 변수 salutation에 대해 닫혀 있는 클로저를 실행 중이다. 루프가 반복될 때 salutation에는 슬라이스 리터럴(literal)의 다음 문자열 값이 할당된다. ==스케줄링 된 고루틴은 미래의 어떤 시점에서든 실행될 수 있기 때문에, 고루틴 내부에서 어떤 값이 출력될지는 결정돼 있지 않다.== 현 컴퓨터에서는 고루틴이 시작되기 전에 루프가 종료될 확률이 높다. 이 말은 곧 salutation 변수가 범위를 벗어났음을 의미한다. 그러면 어떻게 될까? 고루틴이 범위를 벗어난 무언가를 여전히 참조할 수 있을까? 고루틴이 잠재적으로 가비지 컬렉션된 메모리에 접근하지는 않을까?

이를 통해 Go가 메모리를 관리하는 방식을 이해할 수 있다. Go 런타임은 Salutation 변수에 대한 참조가 여전히 이루어지고 있으므로 고루틴이 계속 접근할 수 있도록 메모리를 힙(heap)으로 옮길 것이라는 사실을 알고 있다.


일반적으로 현재 컴퓨터에서는 고루틴이 실행되기도 전에 루프가 종료되므로 salutation은 문자열의 슬라이스의 마지막 값인 "good day"에 대한 참조를 저장하고 있는 힙으로 옮겨지게 되고, 이에 따라 보통은 "good day"가 세 번 출력된다. 이 반복문을 작성하는 올바른 방법은, 클로저 내부로 salutation의 사본을 전달해 고루틴이 실행될 때, 반복문에서 해당 반복 회차(iteration)에 고루틴으로 전달된 데이터를 가지고 동작하게 하는 것이다.

``` go
package main

import (
    "fmt"
    "sync"
)

  

func main() {

    var wg sync.WaitGroup

    for _, salutation := range []string{"hello", "greetings", "good day"} {

        wg.Add(1)

        go func(salutation string) {  // #1
            defer wg.Done()
            fmt.Println(salutation)
        }(salutation) // #2
    }
    wg.Wait()
}
```

1. 여기서 다른 함수처럼 매개 변수를 선언한다. 어떤 일이 발생하는지 보다 확실하게 하기 위해 원래의 salutation 변수를 가린다(shadow).
2. 여기서는 현재 반복 회차의 변수를 클로저로 전달한다. 문자열 구조체의 복사본이 만들어지므로, 고루틴이 실행될 대 적절한 문자열을 참조하리라는 것을 보장할 수 있다.

그러면 다음과 같이 올바른 출력이 나온다.

``` bash
PS C:\Users\kgm09\goproject\src\Concurrency> go run main.go 
good day
hello
greetings
```

이 예제는 예상한 대로 동작하며, 코드가 약간 더 길어졌을 뿐이다.

고루틴은 서로 동일한 주소 공간에서 작동하며, 단순히 함수를 호스팅하는 것이기 때문에, 고루틴을 사용하면 동시에 실행되지 않는 코드를 작성하는 것에서 동시성 코드를 작성하는 것으로 자연스럽게 확장할 수 있다. Go의 컴파일러는 고루틴이 실수로 할당이 해제된 메모리에 접근하지 않도록 메모리상에 고정돼 있는 변수를 잘 처리하기 때문에, 개발자는 메모리 관리 대신 문제 공간에 집중할 수 있다. 그러나 이것이 만능은 아니다.


여러 고루틴이 동일한 주소 공간에 대해 동작할 수 있기 때문에 여전히 동기화에 대해 걱정해야 한다. 앞서 논의한 것처럼, 고루틴이 접근하는 공우 메모리에 대해 동기화된 접근을 하도록 하거나, CSP 기본 요소를 사용해 통신으로 메모리를 공유할 수 있다. 이 기법은 나중에 "채널"과 "sync 패키지"에서 설명한다.


고루틴의 또 다른 이점은 매우 가볍다는 점이다. 다음은 Go의 FAQ에서 발췌한 것이다.

` 새롭게 만들어진 고루틴에게는 몇 킬로바이트 정도가 주어지며, 이것만으로도 충분하다. 혹시 충분하지 않다면 런타임이 자동으로 스택을 저장하기 위한 메모리는 늘리거나 줄인다. 이를 통해 고루틴이 적정 메모리량으로 살아갈 수 있게 해준다. CPU 오버헤드는 평균적으로 함수 호출 한 번에 약 3개의 명령어 정도이다. 실제로 동일한 주소 공간에 수십만 개의 고루틴이 만들어진다. 만약에 고루틴이 그저 스레드와 같다면 고루틴이 훨씬 적어도 시스템 자원이 고갈될 것이다.`


고루틴 하나당 몇 킬로바이트라면 이는 전혀 나쁘지 않다. 직접 검증해보겠다. 그러나 그 전에 고루틴에 관해 흥미로운 점 하나를 먼저 언급하려고 한다. ==가비지 컬렉터는 버려진 고루틴을 회수하기 위한 어떤 조치도 하지 않는다.== 다음과 같은 코드를 작성한다고 해보자.

``` go
go func(){
// <영원히 대기하는 동작>
}()
// 작업 수
```

이 고루틴은 프로세스가 종료될 때까지 대기하고 있을 것이다. 이 문제를 해결하는 방법에 대해서는 4장의 "고루틴 누수 방지"에서 설명한다. 다음 예제에서는 절대 종료되지 않는 고루틴을 실제로 고루틴의 크기를 측정하는데 사용한다.

이 예제는 고루틴이 가비지 컬렉션되지 않는다는 사실과 런타임이 스스로를 검사할 수 있는 기능을 조합해, 고루틴 생성 전후에 할당된 메모리의 양을 측정한다.


``` go
package main

  

import (
    "fmt"
    "runtime"
    "sync"
)

  

func main() {

    memConsumed := func() uint64 {
        runtime.GC()
        var s runtime.MemStats
        runtime.ReadMemStats(&s)
        return s.Sys
    }

  

    var c <-chan interface{}
    var wg sync.WaitGroup
    noop := func() { wg.Done(); <-c } //#1

  

    const numGoroutine = 1e4 // #2
    wg.Add(numGoroutine)
    before := memConsumed() // #3

  

    for i := numGoroutine; i > 0; i-- {
        go noop()
    }

  

    wg.Wait()
    after := memConsumed() // #4
    fmt.Printf("%.3fkb", float64(after-before)/numGoroutine/1000)
}
```


1. 측정을 위해서는 메모리상에 다수의 고루틴이 유지돼야 하며, 이를 위해 고루틴이 절대 종료되지 않도록 해야 한다. 하지만 지금은 이 조건을 어떻게 수행해야 할지 걱정하지 않아도 된다. 다만 이 고루틴이 프로세스가 끝날 때까지 종료되지 않는다는 것만 알아둬라.
2. 여기세 우리가 생성할 고루틴의 수를 정의한다. 고루틴 하나의 크기에 점근적으로 접근하기 위해 **대수의 법칙**(law of large number)을 사용할 것이다.
3. 여기서는 고루틴들을 생성하기 전에 소비된 메모리 양을 측정한다.
4. 그리고 여기서 고루틴들을 생성한 후에 소비된 메모리의 양을 측정한다.

결과는 이러하다.

``` go
PS C:\Users\kgm09\goproject\src\Concurrency> go run main.go
8.867kb
PS C:\Users\kgm09\goproject\src\Concurrency> go run main.go
8.474kb
PS C:\Users\kgm09\goproject\src\Concurrency> go run main.go
8.500kb
```

이 예제들의 고루틴들은 아무것도 하지 않는 빈 고루틴이지만, 고루틴을 몇 개나 만들 수 있는지 암시한다. 표 3-1에서는 스왑 공간을 사용하지 않고 64비트 CPU에서 생성할 수 있는 고루틴의 수를 대략적으로 추정한다.

**표 3-1**: 주어진 메모리 내에서 가능한 고루틴이 대략적인 수에 대한 분석


| 메모리(GB) | 고루틴(#/100000) | 자릿수 |
| ------- | ------------- | --- |
| 2^0     | 3.718         | 3   |
| 2^1     | 7.436         | 3   |
| 2^2     | 14.873        | 6   |
| 2^3     | 29.746        | 6   |
| 2^4     | 59.492        | 6   |
| 2^5     | 118.983       | 6   |
| 2^6     | 237.967       | 6   |
| 2^7     | 475.934       | 6   |
| 2^8     | 951.867       | 6   |
| 2^9     | 1903.735      | 9   |

정말로 큰 수치다. 8기가의 Ram일 경우, 스와핑 없이 수백만개의 고루틴이 돌아간다. 물론  이는 컴퓨터에 실행 중이 다른 프로그램과 고루틴의 실제 내용을 무시한 것이다. 하지만 이 계산을 통해 고루틴이 얼마나 가벼운 것인지 알 수 있다.


우리를 기운 빠지게 하는 또 다른 요소는 컨텍스트 스위칭(context switching)이다. ==동시에 실행되는 프로세스를 호스팅하는 어떤 것이 다른 동시 프로세스를 실행하도록 전환하기 위해 자신의 상태를 저장해야 하는 경우를 의미한다.== 동시에 실행되는 프로세스가 너무 많으면 프로세스 사이의 컨텍스트 스위칭에 모든 CPU 시간을 소모하느라 실제 작업은 전혀 수행하지 못할 수 있다. OS 수준에서 스레드를 사용하면 이로 인해 상당히 많은 비용이 발생할 수 있다. OS 스레드는 레지스터 값이나 룩업 테이블, 메모리 맵 등을 저장해야만 시간이 됐을 때 현재 스레드로 다시 전환할 수 있다. 그런 다음 진입하는 스레드에 대해서도 동일한 정보를 로드해야 한다.


소프트웨어에서의 컨텍스트 스위치은 상대적으로 훨씬 저렴하다. 소프트웨어로 정의된 스케줄러 아래에서 런타임은 되돌아오기 위해 유지해야 할 항목, 유지 방식, 유지 시점을 보다 선별적으로 고를 수 있다. 내 휴대용 컴퓨터에서 OS 스레드와 고루틴 사이의 상대적인 컨텍스트 스위칭 성능을 비교해보자. 먼저, 리눅스의 내장 벤치마킹 제품군을 사용해 동일한 코어에서 두 스레드 간에 메시지를 보내는 데 걸리는 시간을 측정한다.

(여기서는 실험을 할 수 없기 때문에 직접 타이핑하겠다.)

`taskset -c 0 perf bench sched pipe -T`

다음과 같은 결과가 나온다.

```
# Running 'sched/pipe' benchmark('sched/pipe' 벤치마크 실행):
# Excuted 1000000 pipe operations between two threads(두 스레드 사이에서 1000000개의 파이프 연산 실행)

	Total time: 2.935 [sec]
	340624 ops/sec

```

이 벤치마크는 사실 하나의 스레드에서 메시지를 보내고 받는 데 걸리는 시간을 측정하므로, 이 결과를 2로 나눌 것이다. 그러면 컨텍스트 스위치당 $1.467 \mu s$가 나온다. 그렇게 나쁘지는 않지만, 고루틴 간의 컨텍스트 전환을 조사할 때까지 일단 판단을 보류한다. 우선 Go를 사용해서 유사한 벤치마크를 구성한다.

다음 예제에서는 두 개의 고루틴을 생성하고 둘 사이에서 메시지를 전송한다.

``` go
package test
import (
    "sync"
    "testing"
)

  

func BenchmarkContextSwitch(b *testing.B) {

    var wg sync.WaitGroup
    begin := make(chan struct{})
    c := make(chan struct{})

  
    var token struct{}
    sender := func() {
        defer wg.Done()
        <-begin // #1
        for i := 0; i < b.N; i++ {
            c <- token // #2
        }
    }

  

    receiver := func() {
        defer wg.Done()
        <-begin // #1
        for i := 0; i < b.N; i++ {
            <-c // #3
        }
    }

    wg.Add(2)
    go sender()
    go receiver()
    b.StartTimer() // #4
    close(begin)   // #5
    wg.Wait()
}
```

1. 시작한다고 말해주기 전까지는 여기서 대기한다. 고루틴을 설정하고 시작하는 비용이 컨텍스트 스위칭을 측정하는데 영향을 미치는 것을 원치 않기 때문이다.
2. 여기서는 수신측 고루틴에게 메시지를 보낸다. struct{}{}는 **빈 구조체**라고 부르며, 메모리를 사용하지 않는다. 그러므로 메시지를 보내는데 걸리는 시간만 측정하게 된다.
3. 여기서는 메시지를 수신하겠지만, 메시지로 아무런 작업도 하지 않는다.
4. 여기서 성능 타이머를 시작시킨다.
5. 여기서 두 고루틴에게 시작하라고 한다.


리눅스의 벤치마크와 유사하게 만들기 위해 하나의 CPU 만 사용하도록 지정하고 벤치마크를 실행한다. 결과를 살펴보자.

``` shell
PS C:\Users\kgm09\goproject\src\test> go test -bench=. -cpu=1 \ goproject/src/fig-ctx-switch_test.go
```

``` shell
BenchmarkContextSwitch-6     3966256           311.5 ns/op         0 B/op          0 allocs/op

PASS

ok      goproject/src/test  1.801s
```

컨텍스트 스위치당 311.5로 이는 OS 컨텍스트 스위치보다 90% 더 빠르다. 고루틴이 얼마나 많아야 과도한 컨텍스트 스위칭이 발생하는지에 대해서는 어떤 주장도 어렵지만, 이 상한이 고루틴 사용에 있어 어떠한 장애물도 되지 않을 것이라고는 자신 있게 말할 수 있다.

지금까지 고루틴을 시작하는 방법과 그들이 작동하는 방법에 대해 약간은 이해했을 것이다. 또한 문제 공간에서 적절하다고 생각되면 얼마든지 안전하게 고루틴을 만들 수 있다는 점도 배웠다. 49페이지의 "[[코드 모델링, 순차적인 프로세스 간의 통신#동시성과 병렬성의 차이]]"에서 설명했듯이, ==문제 공간이 암달의 법칙에 의해 하나의 동시 실행 세그먼트로 제한되지만 않는다면 더 많은 고루틴을 만들수록 프로그램은 여러 개의 프로세서로 조정될 것이다.== 고루틴의 생성에는 큰 비용이 들지 않기 때문에, ==이것이 성능 문제의 근본 원인으로 입증된 경우에만 그 비용을 논의해야 한다.==



---
### sync 패키지

sync 패키지에는 저수준의 메모리 접근 동기화에 가장 유용한 동시성 기본 요소들이 포함되어 있다. 주로 메모리 접근 동기화를 통해 동시성을 처리하는 언어로 작업해본 적이 있다면 이러한 유형에 이미 익숙할 것이다. Go와 이들 언어의 차이점은, Go가 메모리 접근 동기화 기본 요소 위에 함께 동작할 수 있는 확장 세트를 제공하기 위해 새로운 동시성 기본 요소 세트를 구축했다는 점이다.

[[코드 모델링, 순차적인 프로세스 간의 통신#Go의 동시성에 대한 철학]]에서 설명했듯이, 이런 연산들은 모두 자기만의 쓰임이 있으며, 주로 구조체와 같은 작은 범위에서 쓰인다. 메모리 접근 동기화를 사용할 적절한 시점이 어디인지 결정하는 것은 개개인 에게 달렸다.

그렇기는 하지만, sync 패키지가 제공하는 다양한 기본 요소들을 살펴보자.


##### waitGroup

waitGroup은 동시에 수행된 연산의 결과를 신경쓰지 않거나, 결과를 수집할 다른 방법이 있는 경우 동시에 수행될 연산 집합을 기다릴 때 유용하다. 둘 중 어느 조건도 중족되지 않는다면 대신 채널과 select문을 사용하는 것이 좋다. WaitGroup은 매우 유용하다. 다음은 WaitGroup을 사용해 고루틴들이 완료되기를 기다리는 기본 예제이다.



``` go

package main

import (
    "fmt"
    "sync"
    "time"
)
func main() {
    var wg sync.WaitGroup

  

    wg.Add(1) // #1

    go func() {
        defer wg.Done() // #2
        fmt.Println("1st goroutine sleeping...")
        time.Sleep(1)
    }()

  

    wg.Add(1) // #1

    go func() {
        defer wg.Done() // #2
        fmt.Println("2nd goroutine sleeping...")
    }()

    wg.Wait() // #3
   fmt.Println("All groutine complete.")
}

```

1. 여기서는 1을 인자로 Add를 호출해 하나의 고루틴이 시작된다는 것을 나타낸다.
2. 여기에서는 고루틴의 클로저를 종료하기 전에 WaitGroup에게 종료한다고 알려주기 위해, defer 키워드를 사용해 Done을 호출한다.
3. 여기서 Wait를 호출하는데, 이 호출로 인해 main 고루틴은 다른 모든 고루틴이 자신들이 종료되었다고 알릴 때까지 대기한다. 


``` shell
PS C:\Users\kgm09\goproject\src\Concurrency> go run main.go
2nd goroutine sleeping...
1st goroutine sleeping...
All groutine complete.
```

WaitGroup을 동시에 실행해도 안전한(concurrent-safe)카운터라고 생각해도 된다. Add 호출은 전달된 정수만큼 카운터를 증가시키고, Done은 카운터를 1만큼 감소시킨다. Wait를 호출하면 카운터가 0이 될 때까지 대기한다.

Add에 대한 호출의 고루틴의 외부에서 수행하는 것이 추적에 도움이 된다는 점에 주목하자. 이렇게 하지 않았다면 아마도 레이스 컨디션이 일어났을 것이다. [[Go의 동시성 구성 요소#고루틴]]에서 언급했듯이, 언제 고루틴이 스케줄링될지 확신할 수 없다는 점을 기억해야 한다. 고루틴 중 하나가 시작되기도 전에 Wait 호출에 도달할 수 있다.
Add에 대한 호출이 일어나지 않을 수도 있다. 따라서 Wait를 호출하더라도 전혀 대기하지 않고 바로 리턴될 수 있었다.

추적을 돕기 위해 Add에 대한 호출은 고루틴에 가능한 한 가깝게 짝지어 두는 것이 관례이다. 그러나 때때로 여러 개의 고루틴을 한꺼번에 추적하기 위해 Add를 호출하는 경우도 보게 될 것이다. 보통은 반복문 앞에서 이런 작업을 한다.

```shell
PS C:\Users\kgm09\goproject\src\Concurrency> go run main.go
Hello from 3! 
Hello from 2!
Hello from 1!
Hello from 5!
Hello from 4!
```


##### Mutex와 RWMutex

이미 메모리 접근 동기화를 통해 동시성을 처리하는 언어에 익숙하다면, 바로 Mutex를 배워보자. 만약 그렇지 않다고 해도 Mutex는 이해하기 쉽기 때문에 전혀 걱정할 필요가 없다.

Mutex는 **"상호 배제"**(mutual exclusion)의 약자로, 프로그램의 임계 영역을 보호하는 방법이다. 1장의 내용을 떠올려보자. 임계 영역이란, 공유 리소스에 독점적으로 접근해야 하는 프로그램 영역을 말한다. Mutex는 이러한 공유 리소스에 대해 동시에 실행해도 안전한 방식의 배타적 접근을 나타내는 방법을 제공한다.

Go의 방식을 빌려서 말하자면, 채널은 통신을 위해 메모리를 공유하는 반면, Mutex는 개발자가 메모리에 대한 접근을 동기화하기 위해 따라야 하는 규칙을 만들어 메모리에 공유한다. Mutex를 사용해 메모리에 대한 접근을 보호하는 방식으로 이 메모리에 대한 접근을 조정해야 할 책임은 당신에게 있다. 다음은 공통된 값을 증가 및 감소시키려는 두 개의 고루틴에 대한 간단한 예시로, Mutex를 사용해 접근을 동기화한다.


``` go
package main

  

import (
    "fmt"
    "sync"
)

  

func main() {
    var count int
    var lock sync.Mutex

  

    increment := func() {
        lock.Lock()             // #1
        defer lock.Unlock()     // #2
        count++
        fmt.Printf("Incrementing: %d\n", count)
    }

  

    decrement := func() {
        lock.Lock()             // #1
        defer lock.Unlock()     // #2
        count--
        fmt.Printf("Decrementing: %d\n", count)
    }

  

    // 증가

    var arithmetic sync.WaitGroup

    for i := 0; i < 5; i++ {
        arithmetic.Add(1)
        go func() {
            defer arithmetic.Done()
            increment()
        }()
    }

  

    //감소

    for i := 0; i < 5; i++ {
        arithmetic.Add(1)
        go func() {
            defer arithmetic.Done()
            decrement()
        }()
    }

    arithmetic.Wait()
    fmt.Println("Arithmetic complete")
}
```

1. 여기서는 lock 이라는 Mutex에 의해 보호되는 임계 영역(count 변수)의 독점적인 사용을 요청한다.
2. 여기서는 lock이 보호하고 있는 임계 영역에 대한 작업이 끝났음을 표시한다.

다음과 같이 출력된다.

``` shell
PS C:\Users\kgm09\goproject\src\Concurrency> go run main.go
Incrementing: 1
Decrementing: 0
Incrementing: 1
Decrementing: 0
Decrementing: -1
Decrementing: -2
Decrementing: -3
Incrementing: -2
Incrementing: -1
Incrementing: 0
Arithmetic complete
```

항상 defer 구문 내에서 Unlock을 호출한다는 점을 알 수 있다. ==이것은 Mutex에서 매우 흔하게 사용되는 관용구로, 패닉(panic)이 발생하는 경우를 포함해 모든 경우에 Unlock이 확실하게 호출되는 것을 보장한다.== 그렇게 하지 않으면 프로그램이 데드락에 빠질 수 있다.

**임계 영역**은 이름에서도 알 수 있듯이 프로그램상의 병목 지점을 알 수 있다. 임계 영역에 진입하고 벗어나는 것은 다소 비용이 많이 들기 때문에, 사람들은 임계 영역에서 소비하는 시간을 최소화하려고 노력한다.

이를 위한 전략 중 하나는 임계 영역의 단면을 줄이는 것이다. 동시에 실행되는 여러 프로세스 간에 공유해야 하는 메모리가 있을 수 있지만, 이러한 프로세스들 모두가 이 메모리를 읽고 쓰지 않을 수도 있다. 이 경우 다른 유형의 뮤텍스인 sync.RWMutex를 이용할 수도 있다.

sync.RWMutex는 Mutex와 동일한 개념이다. 즉, 둘다 메모리에 대한 접근을 보호한다. 그러나 RWMutex는 조금 더 메모리를 제어할 수 있게 해준다. 예를 들어 읽기 잠금을 요청할 수 있지만,== 다른 프로세스가 쓰기 잠금을 가지고 있지 않는 경우에만 접근 권한이 부여된다.== 즉, 아무도 쓰기 잠금을 보유하고 있지 않다면 몇 개의 프로세스든 읽기 잠금을 보유할 수 있다. 다음은 **코드가 생성하는 수많은 소비자(consumer)** 보다 **덜 활동적인 생산자(producer)** 를 보여주는 예이다.


``` go
package main

import (
    "fmt"
    "math"
    "os"
    "sync"
    "text/tabwriter"
    "time"

)

func main() {
	    producer := func(wg *sync.WaitGroup, l sync.Locker) { // #1
        defer wg.Done()
        for i := 5; i > 0; i-- {
            l.Lock()
            l.Unlock()
            time.Sleep(1) // #2
        }
    }

  

    observer := func(wg *sync.WaitGroup, l sync.Locker) {
        defer wg.Done()
        l.Lock()
        defer l.Unlock()
    }

  

    test := func(count int, mutex, rwMutex sync.Locker) time.Duration {
        var wg sync.WaitGroup
              wg.Add(count + 1)
        beginTesttime := time.Now()
      go producer(&wg, mutex)


        for i := count; i > 0; i-- {
            go observer(&wg, rwMutex)
        }

        wg.Wait()
        return time.Since(beginTesttime)
    }

  

    tw := tabwriter.NewWriter(os.Stdout, 0, 1, 2, ' ', 0)
    defer tw.Flush()

  
    var m sync.RWMutex
    fmt.Fprintf(tw, "Readers\tRWMutex\tMutex\n")

  

    for i := 0; i < 20; i++ {
        count := int(math.Pow(2, float64(i)))
        fmt.Fprintf(tw, "%d\t%v\t%v\n", count, test(count, &m, m.RLocker()), test(count, &m, &m))

    }

  

}
```

1. producer함수의 두 번째 매개 변수는 sync.Locker 타입이다. 이 인터페이스는 Lock과 Unlock이라는 두 개의 메서드가 있는다, Mutex와 RWMutex 타입은 이 인터페이스를 충족시킨다.
2. 여기서는 producer를 1초 동안 대기해서 observer 고루틴보다 덜 활동적이도록 만든다.


출력
``` shell
PS C:\Users\kgm09\goproject\src\Concurrency> go run main.go
Readers  RWMutex     Mutex
1        40.5361ms   76.295ms
2        76.1164ms   77.3792ms
4        61.9825ms   76.6218ms
8        62.2634ms   62.1017ms
16       61.8006ms   63.4527ms
32       60.8468ms   60.9876ms
64       61.6835ms   62.6352ms
128      62.0121ms   47.4226ms
256      61.4324ms   78.3364ms
512      77.4268ms   77.8716ms
1024     46.641ms    76.1353ms
2048     63.2946ms   75.7662ms
4096     46.4365ms   62.2298ms
8192     47.0646ms   61.3077ms
16384    46.2174ms   5.24ms
32768    10.2803ms   10.8539ms
65536    20.2059ms   19.9654ms
131072   41.9342ms   40.1455ms
262144   78.8371ms   83.4278ms
524288   166.4314ms  164.4805ms

```

이 예제의 경우, 실제로 읽는 쪽이 213명 정도는 돼야 임계 영역을 단면을 줄인 효과가 있다는 점을 알 수 있다. 이것은 임계 영역이 어떤 작업을 하느냐에 따라 달라질 수 있지만, 논리적으로 합당하다면 Mutex 대신 RWMutex를 사용하는 것이 좋다.



##### Cond

Cond 타입에 대한 주석은 그 목적을 정말로 잘 설명하고 있다.

// ... 고루틴들이 대기하거나, 어떤 이벤트의 발생을 알리는 집결 지점(rendezvous point)


이 정의에서 "이벤트"란, 두 개 이상의 고루틴 사이에서, ==어떤 사실이 발생했다는 사실 외에는 아무런 정보를 전달하지 않는 임의의 신호를 말한다.== 대개는 하나의 고루틴에서 실행을 계속하기 전에 이러한 신호들 중 하나를 기다리고 싶을 것이다. Cond 타입 없이 이 작업을 수행하는 방법을 살펴보자면, 단순한 접근 방법 중 하나는 무한루프를 사용하는 것이다.

``` go

for condition () == false{

}
```

그러나 이것은 한 개 코어의 모든 사이클을 소모한다. 이를 개선하기 위해 time.Sleep을 소개한다.

``` go
for conditionTrue()== false{
time.Sleep(1*time.Millisecond)
}

```

이 방법이 조금 더 낫지만 여전히 비효율적이며, 얼마나 오랫동안 슬립(Sleep)할지 계산해야 한다. 너무 길면 성능이 부자연스럽게 저하된다. 너무 짧으면 불필요하게 CPU를 많이 소비하게 된다. ==고루틴이 신호를 받을 때까지 슬립하고 자신의 상태를 확인할 수 있는 효율적인 방법이 있다면 더 좋을 것이다.== 이것이 정확히 Cond 타입이 해주는 일이다. Cond를 사용하면 앞의 코드를 다음과 같이 작성할 수 있다.


``` go

	c:=sync.NewCond(&sync.Mutex{}) // #1
c.L.Lock() // #2
for conditionTrue()==false{
c.Wait() // #3
}
c.L.Unlock() // #4

```

1. 여기서 새로운 Cond를 초기화한다. NewCond 함수는 sync.Locker 인터페이스를 만족하는 타입을 인수로 받는다. 이 때문에 Cond 타입은 동시에 실행해도 안전한 방식으로 손쉽게 다른 고루틴들과의 조정이 가능하다.
2. 여기서는 Locker를 이 상태로 고정시킨다. Wait가 호출되면, Wait 호출에 진입할 때 자동적으로 Locker의 Unlock을 호출하기 때문에 이 작업이 필요하다.
3. 여기서는 조건(condition)이 충족됐다는 알림을 기다린다. 이것은 대기하는 호출로서, 해당 고루틴은 일시 중지된다.
4. 여기서는 이 조건에 대한 Locker의 잠금을 해제한다. Wait 호출을 빠져나오면서 이 조건에 대한 Locker의 Lock을 호출하기 때문에 이 작업이 필요하다.


이 방법은 훨씬 더 효율적이다. Wait에 대한 호출은 단지 멈춰서 대기하는 것이 아니라, ==현재 고루틴을 일시 중단해 다른 고루틴들이 OS스레드에서 실행될 수 있도록 한다.== ==Wait을 호출하면 몇몇 다른 작업도 이루어진다. 진입할 때 Cond 변수의 Locker에서 Unlock이 호출되고, Wait가 종료되면 Cond 변수의 Locker에서 Lock이 호출된다.==

여기에는 익숙해지는데 시간이 필요하다. 실제로 이것은 이 메서드의 숨겨진 부작용이다. 조건이 발생할 때까지 기다리면 계속 이 lock을 가지고 있는 것처럼 보이지만 그렇지 않다. 코드를 훑어볼 때 이 패턴을 주의 깊게 보아야 한다. 

이 예제를 확장해서 **신호를 기다리는 고루틴**과 **신호를 기다리를 고루틴**이라는 방정식의 양면을 모두 살펴보자. 길이가 2로 고정된 큐(queue)와 큐에 넣을 10개의 항목이 있다고 가정해보자. 여유가 생기면 즉시 항목을 큐에 넣기를 원하므로, 큐에 여유가 있는 경우 즉시 알림을 받고자 한다. 이러한 조정을 관리하기 위해 Cond를 사용해보자.

``` go
package main

  

import (
    "fmt"
    "sync"
    "time"
)

  

func main() {
    c := sync.NewCond(&sync.Mutex{}) // #1
    queue := make([]interface{}, 0, 10)// #2

  
    removeFromQueue := func(delay time.Duration) {
        time.Sleep(delay)
        c.L.Lock() // #8

        queue = queue[1:] // #9
        fmt.Println("Removed from queue")

        c.L.Unlock() // #10
        c.Signal() // #11

    }

  

    for i := 0; i < 10; i++ {
        c.L.Lock() // #3
        for len(queue) == 2 { // #4
            c.Wait() // #5
        }

        fmt.Println("Adding to queue")
        queue = append(queue, struct{}{})
        go removeFromQueue(1 * time.Second) // #6
        c.L.Unlock() // #7
    }

}
```
1. 먼저 표준 sync.Mutex를 Locker로 사용해 조건을 생성한다.
2. 다음으로 길이가 0인 슬라이스를 만든다. 최종적으로 10개의 항목을 추가할 것이므로, 10개를 저장할 수 있도록 인스턴스화한다.
3. 조건의 Locker에서 Lock를 호출해 조건의 임계 영역으로 진입한다.
4. 여기서는 루프 내부에서 큐의 길이를 확인한다. 조건에 대한 신호는 중요하다. 예상한 것이 발생했음을 알리는 것이 아니라, 뭔가 발생했가는 것을 알려주기 위해서이다.
5. Wait를 호출하면 조건에 대한 신호가 전송될 때까지 main 고루틴은 일시 중단된다.
6. 여기서 1초 후에 항목의 큐에서 꺼내는 새로운 고루틴을 생성한다.
7. 항목을 대기열에 성공적으로 추가했으므로 조건의 임계영역에 벗어난다.
8. 다시 조건의 임계 영역에 들어왔기 때문에 조건과 관련된 데이터를 수정할 수 있다.
9. 여기에서는 슬라이스의 헤드를 두 번째 항목에 재할당해, 항목이 큐에서 제거되는 것을 시뮬레이션한다.
10. 항목을 성공적으로 큐에서 제거했기 때문에 조건에 임계 영역을 벗어난다.
11. 조건을 기다리는 고루틴에게 뭔가 발생했음을 알린다.

```shell
PS C:\Users\kgm09\goproject\src\Concurrency> go run main.go
Adding to queue
Adding to queue
Removed from queue
Adding to queue
Removed from queue
Adding to queue
Removed from queue
Adding to queue
Removed from queue
Adding to queue
Removed from queue
Adding to queue
Removed from queue
Adding to queue
Removed from queue
Adding to queue
Removed from queue
Adding to queue
```


출력 결과에서 볼 수 있듯이, 이 프로그램은 성공적으로 10개의 항목을 큐에 추가했으며, 마지막 두 개의 항목을 큐에서 꺼내기 전에 종료된다. 또한 하나 항목을 대기열에 추가하기 전에는 항상 적어도 하나의 항목이 대기열에서 제외될 때까지 대기한다.

이 예제에서는 Signal이라는 새로운 메서드도 등장한다. ==Signal은 Cond 타입이 Wiat 호출에서 멈춰서 대기하는 고루틴들에게 조건이 발생하였음을 알라는 두 가지 메서드 중 하나이다.== 다른 하나는 Broadcast 라고 하는 메서드이다. 내부적으로 런타임은 신호가 오기를 기다리는 고루틴의 FIFO 목록을 유지한다.

[^1]: fork-join 모델은 동시성이 수행되는 방법에 대한 논리적인 모델이다. 이 모델은 fork와 wait를 호출하는 C 프로그램을 논리적인 수준에서만 설명해준다. fork-join 모델은 메모리가 관리되는 방식에 대해서는 아무것도 말해주지 않는다.