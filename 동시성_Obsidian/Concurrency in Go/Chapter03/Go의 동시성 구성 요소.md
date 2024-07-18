
3장에서는 Go의 동시성을 지원하는 풍부하고 정교한 기능에 대해 설명한다. 3장을 배우고 나면 활용할 수 있는 문법과 함수, 패키지로는 무엇이 있는지 파악할 수 있다. 또한 각 기능에 대해서도 이해할 수 있다. 


---

### 고루틴

^9ccee2

고루틴은 Go 프로그램을 구성하는 가장 기본적인 단위 중 하나다. 실제 모든 Go 프로그램에서는 적어도 하나의 고루틴이다. ==프로세스가 시작될 때 자동으로 생성되고 시작되는 main **고루틴**이 바로 그것이다.== 거의 모든 프로그램에서 문제를 해결할 때 언젠가는 고루틴을 사용하게 될 것이다. 그럼 고루틴이 무엇인가?


간단히 말하면 **고루틴**은 ==다른 코드와 함께 동시에 실행되는 함수==이다. 그렇다고 고루틴이 반드시 병렬로 실행되는 것은 아니다! 함수 앞에 go 키워드를 보면 간단히 시작할 수 있다.

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

일부 다른 언어는 비슷한 동시성 기본 요소가 있기는 하지만, 고루틴은 Go에만 존재한다. 고루틴은 OS 스레드가 아니다. 언어의 런타임에 의해 관리되는 스레드인 그린 스레드(green thread)도 아니다. ==고루틴은 **코루틴**(coroutine)이라 불리는 더 높은 수준의 추상화이다. 코루틴은 단순히 동시에 실행되는 서브루틴(함수, 클로저, 또는 Go의 메서드)으로서, **비점적**(nonpreemptive), 다시 말해 인터럽트할 수 없다.== 대신 코루틴은 잠시 중단(suspend)하거나 재진입(reentry)할 수 있는 여러 개의 지점을 가지고 있다.


고루틴을 Go만의 고유한 특징으로 만드는 것은 바로 Go 런타임과의 긴밀한 통합이다. ==고루틴은 자신의 일시 중단 시점이나 재진입 지점을 정의하지 않는다.== Go의 런타임은 고루틴의 실행 시 동작을 관찰해, 고루틴이 멈춰서 대기(block) 중일 때 자동으로 일시 중단시키고, 대기가 끝나면 다시 시작시킨다. ==Go 런타임이 이런 식으로 고루틴을 선점 가능하게 해 주기는 하지만, 고루틴이 멈춰 있는 지점에서만 선점 가능하다.== 이게 바로 런타임과 고루틴 로직 사이의 우아한 파트너십이다. 따라서 고루틴은 코루틴의 특별한 클래스로 간주할 수 있다.


코루틴은 동시에 실행되는 구조물이라고 여겨지지만, 그렇다고 해서 동시성이 코루틴의 속성인 것은 아니며, 고루틴은 코루틴의 일종이므로 고루틴 역시 그렇다. 즉, 누군가가 여러 코루틴이 동시에 주관(host,호스팅)하면서 각 코루틴이 실행될 수 있는 기회를 제공해야 한다. 그렇지 않으면 코루틴은 동시에 실행되지 않을 것이다! 그렇다고 이 말이 코루틴은 절대적으로 병렬적이라고 암시하는 것은 아니라는 점에 유의하자. 병렬로 처리된다는 환상을 심어주기 위해 순차적으로 실해되는 여러 개의 코루틴을 생성할 가능성이 있으며, 실제로 Go에서는 항상 이러한 일이 발생한다.

고루틴을 호스팅하는 Go의 매커니즘은 **M:N스캐줄러**를 구현한 것으로, M개의 그린 스레드를 N개의 OS스레드에 매핑한다는 의미이다. 그런 다음 고루틴은 스레드에 스케줄링 된다. 사용 가능한 그린 스레드보다 더 많은 고루틴이 있을 경우, 스케줄러는 사용 가능한 스레드로 고루틴을 분배하고 이 고루틴들이 대기 상태가 되면 다른 고루틴이 실행될 수 있도록 한다. 이 모든 것이 어떻게 작동하는지에 대해서는 6장에서 논의할 것이며, 3장에서는 Go가 동시성을 어떻게 모델링하는지 설명할 것이다.


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


그러나 이 예제에서는 한 가지 문제가 있다. ==작성된 대로 sayHello 함수가 실행될지 아닐지 전혀 알 수 없다는 점이다.== 이 고루틴은 Go의 런타임을 통해 실행되도록 생성 및 스케줄링되지만, 실제로는 main 고루틴이 종료되지 전에 실행될 기회를 얻지 못할 수도 있다.

실제로 이 간단한 예제를 실행할 때, 단순함을 위해 main 함수의 나머지 부분을 생략했기 때문에 sayHello에 대한 호출을 호스팅하는 고루틴이 시작되기도 전에 프로그램의 실행이 끝날 것이다. 결과적으로 표준 출력(stdout)에는 "hello"라는 단어가 출력되지 않는다. ==고루틴을 생성해 후에 time.Sleep을 둘 수도 있지만, 이는 실제로 합류 지점을 만드는 것이 아니라 단지 레이스 컨디션을 일으킬 뿐이라는 것을 기억하자.== [[동시성 소개]]인 1장을 보면, 종료되기 전에 실행될 확률이 늘어나기는 하지만 실행을 보장할 수 없다. ==**합류 지점**은 프로그램의 정확성을 보증하고 레이스 컨디션을 제거하는 요소이다.==

합류 지점을 생성하려면 main 고루틴과 sayHello 고루틴을 동기화해야 한다. 이 작업은 여러 가지 방법으로 수행할 수 있지만, 여기서는 78페이지의 "[[Go의 동시성 구성 요소#sync 패키지|sync 페이지]]"에서 설명할 내용인 sync.WaitGroup을 사용한다. 이 예제가 어떻게 두 개의 고루틴 사이에 조인 지점을 만드는지 이해하는 것은 현재로서는 중요하지 않다. 다음은 앞서 언급한 문제점을 수정한 예시이다.


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

이 예제는 ==sayHello 함수를 호스팅하는 고루틴이 종료될 때까지 확정적으로 main 고루틴을 멈춰 놓는다.== sync.WaitGroup 의 동작 방식에 대해서는 78페이지의 "sync 패키지"에서 배우겠지만, 여기서는 이 예제가 정확하게 동작하도록 만들기 위해 sync.WaitGroup 사용해 합류 지점을 만든다.

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


일반적으로 현재 컴퓨터에서는 ==고루틴이 실행되기도 전에 루프가 종료되므로 salutation은 문자열의 슬라이스의 마지막 값인 "good day"에 대한 참조를 저장하고 있는 힙으로 옮겨지게 되고==, 이에 따라 보통은 "good day"가 세 번 출력된다. 이 반복문을 작성하는 올바른 방법은, 클로저 내부로 salutation의 사본을 전달해 고루틴이 실행될 때, 반복문에서 해당 반복 회차(iteration)에 고루틴으로 전달된 데이터를 가지고 동작하게 하는 것이다.

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

이 고루틴은 프로세스가 종료될 때까지 대기하고 있을 것이다. 이 문제를 해결하는 방법에 대해서는 4장의 "[[Go의 동시성 패턴#고루틴 누수 방지|고루틴 누수 방지]]"에서 설명한다. 다음 예제에서는 절대 종료되지 않는 고루틴을 실제로 고루틴의 크기를 측정하는데 사용한다.

이 예제는 고루틴이 가비지 컬렉션되지 않는다는 사실과 런타임이 스스로를 검사할 수 있는 기능을 조합해, **고루틴 생성 전후에 할당된 메모리의 양을 측정**한다.


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


소프트웨어에서의 컨텍스트 스위치는는 상대적으로 훨씬 저렴하다. 소프트웨어로 정의된 스케줄러 아래에서 런타임은 되돌아오기 위해 유지해야 할 항목, 유지 방식, 유지 시점을 보다 선별적으로 고를 수 있다. 내 휴대용 컴퓨터에서 OS 스레드와 고루틴 사이의 상대적인 컨텍스트 스위칭 성능을 비교해보자. 먼저, 리눅스의 내장 벤치마킹 제품군을 사용해 동일한 코어에서 두 스레드 간에 메시지를 보내는 데 걸리는 시간을 측정한다.

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

waitGroup은 동시에 수행된 연산의 결과를 신경쓰지 않거나, 결과를 수집할 다른 방법이 있는 경우 동시에 수행될 연산 집합을 기다릴 때 유용하다. 둘 중 어느 조건도 충충족되지 않는다면 대신 채널과 select문을 사용하는 것이 좋다. WaitGroup은 매우 유용하다. 다음은 WaitGroup을 사용해 고루틴들이 완료되기를 기다리는 기본 예제이다.



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

Mutex는 **"상호 배제"**(mutual exclusion)의 약자로, ==프로그램의 임계 영역을 보호하는 방법==이다. 1장의 내용을 떠올려보자. 임계 영역이란, 공유 리소스에 독점적으로 접근해야 하는 프로그램 영역을 말한다. Mutex는 이러한 공유 리소스에 대해 동시에 실행해도 안전한 방식의 배타적 접근을 나타내는 방법을 제공한다.

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

sync.RWMutex는 Mutex와 동일한 개념이다. 즉, 둘다 메모리에 대한 접근을 보호한다. 그러나 RWMutex는 조금 더 메모리를 제어할 수 있게 해준다. 예를 들어 읽기 잠금을 요청할 수 있지만, == 다른 프로세스가 쓰기 잠금을 가지고 있지 않는 경우에만 접근 권한이 부여된다.== 즉, 아무도 쓰기 잠금을 보유하고 있지 않다면 몇 개의 프로세스든 읽기 잠금을 보유할 수 있다. 다음은 **코드가 생성하는 수많은 소비자(consumer)** 보다 **덜 활동적인 생산자(producer)** 를 보여주는 예이다.


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

이 예제를 확장해서 **신호를 기다리는 고루틴**과 **신호를 보내는 고루틴**이라는 방정식의 양면을 모두 살펴보자. 길이가 2로 고정된 큐(queue)와 큐에 넣을 10개의 항목이 있다고 가정해보자. 여유가 생기면 즉시 항목을 큐에 넣기를 원하므로, 큐에 여유가 있는 경우 즉시 알림을 받고자 한다. 이러한 조정을 관리하기 위해 Cond를 사용해보자.

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

이 예제에서는 Signal이라는 새로운 메서드도 등장한다. ==Signal은 Cond 타입이 Wait 호출에서 멈춰서 대기하는 고루틴들에게 조건이 발생하였음을 알라는 두 가지 메서드 중 하나이다.== 다른 하나는 Broadcast 라고 하는 메서드이다. 내부적으로 런타임은 신호가 오기를 기다리는 고루틴의 FIFO 목록을 유지한다.

Signal 메서드는 가장 오래 기다린 고루틴을 찾아서 알려주는 반면, Broadcast가 Signal보다 더 흥미로운 이유가 있다. ==Broadcast는 여러 개의 고루틴들과 한 번에 통신할 수 있는 방법을 제공하기 때문이다.== 여러 채널들로 Signal을 쉽게 복제할 수 있지만 (채널 참고), Broadcast를 반복적으로 호출하는 동작을 복제하는 것은 더 어려울 수 있다. 또한 Cond 타입은 채널을 사용하는 것보다 훨씬 더 뛰어나다.

Broadcast를 사용하는 것이 어떤 느낌인지 알아보기 위해, 버튼이 있는 GUI 애플리케이션을 만들고 있다고 생각해보자. 버튼을 클릭했을 때 실행되는 임의의 수의 함수를 등록하고자 한다. ==Cond는 Broadcast 메서드를 사용해 등록된 모든 핸들러에 통지할 수 있기 때문에 이 상황에 완벽하게 부합한다.==

``` go

package main

  

import (

    "fmt"
    "sync"

)

  

type Button struct { // #1
   Clicked *sync.Cond
}

  

func main() {
    button := Button{Clicked: sync.NewCond(&sync.Mutex{})}

  

    subscribe := func(c *sync.Cond, fn func()) { // #2
        var goroutineRunning sync.WaitGroup
        goroutineRunning.Add(1)
     
go func() {
            goroutineRunning.Done()
            c.L.Lock()
            defer c.L.Unlock()
            c.Wait()
            fn()
        }()

        goroutineRunning.Wait()
    }

  

    var clickRegistered sync.WaitGroup // #3
    clickRegistered.Add(3)

  

   subscribe(button.Clicked, func() { // #4
        fmt.Println("Maximizing window.")
        clickRegistered.Done()
    })
  

    subscribe(button.Clicked, func() { // #5
        fmt.Println("Displaying annoying dialog box!")
        clickRegistered.Done()
    })

  

    subscribe(button.Clicked, func() { // #6
        fmt.Println("Mouse clicked.")
        clickRegistered.Done()
    })

  

    button.Clicked.Broadcast() // #7
    clickRegistered.Wait()

  

}
```

1. Clicked라는 조건을 가지고 있는 Button 타입을 정의한다.
2. 여기서는 조건의 신호들을 처리하는 함수를 등록할 수 있는 편의 함수를 정의한다. 각 핸들러의 자체 고루틴에서 실행되며, 고루틴이 실행 중이라는 것을 확인하기 전까지 subscribe 종료되지 않는다.
3. 마우스 버튼이 (눌렸다가) 올라왔을 대의 핸들러를 설정한다. 이는 결국 Clicked Cond에서 Broadcast를 호출해 모든 핸들러에게 마우스 버튼이 클릭됐음을 알린다.(보다 안정적인 구현을 위해서는 먼저 버튼이 눌렸는지 확인하면 된다.)
4. 여기서는 WaitGroup을 생성한다. 이는 stdout에 대한 쓰기가 발생하기 전에 프로그램이 종료되지 않도록 하기 위해서이다.
5. 여기서는 버튼을 클릭할 대 버튼의 윈도우를 최대화하는 것을 시뮬레이션하는 핸들러를 등록한다.
6. 여기서는 마우스는 클릭됐을 때 대화 상자를 표시하는 것을 시뮬레이션하는 핸들러를 등록한다.
7. 다음으로 사용자가 애플리케이션의 버튼을 클릭했다가 떼었을 때를 시뮬레이션한다.

이는 다음과 같이 출력될 것이다.

``` shell
PS C:\Users\kgm09\goproject\src> go run main.go
Displaying annoying dialog box!
Maximizing window.
Mouse clicked.
```


Clicked Cond 에서 Broadcast를 한 번 호출하면 세 개의 핸들러가 모두 실행된다는 것을 알 수 있다. clickRegistered WaitGroup을 위한 것이 아니라면 button.Clicked.Broadcast()을 여러 번 호출할 수 있으며, ==호출될 때마다 세 개의 핸들러가 모두 호출된다.== 이는 채널이 쉽게 할 수 없는 것으로, Cond 타입을 사용하는 주된 이유 중 하나이다.

sync 패키지의 다른 대부분의 것들과 마찬가지로, ==Cond는 좁은 범위의 (scope)에서 사용하거나 이를 캡슐화하는 다른 타입을 통해 더 넓은 범위로 노출시키는 것이 가장 좋다.==


### Once

``` go
package main

  

import (
    "fmt"
    "sync"
)

type Button struct { // #1
    Clicked *sync.Cond
}

  

func main() {

    var count int

    increment := func() {

        count++

    }

    var once sync.Once
    var increments sync.WaitGroup

    increments.Add(100)

    for i := 0; i < 100; i++ {
        go func() {
            defer increments.Done()
            once.Do(increment)
        }()
    }
  

    increments.Wait()
    fmt.Printf("Count is %d\n", count)
}
```

출력 결과가 Count is 100이라고 생각하기 쉽지만, 위 코드에서 sync.Once 변수를 선언하고, once의 Do 메서드로 increment 호출을 감싼 것을 봤을 것이다. 사실 이 코드는 다음과 같이 호출한다.

```
PS C:\Users\kgm09\goproject\src> go run main.go
Count is 1
```

이름에서 알 수 있듯이, sync.Once은 Do에 대해, 하나의 호출(심지어 서로 다른 고루틴들 사이에서도)만이 인수로 전달된 함수를 호출할 수 있도록 내부적으로 sync 패키지의 몇 가지 기본 요소를 사용하는 타입이다. ==이 결과는 전적으로 sync.Once의 Do 메서드에서 increment 호출을 감싸고(wrap) 있기 때문이다.==

함수를 정확하게 한 번만 호출하는 기능을 캡슐화하고 표준 패키지에 넣는 것이 이상해 보일 수도 있지만, 이 패턴이 필요한 경우가 의로 있다. 재미삼아 Go의 표준 라이브러리를 확인해 Go가 이 기본 요소를 얼마나 자주 사용하는지 확인해보자. 다음은 검색을 수행하는 grep 명령어이다.


```
grep -ir sync.Once $(go env GOROOT)/src |wc -l
```

실행하면 다음과 같이 출력된다.

70

sync.Once를 사용할 때 알아두어야 할 점이 몇 가지 더 있다. 다음 예제를 살펴보자. 어떤 결과가 출력할 것 같은가?

``` go
package main

  

import (
    "fmt"
    "sync"
)


func main() {
    var count int
    increment := func() {
        count++
    }

    decrement := func() {
        count--
    }

  

    var once sync.Once
    once.Do(increment)
    once.Do(decrement)

    fmt.Printf("Count: %d\n", count)

  

}
```


다음과 같이 출력한다.
``` go
PS C:\Users\kgm09\goproject\src> go run main.go
Count: 1
```


0이 아니라 1이 출력돼어서 놀랐는가? 그 이유는 sync.Once가 Do에 전달된 각 함수가 호출된 횟수가 아니라,==Do가 호출된 횟수만을 계산하기 때문이다.== 이로 인해 sync.Once의 복사본들은 자신들이 호출하게 되어 있는 함수들과 밀접하게 결합한다.

sync 패키지 내의 타입은 ==좁은 범위 내에서 사용할 때== 가장 잘 작동한다는 것을 다시 확인할 수 있다. 작은 어휘 블록이나 함수 내부의 모든 sync.Once 사용을 작은 함수로 감싸거나 sync.Once와 함수를 하나의 타입으로 감싸서 이러한 결합이 형식적으로 드러나게 할 것을 추천한다. 

이 예제는 어떤가? 어떤 일이 일어날 것 같은가?


``` go
package main

  

import (
    "sync"
)

  

func main() {

    var onceA, onceB sync.Once
    var initB func()

  

    initA := func() { onceB.Do(initB) }
    initB = func() { onceA.Do(initA) } // #1
    onceA.Do(initA) // #2

}
```

#1 이 호출은 `#2`의 호출이 리턴될 때가지 진행되지 않는다.

`#2`의 Do 호출이 종료될 때까지 `1#`의 Do 호출이 진행되지 않기 때문에, ==이 프로그램은 데드락에 빠지며 이는 데드락의 전형적인 예라고 할 수 있다.== 경우에 따라 마치 여러 차례 초기화되는 것을 예방하기 위해 sync.Once를 사용하고 있는 것처럼 보이기 때문에, 조금 직관적이지 않을 수 있지만, sync.Once가 보증하는 유일한 한 가지는 함수들이 한 번만 호출된다는 것이다. 때때로 이는 프로그램을 데드락 상태로 만들고 논리의 결함(이 예제에서는 순환 참조)을 드러내면서 이루어진다.



### Pool

Pool은 동시에 실행해도 안전한 객체 풀(object pool)패턴의 구현이다. 객체 풀 패턴에 대한 완전한 설명은 디자인 패턴에 대한 문서[^2]를 참고하는 것이 가장 좋다. 그러나 Pool이 sync 패키지 내에 존재하기 때문에, 이를 활용하면 좋은 이유에 대해 간략히 논의할 것이다.

높은 수준에서의 풀 패턴은 고정된 개수만큼의 사용할 것들, 즉 풀을 생성해두고 활용할 수 있게하는 방법이다. ==일반적으로 데이터베이스 연결과 같이 비용이 많이 드는 것의 생성을 제한해 고정된 수의 개체만 생성하도록 하지만==, 이러한 개체에 대한 접근을 요청하는 연산이 얼마나 될지는 쉽게 알 수 없다. Go의 sync.Pool 같은 경우, 이 데이터 타입은 여러 고루틴에서 안전하게 사용할 수 있다.

Pool의 기본 인터페이스는 Get 메서드이다. Get 메서드가 호출되면 먼저 호출자에게 리턴할 수 있는 인스턴스가 풀 내에 있는지 확인하고, 그렇지 않으면 새 인스턴스를 만들기 위해 New 멤버 변수를 호출한다. ==사용이 끝나면 호출자는 사용했던 인스턴스를 다른 프로세스가 사용할 수 있도록 풀에 다시 돌려주기 위해 Put을 호출한다.==


``` go
package main

  

import (
    "fmt"
    "sync"
)

  

func main() {

    myPool := &sync.Pool{
        New: func() any {
            fmt.Println("Creating new instance.")
            return struct{}{}
        },
    }

  

    myPool.Get()             // #1
    instance := myPool.Get() // #1
    myPool.Put(instance)     // #2
    myPool.Get()             // #3

}
```

1. 여기서는 풀의 Get을 호출한다. 인스턴스가 아직 초기화되지 않았기 때문에 이 호출은 정의된 New 함수를 호출한다.
2. 여기서는 이전에 조회했던 인스턴스를 다시 풀에 돌려놓는다. 이는 사용가능한 인스턴스의 수를 1로 증가시킨다.
3. 이 호출이 실행되면 이전에 할당됐다가 다시 풀에 넣는 인스턴스를 다시 사용한다. **New함수는 호출되지 않는다.**

다음에서 볼 수 있듯이, New 함수는 두 번만 호출된다.
``` shell
PS C:\Users\kgm09\goproject\src> go run main.go
Creating new instance. // 새로운 인스턴스 생성
Creating new instance. // 새로운 인스턴스 생성 중
```

그렇다면 왜 쓰는 만큼 객체를 인스턴스화하지 않고 풀을 사용하는가? Go에는 가비지 컬렉터가 있으므로 인스턴스화된 객체는 자동으로 정리된다. 요점이 무엇인가?

다음과 같은 예제는 생각해보자.
``` go
package main

import (
    "fmt"
   "sync"
)

  

func main() {

  
    var numCalcsCreated int

    calcPool := &sync.Pool{
        New: func() any {
            numCalcsCreated += 1
            mem := make([]byte, 1024)
            return &mem // #1
        },

    }

    // 4KB로 풀을 시작한다.

    calcPool.Put(calcPool.New())
    calcPool.Put(calcPool.New())
    calcPool.Put(calcPool.New())
    calcPool.Put(calcPool.New())

  

    const numWorkers = 1024 * 1024
    var wg sync.WaitGroup
  
    wg.Add(numWorkers)

    for i := numWorkers; i > 0; i-- {
        go func() {
            defer wg.Done()
            mem := calcPool.Get().(*[]byte) // #2
            defer calcPool.Put(mem)

            // 이 메모리에서 뭔가 흥미롭지만 빠른 작업이

            // 이루어진다고 가정하자.
        }()
    }

    wg.Wait()
    fmt.Printf("%d calculators were created.", numCalcsCreated)
}
```
1. byte 슬라이스들의 주소를 저장하고 있음을 유의하라.
2. 그리고 여기서는 타입이 당연히 byte 슬라이스에 대한 포인터라고 가정하고 있다.


이는 다음과 같이 출력된다.

``` go
PS C:\Users\kgm09\goproject\src> go run main.go
7 calculators were created.
```

결과가 언제나 확정적인 것은 아니나, sync.Pool 없이 이 예제를 실행했다면 최악의 경우 기가 바이트 단위의 메모리를 할당하려고 시도했을 수 있다. 하지만 출력에서 보듯이 겨우 4KB만 할당했다.

Pool은 가능한 빨리 실행해야 하는 작업을 위해 사전에 할당된 객체의 캐시를 준비해 두는 경우에도 유용하다. 이 경우 생성될 객체의 수를 제한해 호스트 시스템의 메모리를 보호하는 것이 아니라, ==사전 로딩을 통해 다른 객체에 대한 참조를 가져오는 데 걸리는 시간을 아껴서 고객의 시간을 보호한다.== 가능한 한 신속하게 요청에 응답하려고 하는, 높은 처리 성능의 네트워크 서버를 작성하는 경우에 이는 매우 일반적이다. 그런 시나리오를 살펴보자.

먼저, 서비스에 연결하는 일을 시뮬레이션하는 함수를 생성해보자. 이 연결에 시간이 오래 걸리도록 만들 것이다.

``` go
func connectToService() any {

    time.Sleep(1*time.Second)

    return struct{}{}

}
```

다음으로 모든 요청에 대해, 서비스에 대한 새로운 연결을 생성하는 경우에 성능이 어느 정도인지 살펴본다. ==우리는 네트워크 핸들러가 받아들이는 모든 연결에 대해, 서로 다른 서비스에 연결을 여는 네트워크 핸들러를 작성한다.== 성능 측정을 간단하게 만들기 위해 한 번에 하나의 연결만 가능하게 한다.

``` go
func startNetworkDaemon() *sync.WaitGroup {

    var wg sync.WaitGroup
    wg.Add(1)

  
    go func() {
        server, err := net.Listen("tcp", "localhost:8080")
        if err != nil {
            log.Fatalf("cannot listen: %v", err)
        }

        defer server.Close()
        wg.Done()

        for {
            conn, err := server.Accept()
            if err != nil {
                log.Printf("cannot accept connection: %v", err)
                continue
            }

            connectToService()
            fmt.Fprintln(conn, "")
            conn.Close()

        }

    }()
    return &wg
}
```

이제 이 성능을 측정해보자.

``` go
func init() {
    daemonStarted := startNetworkDaemon()
    daemonStarted.Wait()
}

  

func BenchmarkNetworkRequest(b *testing.B) {

    for i := 0; i < b.N; i++ {
        conn, err := net.Dial("tcp", "localhost:8080")
        if err != nil {
            b.Fatalf("cannot dial host: %v", err)
        }

  

        if _, err := io.ReadAll(conn); err != nil {
            b.Fatalf("cannot read: %v", err)
        }
        conn.Close()
    }
}
```


다음과 같이 출력될 것이다.

``` go
Running tool: C:\Users\kgm09\go\bin\go.exe test -benchmem -run=^$ -bench ^BenchmarkNetworkRequest$ src/file

  

goos: windows

goarch: amd64

pkg: src/file

cpu: AMD Ryzen 5 4500U with Radeon Graphics        

BenchmarkNetworkRequest-6              1    1013339000 ns/op        9584 B/op        115 allocs/op

PASS

ok      src/file    2.273s
```

위의 경우 합리적인 성능으로 보이지만, sync.Pool을 사용해 중요한 서비스에 대한 연결을 호스트함으로써 얼마나 개선할 수 있는지 알아보자.

``` go
  

func warmServiceConnCache() *sync.Pool {

    p := &sync.Pool{
        New: connectToService,
    }

    for i := 0; i < 10; i++ {
        p.Put(p.New())
    }
    return p

}

  

func startNetworkDaemon() *sync.WaitGroup {

    var wg sync.WaitGroup
    wg.Add(1)

    go func() {
        connPool := warmServiceConnCache()

  

        server, err := net.Listen("tcp", "localhost:8080")
        if err != nil {
            log.Fatalf("cannot listen: %v", err)
        }

        defer server.Close()
  
        for {
            conn, err := server.Accept()
            if err != nil {
                log.Printf("cannot accept connection: %v", err)
                continue
            }

            svcConn := connPool.Get()
            fmt.Fprintln(conn, "")

            connPool.Put(svcConn)
            conn.Close()
        }
    }()
    return &wg
}
```

다음과 같이 측정된다.

``` shell
2024/06/27 16:28:25 cannot listen: listen tcp 127.0.0.1:8080: bind: Only one usage of each socket address (protocol/network address/port) is normally permitted.

exit status 1

FAIL    src/file    11.037s

FAIL
(다만 지금 자꾸 실패가 되는데 원인을 모르겠다.)
```

자릿수가 3개나 차이가 날 정도로 빠르다. 생성 비용이 많이 드는 것으로 작업할 때 이 패턴을 이용하면 응답 시간이 크게 향상될 수 있으므로 알 수 있다.

그러나 Pool을 활용해야 하는지 여부를 결정할 때 조심해야 할 점이 있다. ==Pool을 사용하는 코드에서 서로 다른 객체가 필요한 경우, 처음부터 인스턴스화 하는 작업보다 Pool에서 가져온 것을 변환하는 작업이 더 오래 걸릴 수 있다.== 예를 들어, 프로그램이 임의의 가변 슬라이스를 필요로 한다면, Pool은 별로 도움이 되지 않을 것이다. 필요로 하는 길이의 슬라이스를 받을 확률이 낮기 때문이다.

따라서 Pool을 사용하여 작업할 때는 다음 사항을 기억하자.

- sync.Pool을 인스턴스화 할 때, 호출 시 스레드로부터 안전한 New 멤버 변수를 전달한다.
- Get에서 인스턴스를 받았을 때, 돌려받은 객체의 상태에 대한 가정을 해서는 안 된다.
- Pool에서 꺼낸 객체로 작업을 마치면 반드시 Put을 호출한다. 그렇게 하지 않으면 Pool은 아무런 소용이 없게 된다. 보통 이 작업은 defer로 이루어진다.
- 풀 내에 있는 객체들은 구조가 거의 균일해야 한다

---
# 채널

^f1c508

채널은 호어의 CSP에서 파생된 Go의 동기화 기본 요소 중의 하나다. 메모리 접근을 동기화하는 데 이것을 사용할 수도 있지만, 채널은 고루틴 간의 정보를 전달할 때 가장 적합하다. 60페이지의 "[[코드 모델링, 순차적인 프로세스 간의 통신#Go의 동시성에 대한 철학|Go의 동시성에 대한 철학]]"에서 논의했듯이, 채널은 여러 개를 함께 구성할 수 있기 때문에 어떠한 크기의 프로그램에서든 매우 유용하다. 지금부터는 채널을 소개한 다음, 다음 절인 "**select 구문**"에서 구성에 대해 알아본다. 채널은 강물처럼 정보 흐름을 위한 통로 역할을 한다.

값은 채널을 따라 흘러가 다음 하류 쪽에서 읽을 수 있다. 이런 이유로 나는 보통 채널 변수 이름을 "Stream"이라는 단어로 끝낸다. 채널을 사용할 때 값을 chan 변수에 전달한 다음, 프로그램의 다른 곳에서 채널을 읽는다. ==프로그램의 서로 다른 두 부분은 서로에 대해 알 필요가 없으며==, ==채널이 존재하는 메모리상의 동일한 위치에 대한 참조만 알면 된다.== 이 과정은 여기 저기에 채널에 대한 참조를 전달함으로써 진행된다.

채널을 생성하는 일은 매우 간단하다. 다음은 채널을 생성하는 구문을 채널의 선언과 인스턴스화로 나누어, 선언 및 인스턴스화를 모두 볼 수 있도록 한 예제이다. Go의 다른 값들과 마찬가지로 := 연산자를 통해 한 단계로 채널을 만들 수도 있지만, 채널을 선언해야 하는 경우가 자주 있을 것이므로 두 단계를 개별 단계로 나누는 편이 좋다.

``` go
var dataStream chan interface{ }   // #1

    dataStream=make(chan  interface{})  // #2
```

1. 여기서는 채널을 선언한다. 우리가 선언한 타입은 빈 인터페이스이므로, 이를 interface{} "타입"이라고 한다.
2. 여기서는 내장 make 함수를 사용해 채널을 인스턴스화 한다.

이 예제는 비어 있는 인터페이스를 사용했기 때문에 값을 쓸 수도 있고 읽을 수도 있는, dataStream 채널을 정의한다. 채널은 단방향의 데이터 흐름만 지원하도록 선언할 수도 있다. 즉, 정보 송신 또는 수신만 지원하는 채널을 정의할 수 있다. 이 절의 뒷부분에서 이것이 왜 중요한지 설명할 것이다.


단방향 채널을 선언하려면 <- 연산자만 포함시키면 된다. 읽기 가능한 채널을 선언하고 인스턴스화할 때는 모두 다음과 같이 왼쪽 편에 <- 연산자만 위치시키면 된다.

``` go
var dataStream <- chan interface{}
dataStream=make(<-chan interface{})
```

그리고 송신만 가능한 채널을 선언하고 생성하려면 다음과 같이 오른쪽에 <-연산자를 위치시키면 된다.

``` go
var dataStream chan <- interface{}
dataStream:=make(chan <-interface{})
```

단방향 채널의 인스턴스화는 흔하지 않지만, 함수의 매개 변수나 리턴 타입으로 단방향 채널이 유용하게 사용되는 경우는 자주 볼 수 있을 것이다. ==이는 필요할 때 Go가 양방향 채널을 묵시적으로 단방향 채널로 변환하기 때문이다.== 예제를 살펴보자.

``` go

var receiveChan <-chan interface{}
var sendChan chan <- interface{}
dataStream:=make(chan interface{})

// 유효한 구문
receiveChan=dataStream
sendChan=dataStream

```


채널의 타입이 지정되어 있음을 유의하자. 이 예제에서는 chan interface{} 변수를 만들었는데, 이는 어떤 종류의 데이터도 위치시킬 수 있다는 의미이지만, 더 엄격한 타입을 지정하여 전달할 수 있는 데이터의 타입을 제한할 수 있다. 다음은 정수를 위한 채널의 예제이다. 또한 이미 소개가 끝났기 때문에 채널을 인스턴스화하는 정석적인 방식을 보다 간단한 방식으로 전환할 것이다.

``` go

intStream:=make(chan int)

```

채널을 사용하려면 또 다시 <- 연산자를 사용해야 한다. 송신은 <- 연산자를 오른쪽에, 수신은 연산자를 왼쪽에 놓음으로서 수행된다. 채널에서의  데이터가 송수신되는 방향을 쉽게 이해하려면 화살표가 가리키는 쪽의 변수로 데이터가 흐른다는 점을 떠올리면 된다. 간단한 예제를 살펴보자.


``` go
package main

  

import (
    "fmt"
    "log"
    "net"
    "sync"
    "time"

)

  
  

func main() {

    stringStream:=make(chan string)

    go func() {
        stringStream<-"Hello channels"
    }()

    fmt.Println(<-stringStream)
}
```



채널 변수만 있으면 그곳으로 데이터를 전달하고 읽을 수 있다. 그러나 읽기 전용 채널에 값을 쓰려고 하거나 쓰기 전용 채널에서 값을 읽으려고 하면 에러가 발생한다. 다음 예제를 시도해보면, Go 의 컴파일러는 우리가 잘못된 작업을 하고 있다는 사실을 알려줄 것이다. 

``` go
package main

  

func main() {
    writeStream := make(chan<- interface{})
    readStream := make(<-chan interface{})

  

    <-writeStream
    readStream <- struct{}{}

}
```


``` shell
invalid operation: cannot receive from send-only channel writeStream (variable of type chan<- interface{})
invalid operation: cannot send to receive-only channel readStream (variable of type <-chan interface{})
```

위와 같은 에러 메시지는 동시성 기본 요소를 처리할 때조차도 타입 안정성이 가능하도록 해 주는 Go 타입 시스템의 일부다. 이 정의 뒷부분에서 배우겠지만 이는 추론하기 쉬우면서도, 구성 가능하고, 논리적인 프로그램을 작성하기 위해 API를 선언할 수 있는 강력한 방법이다.

3장의 앞부분에서 고루틴이 스케줄링됐다고 해서 프로세스가 종료되기 전에 고루틴이 실행될 것이라는 보장은 없다는 사실을 강조했다. 그러나 앞의 예제는 실행되지 않고 생략되는 코드 없이 완전하고 정확하다. 어째서 main 고루틴이 끝나기 전에 익명 고루틴이 완료되는지 궁금할 수 있다. 코드를 실행했을 때 운이 좋았던 것일까? 본론에서 벗어나기는 하지만 그 이유를 알아보도록 하겠다.

보통 Go의 채널은 멈춰서 기다리고 있다고 하는데, 이로 인해 이 예제는 잘 동작한다. 즉,==가득 찬 채널에 쓰려고 하는 고루틴은 채널이 비워질 때까지 기다리고, 비어 있는 채널에서 읽으려는 고루틴은 적어도 하나의 항목이 있을 때까지 기다린다.== 이 예제에서 fmt.Println은 stringStream 채널에서 값을 가져오기 때문에, 채널에 값이 입력될 때까지 기다린다. 마찬가지로 익명 고루틴이 stringStream에 문자열 리터럴을 넣으려고 하므로 이 고루틴 역시 쓰기가 완료될 때까지 종료되지 않는다. 따라서 main 고루틴과 익명 고루틴 블록은 확실하게 대기한다.


프로그램을 올바르게 구성하지 않으면 데드락이 발생할 수 있다. ==익명 고루틴이 채널에 값을 넣는 것을 방지하기 위해 의미 없는 조건문을 추가한 다음 예제를 살펴보자.==


``` go
package main

  

import "fmt"

  

func main() {
    stringStream := make(chan string)
    go func() {
        if 0 != 1 {  // #1
            return
        }

        stringStream <- "Hello channels!"

    }()
    fmt.Println(<-stringStream)
}
```

1. 이로 인해 stringStream 채널에는 절대로 값이 추가되지 않는다.

``` shell
PS C:\Users\kgm09\goproject\src> go run main.go
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
main.main()
        C:/Users/kgm09/goproject/src/main.go:13 +0x4b
exit status 2
```

main 고루틴은 stringStream에 들어올 값을 기다리지만, 조건문 때문에 그런 일은 발생하지 않는다. 익명 고루틴이 종료되면 Go는 모든 고루틴이 대기 중임을 제대로 감지하고 데드락을 보고한다. 이 절의 뒷부분에서 이와 같은 데드락을 방지하기 위한 첫 번째 단계의 프로그램의 구성 방법에 대해 설명한다. 그리고 4장에서 데드락을 방지하는 방법 전체를 설명할 것이다. 그 사이에 채널에서 값을 읽는 방법에 대해 다시 살펴보자.

``` go
package main

  

import "fmt"

  

func main() {
    stringStream := make(chan string)
    go func() {}
        stringStream <- "Hello channels!"
    }()

    salutation, ok := <-stringStream // #1
    fmt.Printf("(%v): %v", ok, salutation)

}
```

1. 여기서는 문자열인 salutation과 부울(boolean)값이 ok를 받는다.

``` shell
PS C:\Users\kgm09\goproject\src> go run main.go
(true): Hello channels!
```

이 부울 값은 무엇을 의미하는가? 읽기 연산은 이 두 번째 리턴 값을, ==채널의 첫 번째 값이 프로세스 어딘가에서의 쓰기 연산을 통해 생성된 값인지 아니면 닫힌 채널에서 생성되는 기본값인지 나타내기 위해 사용한다.== 그런데 닫힌 채널은 무엇일까?

닫힌 채널은 프로그램에서 더 이상 값이 채널을 통해 전송되지 않는다는 것을 나타낼 수 있기 때문에 매우 유용하다. ==이렇게 하면 값을 수신하는 프로세스가 언제쯤 종료해야 할지, 혹은 언제쯤 새로운 채널이나 다른 채널에서 통신을 열어야 할지 등을 알 수 있다.== 각각의 타입을 나타내는 특별한 부호를 사용할 수도 있지만, 이렇게 하면 모든 개발자가 개별 타입을 처리하기 위한 수고를 반복하게 될 수 있다. 그리고  사실 이것은 채널의 함수이지 데이터 타입의 함수가 아니므로 채널을 닫는다는 것은 **"송신 측에서는 더 이상 값을 쓰지 않을 것이니 하고 싶은 작업을 하시오"** 라는 하나의 범용적인 신호라 할 수 있다. 채널을 닫으려면 다음과 같이 close 키워드를 사용한다.

``` go
valueStream:=make(chan interface{})
close(valueStream)
```

흥미롭게도 닫힌 채널에서도 값을 읽을 수 있다. 다음 예제를 살펴보자.

``` go
import "fmt"

  

func main() {

    intStream := make(chan int)
    close(intStream)
    integer, ok := <-intStream
    fmt.Printf("(%v):%v", ok, integer)

}
```

1. 여기서 닫힏 스트림으로부터 데이터를 읽는다.

다음과 같이 출력된다.

``` shell
PS C:\Users\kgm09\goproject\src> go run main.go
(false):0
```

이 채널에 아무런 데이터도 넣지 않았다는 사실에 주목하라. 예제에서는 이 채널을 바로 닫았다. 여전히 읽기 작업을 수행할 수 있었으며, 실제로 채널이 닫힌 상태에서도 계속해서 이 채널에 대한 읽기 연산을 수행할 수 있었다. ==이는 해당 채널에 데이터를 쓰는 상류의 고루틴이 하나뿐이더라도 여러 개의 하류 고루틴이 데이터를 읽을 수 있도록 지원하기 위한 것이다==(4장을 보면 이것이 일반적인 상황임을 알 수 있다). 이 예제에서 ok 변수에 저장된 두 번째 리턴 값은 false이다. 이는 수신한 값이 정수형(int) 0이거나 스트림에 값이 없다는 것을 나타낸다.

이는 몇 가지 새로운 패턴을 열어준다. 첫 번째는 한 채널에 range에 사용하는 것이다. for 구문과 함께 사용되는 range 키워드는 채널을 인수로 받을 수 있으며, 채널이 닫힐 때 루프를 자동으로 중단한다. 이를 통해 간결하게 채널의 값을 반복할 수 있다. 이 예제를 살펴보자.


``` go
package main

  

import "fmt"

  

func main() {

    intStream := make(chan int)

    go func() {
        defer close(intStream)  // #1
        for i := 1; i <= 5; i++ {
            intStream <- i
        }
    }()

  

    for integer := range intStream {// #2
        fmt.Printf("%v ", integer)
    }

}
```

1. 여기서 고루틴을 종료하기 전에 채널이 닫혀 있는지 확인한다. 이는 매우 일반적인 패턴이다.
2. 여기서 range를 통해 intStream 범위를 순회한다.

이 모든 값이 출력되고 프로그램이 종료되었다는 것을 볼 수 있다.

```
1 2 3 4 5
```
루프에 종료 조건이 필요하지 않으며, range가 두 번째 부울 값을 리턴하지 않는다는 점에 주목하자. 루프를 간결하게 유지하기 위해 닫힌 채널에 대한 세부적인 처리를 관리해준다.

==채널을 닫는 것은 여러 개의 고루틴에 동시에 신호를 보낼 수 있는 방법 중 하나이기도 하다.== 하나나의 채널을 기다리고 있는 n개의 고루틴이 있다면, 각 고루틴의 대기를 해제하기 위해 n 번씩 채널에 쓰는 대신 간단히 채널을 닫아버려도 된다. 닫힌 채널은 무한히 읽혀질 수 있으므로, 얼마나 많은 고루틴이 기다리고 있느냐는 중요하지 않으며, 채널을 닫는 것은 n번의 쓰기를 수행하는 것보다 빠르고 부하가 적다.

다음은 여러 개의 고루틴으로 한 번에 대기 해제하는 예제이다.

``` go
package main

  

import (
    "fmt"
    "sync"
)

  

func main() {

    begin := make(chan interface{})

  

    var wg sync.WaitGroup

    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(i int) {
            defer wg.Done()
            <-begin // #1
            fmt.Printf("%v has begun\n", i)
        }(i)

    }

    fmt.Println("Unblocking goroutines...")
    close(begin) // #2
    wg.Wait()

}
```

1. 계속 진행해도 된다고 할 때까지 고루틴은 여기서 대기한다. 
2. 여기서 채널을 닫으면 모든 고루틴이 동시에 대기 상태에서 벗어난다.

begin 채널을 닫기 전까지 어떠한 고루틴도 시작하지 않는 것을 볼 수 있다.

``` go
PS C:\Users\kgm09\goproject\src> go run main.go
Unblocking goroutines...
0 has begun
4 has begun
1 has begun
3 has begun
2 has begun
```

78페이지의 "sync 패키지"에서 이와 동일한 동작을 수행하기 위해 sync.Cond 타입을 사용하는 방법에 대해 설명했던 것을 떠올려보자. 확실하게 sync.Cond를 사용할 수도 있지만, ==앞서 설명한 것처럼 채널들은 구성 가능하기 때문에 개인적으로는 동시에 여러 고루틴을 대기 해제하기 위해 채널을 닫는 방법을 선호한다.==


인스턴스화될 때 기본 용량을 제공하는 채널인 **버퍼링된 채널**(buffered channel)을 생성할 수도 있다. 버퍼링된 채널에서는 채널에서 읽기가 전혀 수행되지 않더라도 고루틴이 n번의 쓰기를 수행할 수 있다. 여기서 n은 버퍼링된 채널의 용량이다. 다음은 버퍼링된 채널을 선언하고 인스턴스화하는 방법이다.

``` go
var dataStream chan interface{}
dataStream=make(chan interface{},4) //#1
```

1. 여기서 4개의 용량을 가지는 버퍼링된 채널을 생성한다. 즉, 채널을 읽는 중인지 여부에 관계없이 채널에 4개를 배치할 수 있다.


다시 인스턴스를 생성하는 코드를 두 줄로 나누었는데, ==버퍼링된 채널이 선언이 버퍼링 되지 않은 채널의 선언과 다르지 않음을 보여준다.== **이것이 흥미로운 이유는 채널을 인스턴스화하는 고루틴이 버퍼링 여부를 결정한다는 것을 보여주기 때문이다.** 이는 채널의 행동과 성능을 보다 쉽게 추론할 수 있도록, 채널의 생성이 채널에 데이터를 쓰는 고루틴과 밀접하게 결합돼야 한다는 의미이다. 여기에 대해서는 이 절의 뒷부분에서 다시 살펴본다.


==버퍼링되지 않은 채널 역시 버퍼링된 채널의 관점에서 정의된다.== 버퍼링되지 않은 채널은 단순히 0의 용량으로 생성된 버퍼링된 채널이다. 다음은 동일한 기능을 가지는 두 채널을 생성하는 예제이다.

``` go
a:=make(chan int)
b:=make(chan int,0)
```

두 채널은 모두 0의 용량을 가지는 정수(int) 채널이다. 대기(blocking)에 대해 이야기하면서 채널에 쓰려고 할 때 채널이 가득 찼다면 빈 공간이 생길 때까지 대기해야 하고, 읽으려고 할 때 채널이 비어 있으면 데이터가 써질 때까지 대기해야 한다고 했던 것을 기억하는가?

"가득 차"거나 "비어 있음"은 용량 또는 버퍼 크기에 대한 함수이다. 버퍼링되지 않은 채널의 용량은 0이기 때문에 쓰기 전에 이미 가득 차 있다. 수신자가 없고 버퍼 용량이 4인 버퍼링 된 채널은 4회 쓰기 후에 가득 차고, 다섯 번째 쓰기가 발생하면 다섯 번재 항목을 둘 곳이 없기 때문에 대기시킨다. 버퍼링되지 않은 채널과 마찬가지로 버퍼링된 채널도 마찬가지로 대기해야 한다. 이처럼 버퍼링된 채널은 동시에 실행되는 프로세스가 통신할 수 있는 메모리 내의 FIFO 대기열이다.

이를 이해하기 위해 용량이 4인 버퍼링된 채널의 예에서 어떤 일이 일어나는지 살펴보자. 먼저 채널을 초기화하자.

```go
c:=make(chan rune,4)
```

논리적으로 이것은 다음과 같은 4개의 슬롯을 가지는 버퍼로 채널을 만든다.


|     |     |     |     |
| --- | --- | --- | --- |


이제 채널에 써 보자.

```go
c<-'A'
```

이 채널을 읽는 프로세스가 없다면, A 문자는 다음과 같이 채널 버퍼의 첫 번째 슬롯에 위치하게 된다.


| A   |     |     |     |
| --- | --- | --- | --- |


여전히 읽기 프로세스가 없다면, 버퍼링된 채널에서 이루어지는 후속 쓰기 연산들도 다음과 같이 이 채널의 남아 있는 빈 공간을 채울 것이다.

``` go
c<-'B'
```


| A   | B   |     |     |
| --- | --- | --- | --- |


```go
c<-'C'
```


| A   | B   | C   |     |
| --- | --- | --- | --- |


``` go
c<-'D'
```

| A   | B   | C   | D   |
| --- | --- | --- | --- |

4번의 쓰기 이후에는 버퍼링된 채널의 용량인 4는 가득 차게 된다. 이 때 채널에 데이터를 쓰려고 하면 어떤 일이 일어날까?
```go
c<-'E'
```


| A   | B   | C   | D   |
| --- | --- | --- | --- |
~~~ 
<-!E
~~~
이 쓰기 연산을 수행한 고루틴은 대기하게 될 것이다! 이 고루틴은 다른 고루틴이 읽기를 수행하여 버퍼에 공간이 만들어 질 때까지 대기 상태로 유지된다. 이것이 어떤 식으로 이루어지는지 살펴보자.



```
<-c
```

A<-

| B   | C   | D   | E   |
| --- | --- | --- | --- |

보다시피 읽기 연산은 채널의 위치한 첫 번째 문자인 'A'를 받게 되고, 쓰기 연산은 대기 상태에서 벗어나 버퍼의 마지막에는 'E'가 위치하게 된다.

또한 버퍼링된 채널이 비어 있는데 수신자가 있는 경우에는 버퍼가 무시되고 값이 송신자에서 수신자로 직접 전달된다는 사실은 굳이 이야기할 필요도 없을 것이다. ==실제로 이것은 사용자가 알아채지 못하게 이루어지지만, 버퍼링된 채널의 성능 프로파일을 이해할 필요가 있다.==


==**버퍼링된 채널은 특정 상황에서 유용할 수 있지만 주의해서 만들어야 한다**.== 4장에서 보게 되겠지만, 버퍼링된 채널은 성급한 최적화가 되는 경우가 많으며, 데드락이 발생하기 어렵게 만들어 데드락을 숨길 수 있다. 이는 좋은 생각처럼 보이지만, 실제로는 문제를 빨리 드러내가 하는 것이 좋다고 볼 수 있다.

실제로 작업할 때 더 좋은 아이디어를 얻을 수 있도록 버퍼링된 채널을 사용하는 조금 더 복잡한 다른 코드 예제를 살펴보자.

```go
package main

  

import (
    "bytes"
    "fmt"
    "os"
    "sync"
)

  

func main() {

    var stdoutBuff bytes.Buffer // #1
    defer stdoutBuff.WriteTo(os.Stdout) // #2

  

    intStream:=make(chan int,4) // #3

    go func ()  {
        defer close(intStream)
        defer fmt.Fprintln(&stdoutBuff,"Producer Done.")
        for i := 0; i < 4; i++ { // #1
            fmt.Fprintf(&stdoutBuff,"Sending: %d\n",i)
            intStream<-i
        }
    }()

  

    for integer := range intStream {
        fmt.Fprintf(&stdoutBuff,"Receive %v.\n",integer)
    }

}
```


1. 출력이 얼마나 발생할지 모르기 때문에 여기서 메모리 내부에 버퍼를 생성한다. 언제나 그런 것은 아니지만, stdout에 직접 쓰는 것보다 조금 빠르다.
2. 프로세스가 종료되기 전에 버퍼가 stdout에 쓰여지도록 한다.
3. 용량이 4인 버퍼링된 채널을 생성한다.

이 예제에서 stdout으로 출력되는 순서는 비결정적이지만, 익명 고루틴이 어떻게 작동하는지에 대한 대략적인 아이디어를 얻을 수 있다. 결과를 보면 익명의 고루틴이 intStream에 5개의 결과를 모두 넣을 수 있고, main 고루틴이 그 결과를 읽어가기 전에 종료할 수 있다는 것을 알 수 있다.

``` shell
PS C:\Users\kgm09\goproject\src> go run main.go
Sending: 0
Sending: 1
Sending: 2
Sending: 3
Producer Done.
Receive 0.
Receive 1.
Receive 2.
Receive 3.
```
이 예시는 정상적인 상황에서 유용할 수 있는 최적화의 예이다. 채널에 쓰는 고루틴이 얼마나 쓸지 알고 있는 경우, 쓸 만큼의 용량을 가지고 있는 버퍼링 채널을 만드는 것이 도움이 될 수 있다. 그러고 나서 가능한 한 빨리 써야 한다. 물론 여기에는 문제점이 있다. 이는 4장에서 다룬다.

지금까지 버퍼링되지 않은 채널, 버퍼링된 채널, 양방향 채널 및 단방향 채널을 살펴봤다. ==유일하게 다루지 않은 채널의 기본값인 nil이다.== 프로그램은 nil 채널과 어떤 식으로 상호작용할까? 먼저, nil 채널에서 읽기를 시도해보자.

```go
package main

  

import (
    "bytes"
    "fmt"
    "os"
)

  

func main() {

  

    var stdoutBuff bytes.Buffer         // #1
    defer stdoutBuff.WriteTo(os.Stdout) // #2

  

    intStream := make(chan int, 4) // #3
    go func() {
        defer close(intStream)
        defer fmt.Fprintln(&stdoutBuff, "Producer Done.")
        for i := 0; i < 4; i++ { // #1
            fmt.Fprintf(&stdoutBuff, "Sending: %d\n", i)
            intStream <- i
        }
    }()

    for integer := range intStream {
        fmt.Fprintf(&stdoutBuff, "Receive %v.\n", integer)
    }
    var dataStream chan interface{}
    <-dataStream
}
```




``` shell

PS C:\Users\kgm09\goproject\src> go run main.go
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive (nil chan)]:
main.main()
        C:/Users/kgm09/goproject/src/main.go:29 +0x165
exit status 2

```

데드락! 이것은 nil 채널에서의 읽기가 프로그램을 반드시 데드락에 빠뜨리는 것은 아니더라도 대기하게 만든다는 것을 의미한다. 쓰기는 어떨까?

```go
package main

  

import (
    "bytes"
    "fmt"
    "os"
)

  

func main() {

    var stdoutBuff bytes.Buffer         // #1
    defer stdoutBuff.WriteTo(os.Stdout) // #2

  

    intStream := make(chan int, 4) // #3

    go func() {

        defer close(intStream)
        defer fmt.Fprintln(&stdoutBuff, "Producer Done.")

        for i := 0; i < 4; i++ { // #1
            fmt.Fprintf(&stdoutBuff, "Sending: %d\n", i)
            intStream <- i
        }
    }()

  

    for integer := range intStream {
        fmt.Fprintf(&stdoutBuff, "Receive %v.\n", integer)
    }

  

    var dataStream chan interface{}
    dataStream <- struct{}{}
}
```

이는 다음과 같이 출력된다.

``` go
PS C:\Users\kgm09\goproject\src> go run main.go
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send (nil chan)]:
main.main()
        C:/Users/kgm09/goproject/src/main.go:29 +0x188
exit status 2
```

nil 채널에서 쓰는 것도 역시 대기 상태에 빠지는 것으로 보인다. 이제 남은 하나의 연산은 close이다. nil 채널을 닫으려고 하면 어떤 일이 발생할까?


``` go
package main

  

import (

    "bytes"
    "fmt"
    "os"

)

  

func main() {

  

    var stdoutBuff bytes.Buffer         // #1
    defer stdoutBuff.WriteTo(os.Stdout) // #2

  

    intStream := make(chan int, 4) // #3
    go func() {
        defer close(intStream)
        defer fmt.Fprintln(&stdoutBuff, "Producer Done.")
        for i := 0; i < 4; i++ { // #1
            fmt.Fprintf(&stdoutBuff, "Sending: %d\n", i)
            intStream <- i
        }
    }()

  

    for integer := range intStream {
        fmt.Fprintf(&stdoutBuff, "Receive %v.\n", integer)
    }

  

    var dataStream chan interface{}
    close(dataStream)

}
```

이는 다음과 같이 출력한다.

``` shell
PS C:\Users\kgm09\goproject\src> go run main.go
Sending: 0
Sending: 1
Sending: 2
Sending: 3
Producer Done.
Receive 0.
Receive 1.
Receive 2.
Receive 3.
panic: close of nil channel

goroutine 1 [running]:
main.main()
        C:/Users/kgm09/goproject/src/main.go:29 +0x165
exit status 2
```

아마도 nil 채널에 대해 수행되는 모든 연산 중 최악의 결과라고 할 수 있는 패닉이 발생했다. 항상 현재 작업 중인 채널이 초기화돼 있는지 확인하라.


이번에는 채널을 사용해 작업할 때 정의돼 있는 동작에 대한 참고 자료를 작성해보자. 표 3-2 채널에 대한 연산 및 채널 상태에 따른 결과를 나열한다.

- [표 3-2] 채널의 상태에 따른 연산 과


| 연산    | 채널 상태          | 결과                                                 |
| :---- | :------------- | :------------------------------------------------- |
| 읽기    | nil            | 대기                                                 |
| ^     | 열려 있고 비어 있지 않음 | 값                                                  |
| ^     | 열려 있고 비어 있음    | 대기                                                 |
| ^     | 닫혀 있음          | <기본값>, false                                       |
| ^     | 쓰기 전용          | 컴파일 에러                                             |
| 쓰기    | nil            | 대기                                                 |
| ^     | 열려 있고 가득 참     | 대기                                                 |
| ^     | 열려 있고 가득 차지 않음 | 쓰기 값                                               |
| ^     | 닫혀 있음          | 패닉                                                 |
| ^     | 읽기 전용          | 컴파일 에러                                             |
| close | nil            | 패닉                                                 |
| ^     | 열려 있고 비어 있지 않음 | 채널 닫힘, 채널의 모든 값이 빠져나가기 전에 읽기 성고 그 이후에는 기본 값을 읽어온다. |
| ^     | 열려 있고 비어 있음    | 채널 닫힘, 읽기 연산은 기본값을 가져온다.                           |
| ^     | 닫혀 있음          | 패닉                                                 |
| ^     | 읽기 전용          | 컴파일 에                                              |

  
이 표를 살펴보면 문제가 될 만한 몇 몇 항목을 발견할 수 있다. 표에는 고루틴을 대기 상태에 빠뜨리는 세 가지 연산 및 프로그램을 패닉에 이르게 하는 세 가지 연산이 있다. 언뜻 보기에 채널 사용은 위험해 보이지만, 이러한 결과가 나타나는 이유를 검토하고 채널의 사용을 구성하면 무서워할 필요가 없으며, 상황이 이해되기 시작한다. 견고하고 안정적인 무언가를 만들기 위해 다양한 유형의 채널을 구성하는 방법에 대해 살펴보자.


채널을 올바른 상황에 배치하기 위해 가장 먼저해야 할 일은 채널의 **소유권**을 할당하는 것이다. 채널을 인스턴스화하고, 쓰고, 닫는 고루틴이 소유권을 가지고 있다고 정의할 것이다. 가비지 컬랙션이 없는 언어의 메모리와 마찬가지로, 프로그램을 논리적으로 추론하기 위해 어떤 고루틴이 채널을 소유하는지를 명확히 하는 것이 중요하다. ==단방향 채널 선언은 채널을 소유한 고루틴과 단지 채널을 사용하기만 하는 고루틴을 구분할 수 있는 도구 이다.== 채널 소유자는 채널에 대한 쓰기 접근 권한 측면(chan 또는 chan<-)을 가지고 있으며, 채널의 활용자는 읽기 전용 측면(<-chan)을 가지고 있다. 채널의 소유자와 채널 소유자가 아닌 자를 구분하면 위 표 결과는 당연한 것으로, 채널을 소유한 고루틴과 그렇지 않은 고루틴에 책임을 당할 수 있다.

채널의 소유자부터 시작해보자. **채널을 소유한 고루틴**은 반드시 다음을 수행해야 한다.

1. 채널을 인스턴스화 한다.
2. 쓰기를 수행하거나 다른 고루틴으로 소유권을 넘긴다.
3. 채널을 닫는다.
4. 이 목록에 있는 앞의 세 가지를 캡슐화하고 이를 읽기 채널을 통해 노출한다.

이러한 책임을 **채널 소유자**에게 부여하면 몇 가지 일이 일어난다.

- 우리가 채널을 초기화하기 때문에 nil 채널에 쓰는 것으로 인한 데드락의 위험을 제거할 수 있다.
- 우리가 채널을 초기화하기 때문에 nil 채널을 닫을 위험이 있다.
- 우리가 채널이 닫히는 시기를 결정하기 때문에 닫힌 채널에 쓰는 것으로 인한 패닉의 위험을 없앨 수 있다.
- 우리가 채널이 닫히는 시점을 결정하기 때문에 채널을 두 번 이상 닫는 것으로 인한 패닉의 위험을 제거할 수 있다.
- 우리 채널에 부적절한 쓰기가 일어나는 것을 방지하기 위해 컴파일 시점에 타입 검사기를 사용한다.

이제 읽을 때 발생할 수 있는 **대기 연산**을 살펴보자. 채널 소비자로서 두 가지 사항만 신경쓰면 된다.

- 언제 채널이 닫히는지 아는 것
- 어떤 이유로든 대기가 발생하면 책임있게 처리하는 것


첫 번째 사항을 처리하기 위해 앞에서 설명한 것처럼 읽기 연산의 두 번째 리턴 값을 간단히 검사한다. 두 번째 사항은 알고리즘에 달려 있기 때문에 정의하기가 훨씬 더 어렵다. 시간 제한을 원할 수도 있고, 누군가가 당신에게 이야기하면 읽기를 멈추고 싶을 수도 있고, 프로세스의 실행 시간동안 멈춰두는 데 만족할 수도 있다. ==중요한 점은 채널의 소비자로서 읽기가 중단될 수도 있고, 중단될 것이라는 사실을 처리해야 한다는 점이다.== 4장에서 채널을 읽는 고루틴의 목표를 달성하는 방법을 검토한다.

지금 이 개념을 정확히 이해하기 위해 예제를 살펴보겠다. 채널을 명확하게 소유하는 고루틴과 채널의 대기 및 종료를 명확하게 처리하는 소비자를 만들어보자.


``` go

package main

  

import (
    "fmt"
)

  

func main() {

  

    chanOwner := func() <-chan int {

        resultStream := make(chan int, 5) // #1
       
go func() {                       // #2
            defer close(resultStream) // #3

            for i := 0; i < 5; i++ {
                resultStream <- i
            }
        }()
        return resultStream // #4
    }

  

    resultStream := chanOwner()
    
    for result := range resultStream { // #5
        fmt.Printf("Received: %d\n", result)
    }

    fmt.Println("Done receiving!")

  

}
```


1. 여기서는 버퍼링된 채널을 인스턴스화 한다. 6개의 결과를 생산할 것이므로, 고루틴을 가능한 빠르게 완료할 수 있도록 크기가 5인 버퍼링된 채널을 생성한다.
2. 여기서는 resultStream에 쓰기를 수행하는 익명의 고루틴을 시작한다. 고루틴을 만드는 방법을 반대로 한 것에 주목하라. 이는 감싸고 있는 함수로 캡슐화된다.
3. 여기서는 일단 resultSteam이 끝나면 닫히도록 보장한다. 이는 채널 소유자라면 당연한 의무이다.
4. 여기서는 채널을 리턴한다. 리턴 값은 읽기 전용 채널로 선언되므로, resultStream은 소비자를 위해 읽기 전용으로 암시적으로 변환된다.
5. 여기서는 range를 통해 resultStream의 범위를 순회한다. 소비자로서 우리는 대기하고 채널을 닫는 것만 신경 쓰면 된다. 이를 수행한 결과는 다음과 같다.


``` shell
PS C:\Users\kgm09\goproject\src> go run main.go
Received: 0
Received: 1
Received: 2
Received: 3
Received: 4
Received: 5
Done receiving!
```


==resultStream 채널의 수명주기가 chanOwner 함수 내에서 캡슐화되는 방식에 주목하자.== nil 채널이나 닫힌 채널에 대한 쓰기는 발생하지 않으며, 닫기가 언제나 한 번만 발생한다는 사실은 매우 분명하다. 이를 통해 프로그램에서 높은 리스크를 제거할 수 있다. 이를 보장하려면 채널 소유의 범위를 좁게 유지할 수 있도록 프로그램상에서 할 수 있는 일을 하는 것이 좋다.

수많은 메소드가 있는 구조체의 멤버 변수인 채널이 있다면 이 채널의 동작 방식은 금세 불분명해진다.

==소비자 함수는 읽기 채널에만 접근할 수 있으므로, 읽기가 차단됐을 때의 처리 방법 및 채널을 닫는 방법만 알면 된다.== 이 작은 예제에서는 채널이 닫힐 때까지 프로그램의 수명을 완전히 중단시켜도 안전하다는 입장을 취했다.

이 원칙을 따라 코드를 작성하면 시스템에 대해 추론하기가 훨씬 쉬울 것이며, 예상할 수 있겠지만 성능 역시 향상될 것이다. 데드락이나 패닉이 발생하지 않을 것이라고 장담할 수 없지만, 만약 발생한다면 채널 소유의 범위가 너무 크거나 소유권이 명확하지  않는 것을 발견할 수 있을 것이다.

많은 면에서 채널은 고루틴들을 묶는 접착제 역할을 한다. 3장은 채널이 무엇인지, 채널을 어떻게 활용을 해야 하는지에 대한 개요를 제공했다. 진정한 재미는 고차원의 동시성 디자인 패턴을 만들기 위해 채널을 구성할 때 시작된다.


---

# select

select문은 채널을 하나로 묶는 접착제이다. 이 구문은 더 큰 추상화를 형성하기 위해 프로그램에서 여러 채널을 함께 구성할 수 있는 방법이다. 채널이 고루틴들을 묶는 접착제라면 select 문은 무엇을 말하는 것일까? select문은 동시성을 이용한 Go 프로그램에서 가장 중요한 요소중 하나라도 해도 과장이 아니다. ==하나의 함수 또는 타입 내에서 지역적으로 채널들을 바인딩하는 select 문을 찾아볼 수 있으며, 시스템상 두 개 이상의 구성 요소가 교차하는 곳에서도 전역으로 바인딩하는 것을 볼 수 있다.== 구성 요소들을 합치는 것 외에도, 프로그램상의 중요한 특정 시점에 select 문을 사용해 취소, 시간 초과, 대기 및 기본값과 같은 개념을 안전하게 채널에 도입할 수 있다.


반대로 프로그램에서 ==select 구문이 통용되지 않으며 채널을 독점적으로 다루는 경우==에, 이 프로그램의 구성 요소들을 어떤 식으로 상호 조정해야 할까? 이 문제는 5장에서 다룰 것이며, 힌트를 주자면 채널을 사용하는 것을 선호한다.


그렇다면 이 강력한 select 구문은 과연 무엇일까? 어떻게 사용하며, 어떻게 작동할까? 그저 밖으로 꺼내 높기만 하면 된다. 여기 간단한 예제가 있다.


``` go

var c1, c2 <-chan interface{}
var c3 chan <- interface{}
select{
case <-c1:
// 작업 수행

case <-c2:
// 작업 수행

case c3<-struct{}{}:
// 작업 수
}

```

얼핏 switch 블록처럼 보인다. select 블록은 switch 블록과 마찬가지로 일련의 명령문을 보호하는 case 문을 포함한다. 그러나 비슷한 점은 이것이 전부다. ==switch 블록과 달리 select 블록의 case문은 순차적으로 테스트되지 않으며, 조건이 하나도 충족되지 않는다고 다음 조건으로 넘어가지도 않는다.==

대신, select 블록은 채널 중 하나가 준비(읽기의 경우 채워지거나 닫힌 채널, 쓰기의 경우 쓸 수 있는 채널)됐는지 확인하기 위해 모든 채널 읽기와 쓰기를 동시에 고려한다.[^3] 준비된 채널이 없는 경우 select문 전체가 중단돼 대기한다. 그런 다음 채널들 중 하나가 준비되면 해당 연산이 진행되고 관련 구문들이 실행된다.

다음 예제를 살펴보자.


``` go
package main

  

import (
    "fmt"
    "time"
)

  

func main() {

  

    start := time.Now()
    c := make(chan interface{})

    go func() {
        time.Sleep(5 * time.Second)
        close(c) // #1
    }()

    fmt.Println("Blocking on read ...")
    select {
		    case <-c: // #2
        fmt.Printf("Unblocked %v later.\n", time.Since(start))
    }

  

}
```

1. 여기서는 5초 동안 대기한 후에 채널을 닫는다.
2. 여기서는 채널을 읽으려고 시도한다. 이 코드가 작성되었으므로 select 구문은 필요하지 않다. 간단하게 <-c를 쓸 수도 있지만, 이 예제를 확장할 것이다.

이 코드는 다음과 같이 출력된다.

``` shell
PS C:\Users\kgm09\goproject\src> go run main.go
Blocking on read ...
Unblocked 5.0126445s later. // 재개
```

보다시피 select 블록에 진입하고 약 5초 후에 차단을 해제한다.

이것은 무언가 일어나기를 기다리는 동안 대기하는 간단하고 효율적인 방법이다. 그러나 잠시 되돌아보면 몇가지 의문이 생길 수 있다.

- 여러 채널이 읽을 내용이 있을 때는 어떻게 될까?
- 어떠한 채널도 준비되지 않는다면 어떻게 될까?
- 무언가 하고 싶지만 현재 준비된 채널이 없다면 어떻게 될까?

여러 채널이 동시에 준비되는 경우에 대한 첫 번째 질문은 흥미로워 보인다.

무슨 일일 일어나는지 한번 시도해보자!


``` go
package main

  

import (
    "fmt"
)

  

func main() {

  

    c1 := make(chan interface{})
    close(c1)
    c2 := make(chan interface{})
    close(c2)

    var c1Count, c2Count int
  

    for i := 1000; i >= 0; i-- {
        select {
        case <-c1:
            c1Count++
        case <-c2:
            c2Count++
        }
    }

  

    fmt.Printf("c1Count: %d\n c2Count: %d\n", c1Count, c2Count)

}
```


보다시피 1000번을 반복하면 select문의 절반은 c1에서 읽고, 절반은 c2에서 읽는다. 이는 매우 흥미로운 결과로, 어찌보면 지나친 우연으로 보인다. 사실 그렇다! Go의 런타임은 case 구문의 집합에 대해 균일한 의사 무작위(pseudo-random uniform) 선택을 수행한다. 이것은 하나의 집합 내에 case 구문들은 각각 다른 모든 구문과 동일한 확률로 선택될 수 있다는 의미이다.

일견 이것은 중요해 보이지 않을 수 있지만, 여기서 추론할 수 있는 내용은 엄청나게 흥미롭다. Go의 런타임은 select문의 의도에 대해 아무것도 알 수 없다. ==즉, 문제 공간을 추론하거나 채널 그룹을 select 구문에 넣은 이유를 추론할 수 없다.== 이 때문에 Go 런타임이 할 수 있는 최선은 평균적인 경우에 잘 작동하는 것이다. 이를 수행하는 좋은 방법은 랜덤의 변수를 방정식에 도입하는 것이며, 이 경우에는 선택할 채널이 그 변수이다. 각 채널이 활용될 수 있는 기회를 균등하게 가중하면 select 문을 사용하는 모든 Go 프로그램은 일반적으로 잘 수행된다.

두 번째 질문은 어떤가? 준비가 된 채널이 없으면 어떻게 될까? 모든 채널이 대기 중이지만 영원히 대기하는 것도 도움되지 않는다면, 시간 초과가 필요할 수 있다. Go의 time 패키지는, select 패러다임에 잘 맞는 채널을 이용해 이를 수행할 수 있는 우아한 방법을 제공한다.


``` go
package main

  

import (
    "fmt"
    "time"
)

  

func main() {

  

    var c <-chan int

    select {
    case <-c: // #1
    case <-time.After(1 * time.Second):
        fmt.Println("Timed out.")
    }

}
```

1. 이 case문을 nil 채널을 읽기 때문에 대기 상태를 벗어날 수 없다.

```
PS C:\Users\kgm09\goproject\src> go run main.go
Timed out.
```

time.After 함수는 time.Duration 인수를 받아서, 사용자가 넘겨준 기간이 지나 후에 현재 시간을 보낼 채널을 리턴한다. 이것은 select문에서 시간 초과를 구현하는 간결한 방법을 제공한다. 4장에서 이 패턴을 다시 살펴보고, 이 문제에 대한 보다 강력한 해결책을 논의할 것이다.

그렇다면 질문 하나가 떠오른다. ==어떠한 채널도 준비가 되지 않았을 때 어떤 일이 일어나며, 그 동안 무엇을 해야 하는가?== select문은 case 문과 마찬가지로 선택하는 모든 채널이 차단되어 대기하는 경우에 대비해 default 절을 허용한다. 예제를 살펴보자.


``` go
package main

  

import (
    "fmt"
    "time"
)

  

func main() {

  

    start := time.Now()
    var c1, c2 <-chan int

  

    select {
    case <-c1:
    case <-c2:
        fmt.Printf("In default after %v\n\n", time.Since(start))
    }

}
```

다음과 같이 출력될 것이다.

``` shell
PS C:\Users\kgm09\goproject\src> go run main.go
In default after 0s

```

거의 즉시 default 명령문이 실행됐음을 알 수 있다. 이렇게 하면 기다리지 않고 select 블록을 빠져나올 수 있다. 일반적으로 default 절이 for-select 루프와 함께 사용되는 것을 볼 수 있다. ==이렇게 하면 고루틴은 다른 고루틴이 결과를 보고하기를 기다리는 동안 작업을 진행할 수 있다.==

예를 들면 이러하다.

``` go
package main

  

import (
    "fmt"
    "time"
)

  

func main() {

  

    done := make(chan interface{})

    go func() {
        time.Sleep(5 * time.Second)
        close(done)
    }()

  

    workCounter := 0

loop:

    for {
        select {
        case <-done:
            break loop
        default:
        }

        // 작업을 시뮬레이션 한다.
        workCounter++
        time.Sleep(1 * time.Second)

    }

  

    fmt.Printf("Achieved %v cycles of work before signalled to stop.\n", workCounter)

  

}
```

이는 다음과 같이 출력된다.

``` go
PS C:\Users\kgm09\goproject\src> go run main.go
Achieved 5 cycles of work before signalled to stop.
```

이 경우, 어떤 종류의 작업을 수행하면서 때로는 멈춰야 하는지를 점검하는 루프를 가지고 있다.

마지막으로, 빈 select문에는 case 절이 없는 select 문이 있다. select 문의 형태는 다음과 같다.

select{}

이 구문은 단순하게 영원히 대기한다.

6장에서는 select 문이 어떻게 작동하는지 자세히 살펴보겠다. 보다 높은 수준의 관점에서볼 때, select문이 다양한 하위 시스템을 함께 안전하고 효율적으로 구성하는데 어떤 식으로 도움이 되는지 분명하다.


---

# GOMAXPROCS

runtime 패키지는 GOMAXPROCS라는 함수가 있다. 내 생각에 이 이름은 오해의 소지가 있다. 사람들은 종종 이 함수가 호스트 시스템의 논리적인 프로세서의 수와 관련돼 있다고 생각한다. 하지만 실제로 ==이 함수는 소위 "작업 대기열"이라고 불리는 OS 스레드의 수를 제어한다.== 이 함수의 작동 방식 및 이에 대한 자세한 내용은 6장을 참고하자.

Go 1.5 이전 버전에는 GOMAXPROCS가 항상 1로 설정되어 있으며, 일반적으로, 대부분의 Go 프로그램에서 다음과 같은 코드 조각을 찾을 수 있었다.

``` go
runtime.GOMAXPROCS(runtime.NumCPU())
```

개발자 대부분은 자신의 프로세스가 실행 중인 시스템의 코어를 모두 활용하고자 한다. 이 때문에 이후의 Go 버전에서는 호스트 시스템의 논리적인 CPU 수로 자동 설정된다.

그렇다면 이 값을 왜 조정하려고 하는 것일까? 대부분의 경우 그렇게 하는 것을 원치 않았을 것이다. ==Go의 스케줄링 알고리즘은 대부분의 상황에서 충분히 훌륭하기 때문에 작업자 대기열 및 스레드의 수를 늘리거나 줄인다고 해도 득보다 실이 더 클 수 있지만, 여전히 이 값을 변경하는 것이 유용한 상황도 있다.==

예를 들어, 레이스 컨디션에 시달리는 테스트 모음(test suite)이 있는 프로젝트에 참여했던 적이 있다. 이 팀에는 어찌된 일인지 가끔씩 테스트에 실패하는 패키지가 몇 개 있다. 테스트를 실행한 인프라에는 논리적인 CPU가 4개 밖에 없었으므로, 어느 시점이든 동시에 4개의 고루틴들이 실행했다. GOMAXPROCS를 우리가 보유한 논리적 CPU의 수 이상으로 늘림으로써 훨씬 더 자주 레이스 컨디션을 유발할 수 있었으며, 따라서 이 문제를 신속하게 수정할 수 있었다.


누군가는 실험을 통해 자신의 프로그램이 특정 수의 작업자 대기열과 스레드에서 더 잘 실행된다는 사실을 발견할 수도 있지만 주의해야 한다. 이 함수를 이용해 성능을 조정하는 경우에는, 모든 커밋이 이루어질 때마다 사용하는 하드웨어나 Go의 버전에 맞춰 조정을 수행해야 한다. 이 값을 조정하면 프로그램을 최대한의 속도로 몰아부칠 수 있지만, 추상화와 장기적인 성능 안정성을 희생해야 한다.


[^1]: fork-join 모델은 동시성이 수행되는 방법에 대한 논리적인 모델이다. 이 모델은 fork와 wait를 호출하는 C 프로그램을 논리적인 수준에서만 설명해준다. fork-join 모델은 메모리가 관리되는 방식에 대해서는 아무것도 말해주지 않는다.
[^2]: 개인적으로 O'Reilly의 우수한 도서인 [Head first design patterns]을 추천한다.
