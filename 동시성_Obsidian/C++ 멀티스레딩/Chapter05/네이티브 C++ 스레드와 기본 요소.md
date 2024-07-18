
2011년에 시작한 C++ 표준 개정안에서 멀티스레딩 API가 공식적으로 C++ 표준 템플릿 라이브러리(STL, Standard Template Library)의 일부가 됐다. ==이것은 스레드와 스레드 기본 요소(primitive), 동기화 메커니즘이 서드파티 라이브러리를 설치하지 않고 또는 운영체제의 API에 의존하지 않고서 새로운 C++ 애플리케이션에서 이용 가능하다는 것을 의미한다.==


5장에서는 2014년 표준까지 추가된 이런 네이티브 API에서 사용할 수 있는 멀티스레딩 기능을 살펴본다. 

5장에서는 다음과 같은 주제를 다룬다.

- C++ STL에서 멀티스레딩 API가 다루는 기능
- 이런 기능에 대한 사용법을 보여주는 세부적 예제



---

# STL 스레딩 API


3장 'c++ 멀티스레딩 API'에서 멀티스레드 C++ 애플리케이션 개발에 사용할 수 있는 다양한 API를 살펴봤다.  4장, '[[스레드 동기화와 통신]]'에서는 네이티브 C++ 스레딩 API를 이용해 멀티스레드 스케줄러 애플리케이션을 구현해봤다.


### Boost.Thread API

STL의 <\thread> 헤더를 포함하면 추가적인 헤더가 제공하는 상호 배체(뮤텍스 등) 기능을 가진 std::thread 클래스가 접근할 수 있다. 이 API는 기본적으로 Boost.Thread의 멀티스레딩 API와 동일하다. 주요 차이점은 뮤텍스와 조건 변수 같은 기본 요소 위에 구현된 다수의 추가적인 락 유형과 스레드(타임아웃이 있는 합류, 스레드 그룹, 스레드 인터럽션)에 대해 좀 더 많은 제어를 할 수 있다는 점이다.

일반적으로 Boost.Thread는 C++11 지원이 없을 때 또는 추가적인 Boost.Thread 기능이 애플리케이션의 요구 조건일 때를 대비하여 사용돼야 한다. Boost.Thread는 가용한(네이티브) 스레딩을 기반으로 하기 때문에 C++STL 구현에 비해 오버헤드가 추가될 가능성이 있다.



### 2011 표준

C++ 표준에 대한 2011 개정안(흔히 C++11)은 광범위한 새로운 기능을 추가했다. 가장 중요한 기능으로 네이티브 멀티스레딩 지원을 들 수 있는데, [[서드파티 라이브러리]]를 사용하지 않고서 C++ 내에서 스레드를 생성하고 관리하며, 사용할 수 있는 기능이다.

이 표준안은 스레드-로컬 스토리지 같은 기능을 사용할 수 있을 뿐만 아니라 여러 스레드가 공존할 수 있도록 핵심 언어에 대한 메모리 모델을 표준화한다. 최초의 지원은 C++03에서 추가됐지만 C++11 표준이 이 기능을 완전히 사용할 수 있는 최초의 버전이다.


앞서 언급했듯이, 실질적인 스레딩 API 자체는 STL에 구현돼 있다==. C++11(C++0x)표준의 목적 중의 하나는 하나는 핵심 언어의 일부가 아닌 STL에 가급적 많은 기능을 가지는 것이었다.==

새로운 멀티스레딩 API를 작업한 표준 위원회는 자신들의 목표가 있었고 결과적으로 일부에서 원했던 몇몇 기능은 최종 표준안에 들어가지 못했다. ==스레드 취소로 인해 파기될 스레드에서 자원 정리 같은 문제 때문에 POSIX 대변자들이 강력하게 반대했던 다른 스레드를 종료하는 기능이나 스레드 취소같은 기능이 여기에 해당한다.==

다음은 이 API 구현이 제공하는 기능이다.

- std::thread
- std::mutex
- std::recursive_mutex
- std::condition_variable
- std::condition_variable_any
- std::lock_guard
- std::unique_lock
- std::packaged_task
- std::async
- std::future


당분간 이들 각 기능에 대한 세부적 예제를 살펴볼 예정이다. 먼저 차지 C++ 표준 개정안에 추가된 이에 대한 초기 안을 살펴보자.


---
# C++14

2014 표준안은 표준 라이브러리에 다음과 같은 기능을 추가했다.

- std::shared_lock
- std::shared_timed_mutex


이들 두 항목은 `<shared_mutex>` STL 헤더에 정의돼 있다. 락은 뮤텍스를 기반으로 하기 때문에 공유 락은 공유 뮤텍스에 의존한다.

---
# C++17

2017 표준안은 표준 라이브러리에 다음과 같은 추가적인 기능을 갖는다.

- std::shared_mutex
- std::scoped_lock

여기에서 범위가 **지정된 락**(scoped lock)은 범위가 지정된 블록 기간 동안 뮤텍스를 소유하는 RAII-스타일 메커니즘을 제공하는 뮤텍스 래퍼다.

---
# STL 구성

STL에서 다음과 같은 헤더 조직과 이들이 제공하는 기능을 볼 수 있다.


| 헤더                     | 제공하는 것                                           |
| ---------------------- | ------------------------------------------------ |
| `<thread>`             | std::thread 클래스, std::this_thread 이름 공간 아래의 메소드: |
| ^                      | ^ - yield                                        |
| ^                      | ^ -get_id                                        |
| ^                      | ^ - sleep_for                                    |
| ^                      | ^ - sleep_until                                  |
| `<mutex>`              | 클래스:                                             |
| ^                      | - mutex                                          |
| ^                      | - timed_mutex                                    |
| ^                      | - recursive_mutex                                |
| ^                      | - recursive_timed_mutex                          |
| ^                      | - lock_guard                                     |
| ^                      | - scoped_lock (C++17)                            |
| ^                      | - unique_lock                                    |
| ^                      | 함수:                                              |
| ^                      | - try_lock                                       |
| ^                      | - lock                                           |
| ^                      | - call_once                                      |
| ^                      | - std::swap(std::unique_lock)                    |
| `<shared_Mutex>`       | 클래스:                                             |
| ^                      | - shared_mutex (C++ 17)                          |
| ^                      | - shared_timed_mutex  (C++14)                    |
| ^                      | - shared_lock (C++14)                            |
| ^                      | 함수:                                              |
| ^                      | - std::swap(std::shared_lock)                    |
| `<future>`             | 클래스:                                             |
| ^                      | - promise                                        |
| ^                      | - packaged_task                                  |
| ^                      | - future                                         |
| ^                      | - shared_future                                  |
| ^                      | 함수:                                              |
| ^                      | - async                                          |
| ^                      | - future_category                                |
| ^                      | - std::swap (std::promise)                       |
| ^                      | - std::swap (std::packaged_task)                 |
| `<condition_variable>` | 클래스:                                             |
|                        | - condition_variable                             |
|                        | - condition_variable_any                         |
|                        | 함수:                                              |
|                        | - notify_all_at_thread_exit                      |


---

# 스레드 클래스

스레드 클래스는 전체 스레딩 API의 핵심이 되는 부분이다. 이 클래스는 하부의 운영체제 스레드를 감싸고 스레드를 시작하고 중지시키는 데 필요한 기능을 제공한다.

#### 기본 사용


스레드는 생성과 동시에 즉시 시작한다.

``` c++
#include <thread>

  
  

void worker(){
    // Business logic.
}

  

int main(){
    std::thread t(worker);
    return 0;
}

```


이 코드는 스레드를 시작하고 시작한 스레드의 실행 종료를 대기하지 않기 때문에 애플리케이션은 즉시 종료한다. 올바르게 할려면 다음과 같이 스레드의 종료를 대기하거나 합류하기를 기다려야 한다.

``` c++
#include <thread>

  
  

void worker(){
    // Business logic.
}

  

int main(){
    std::thread t(worker);
    t.join(); // 대기한 다음에 합류함.
    return 0;
}
```


### 인자 전달

또한 새로운 스레드에 인자를 전달할 수도 있다. 이들 인자 값은 이동 가능해야 한다 즉, 이동 또는 복사 생성자(rvalue 참조로 부른다)를 가지는 유형임을 의미한다. 실제로 이것은 모든 기본 유형과 대부분의 (사용자 정의) 클래스의 경우에 그렇다.

``` c++
#include <thread>
#include <string>

  

void worker(int n,std::string t){
    // Business logic.
}

  

int main(){

    std::string s="Test";
    int i=1;
    std::thread t(worker,i,s);
    t.join();
    return 0;

}
```

이 코드에서 하나의 정수와 문자열을 스레드 함수에 전달한다. 이 함수들은 이들 두 변수에 대한 복사본을 받는다. 참조나 포인터를 전달할 대 생명 주기 문제와 데이터 경쟁, 이런 부류의 잠재적인 문제로 인해 더욱 복잡해진다.


### 반환값

스레드 클래스 생성자에 전달된 함수에 의해 반환된 값은 무시된다. ==새로운 스레드를 생성한 스레드에 대한 정보를 반환하기 위해(뮤텍스 같은) 스레드 간의 동기화 메커니즘과 일종의 공유 변수를 사용해야 한다.==


### 스레드 이동하기

2011 표준안은 `<utility>` 헤더에 std::move를 추가했다. 이 탬플릿 메소드를 이용하여 객체 간의 자원을 이동할 수 있다. 이것은 또한 스레드 인스턴스도 이동할 수 있음을 의미한다.


``` c++

#include <thread>
#include <string>
#include <utility>


// 스레드 이동하기
//

  
void worker(int n,std::string t){
    // Business logic.
}

  
  

int main(){

    std::string s="Test";
    std::thread t0(worker,1,s);

    std::thread t1(std::move(t0));
    t1.join();
    return 0;
}
```

이 버전의 코드에서 다른 스레드로 스레드를 이동하기 전에 한 스레드를 생성한다. 따라서 (즉시 종료되기 때문에) 스레드 0은 존재하지 않으며 스레드 실행은 생성된 새로운 스레드에서 재개된다.

이로써 첫 번째 스레드가 재합류하기를 대기할 필요가 없고 두 번째 스레드만 대기하면 된다.

### 스레드 ID

스레드는 자신과 관련된 식별자를 가진다. 이 ID(또는 핸들)는 STL 구현에 의해 제공되는 고유한 식별자다. thread 클래스 인스턴스의 get_id() 함수를 호출해 스레드 ID를 구할 수 있고 또는 std::this_thread::get_id()를 호출해 함수를 호출하는 스레드의 ID를 구할 수 있다.

``` c++

#include <iostream>
#include <thread>
#include <chrono>
#include <mutex>

  
  

// 스레드 ID
std::mutex display_mutex;

void worker(){

    std::thread::id this_id=std::this_thread::get_id();
  
	display_mutex.lock();
    std::cout<<"thread"<<this_id<<"sleeping ...\n";
    display_mutex.unlock();

    std::this_thread::sleep_for(std::chrono::seconds(1));
}

  

int main(){

    std::thread t1(worker);
    std::thread::id t1_id=t1.get_id();

    std::thread t2(worker);
    std::thread::id t2_id=t2.get_id();

  

    display_mutex.lock();
    std::cout<<"t1's id:"<<t1_id<<"\n";
    std::cout<<"t2's id:"<<t2_id<<"\n";
    display_mutex.unlock();

    t1.join();
    t2.join();

std::cout<<"end";

    return 0;

}
```

이 코드는 다음과 같은 결과를 보여줄 것이다.

``` go

PS C:\C-language\Mutlithreading_CPP\Chapter05> ./ch05_mt_example_3
t1's id:2
t2's id:3
thread2sleeping ...
thread3sleeping ...
end
```

여기서 내부 스레드 ID는 최초 스레드(ID 1)에 **상대적인 값을 갖는 정수(std::thread::id 유형)** 임을 볼 수 있다. 이것은 POSIX의 ID와 같이 네이티브 스레드 ID와 유사하다. 이들은 native_handle() 함수를 사용해도 구할 수 있다. 이 함수는 하부의 네이티브 스레드 핸들이 무엇이든 이를 반환한다. 이것은 STL 구현에서 사용할 수 없는 매우 특수한 Pthread또는 Win32 기능을 사용하고자 할 때 특히 유용하다.

### 슬립(Sleep)


두 메소드 가운데 하나를 사용해 스레드의 실행을 지연(슬립 sleep)시킬 수 있다. 이 가운데 한 메소드인 **sleep_for()** 는 최소 지정된 기간 동안(이 시간을 초과할 수 있다) 실행을 지연시킨다.

``` c++
#include <iostream>
#include <chrono>
#include <thread> 

using namespace std::chrono_literals;
// 슬립(sleep)
typedef std::chrono::time_point<std::chrono::high_resolution_clock> timepoint;

int main(){

    std::cout<<"Starting sleep.\n";

    timepoint start=std::chrono::high_resolution_clock::now(); //시작
    std::this_thread::sleep_for(2s);
    timepoint end=std::chrono::high_resolution_clock::now(); // 끝

  
    std::chrono::duration<double,std::milli> elapsed=end-start;

  
    std::cout<<"slept for:"<<elapsed.count()<<"ms\n";
}


```

이 코드는 현재 OS에서 가능한 최고 정확도의 카운터를 사용해 정확한 기간을 측정함으로 대략 2초 동안 슬립하는 방법을 보여준다.


(초 단위의 수에 s 접미어를 붙여) 슬립할 시간을 직접 지정할 수 있음에 주목하자. 이것은 `<chrono>` 헤더에 추가된 C++14의 기능이다. C++11 버전에서는 std::chrono::seconds 인스턴스를 생성하여 이를 sleep_for() 함수에 전달해야 했다.

또 다른 메소드인 sleep_until()은 **std::chrono::time_point<Clock,Duration>** 유형의 한 인자를 가진다. ==이 함수를 사용해 지정된 시점에 도달할 때까지 스레드가 슬립하도록 설정할 수 있다.== ==운영체제의 스케줄링 우선순위로 인해 깨어나는 시간은 명시된 시간과 정확하게 일치하지 않을 수 있다.==


### 양보(Yield)

현재 스레드가 재스케줄될 수 있음을 OS에게 알림으로 다른 스레드가 대신 실행되게 할 수도 있다. 이를 위해 **std::this_thread::yield()** 함수를 사용한다. 이 함수의 정확한 결과는 하부의 OS 구현과 스케줄러에 달려 있다. FIFO 스케줄러라면 호출 스레드는 큐의 뒷부분에 놓여질 가능성이 있다.

이는 특수한 경우에 사용하는 매우 특별한 함수다. 이 함수는 애플리케이션의 성능에 대한 영향을 먼저 검증하지 않고서 사용하면 안 된다.


### 분리(Detach)

스레드를 시작한 이후에 이 스레드 객체에 대해 **detach()** 를 호출할 수 있다. 이것은 효과적으로 새 스레드를 호출 스레드로부터 분리한다. 즉, ==새로운 스레드는 호출 스레드가 종료된 이후에도 계속 실행을 한다.==


### 스왑(swap)


독립적 메소드 또는 스레드 인스턴스의 함수로서 **swap()** 을 사용해 스레드 객체의 내부 스레드 핸들을 교환할 수 있다.


``` c++
#include <iostream>
#include <thread>
#include <chrono>

  

void worker(){
    std::this_thread::sleep_for(std::chrono::seconds(1));
}

int main(){

    std::thread t1(worker);
    std::thread t2(worker);

  

    std::cout<<"thread 1 id: "<<t1.get_id()<<"\n";
    std::cout<<"thread 2 id: "<<t2.get_id()<<"\n";
    std::swap(t1,t2);
    std::cout<<"Swapping threads..."<<"\n";

  

    std::cout<<"thread 1 id: "<<t1.get_id()<<"\n";
    std::cout<<"thread 2 id: "<<t2.get_id()<<"\n";
    t1.swap(t2);
    std::cout<<"Swapping threads..."<<"\n";

        std::cout<<"thread 1 id: "<<t1.get_id()<<"\n";
    std::cout<<"thread 2 id: "<<t2.get_id()<<"\n";
    t1.join();
    t2.join();

}
```

이 코드의 결과는 다음과 유사할 것이다.

``` go
PS C:\C-language\Mutlithreading_CPP\Chapter05> ./ch05_mt_example_5
thread 1 id: 2
thread 2 id: 3
Swapping threads...
thread 1 id: 3
thread 2 id: 2
Swapping threads...
thread 1 id: 2
thread 2 id: 3
```

이의 효과는 ==각 스레드의 상태가 다른 스레드의 상태와 교환한다.== 실제로는 이들의 ID를 교환한다.

---
# 뮤텍스

`<mutex>`헤더는 뮤텍스와 락에 대한 여러 유형을 포함한다. 뮤텍스 유형이 가장 흔히 사용되는 유형이며 추가적인 복잡함이 없이 기본적인 락/언락 기능을 제공한다.


### 기본 사용

==뮤텍스의 목표는 **데이터 손상을 방지**하고== ==스레드 안전하지 않는 루틴의 사용으로 인한 충돌을 방지하기 위해 동시 접근 가능성을 배제하는 것이다.==

뮤텍스를 사용해야 하는 경우의 예는 다음과 같다.

``` c++
#include <iostream>
#include <thread>

  

void worker(int i){
    std::cout<<"Outputting this from thread number:"<<i<<std::endl;
}

int main(){

    std::thread t1(worker,1);
    std::thread t2(worker,2);
    t1.join();
    t2.join();

    return 0;

}
```

==방금 살펴본 코드를 있는 그대로 실행하고자 한다면 두 스레드의 텍스트 출력이 차례대로 출력되는 대신 함께 뒤섞이게 된다.== 그 이유는 표준 출력(C 또는 C++ 스타일)이 스레드에 안전하지 않기 때문이다. 애플리케이션이 비정상 종료하지는 않겠지만 그 출력은 뒤섞이게 된다.

이에 대한 수정은 다음과 같이 간단하다.

``` c++
#include <iostream>
#include <thread>
#include <mutex>

  

std::mutex globalMutex;

  

void worker(int i){
    globalMutex.lock();
    std::cout<<"Outputting this from thread number:"<<i<<std::endl;
    globalMutex.unlock();

}

  

int main(){

    std::thread t1(worker,1);
    std::thread t2(worker,2);
    t1.join();
    t2.join();

  
    return 0;

}
```

이제 각 스레드는 먼저 뮤텍스 객체에 대한 접근을 획득한다. ==단 하나의 스레드만이 뮤텍스 객체에 접근할 수 있기 때문에 나머지 스레드는 첫 번째 스레드가 표준 출력에 쓰는 작업을 마치기를 대기할 것이고 문자열은 의도한 대로 차례대로 출력된다.==



### 블록이 없는 락

스레드가 블록되어 뮤텍스 객체가 이용 가능해질 때까지 대기해야 하는 것을 원치 않을 수도 있다. ==예를 들어 요청이 이미 다른 스레드에 의해 처리되고 있는지 여부만을 알고자 하는 경우에는 요청이 끝날 때까지 대기할 필요가 없을 것이다.==

이런 용도로 try_lock() 함수와 함께 뮤텍스가 제공된다.

다음 예제에서는 동일한 카운터를 증가시키고자 하는 두 스레드가 있다. ==이 가운데 한 스레드는 공유 카운터에 대한 접근을 즉시 획득시키지 못하면 자신의 카운터를 증가시킨다.==

``` c++
#include <chrono>
#include <mutex>
#include <thread>
#include <iostream>

  

std::chrono::milliseconds interval(50);
std::mutex mutex;
int shared_counter=0;
int exclusive_counter=0;

  

void worker0(){
    std::this_thread::sleep_for(interval);

while (true)

{
    if(mutex.try_lock()){
        std::cout<<"Shared ("<<shared_counter<<")\n";
        mutex.unlock();
        return;

    }else{
        ++exclusive_counter;
        std::cout<<"Exclusive ("<<exclusive_counter<<")\n";
        std::this_thread::sleep_for(interval);
    }
}
}

void worker1(){

    mutex.lock();
    std::this_thread::sleep_for(10*interval);
    ++shared_counter;
    mutex.unlock();

}

  

int main(){

    std::thread t1(worker0);
    std::thread t2(worker1);
    t1.join();
    t2.join();

}
```


이 예제에서 두 스레드는 다른 작업자 함수를 실행하지만 둘 다 일정 기간 동안 슬립하고서 깨어날 때 공유 카운터에 대한 뮤텍스를 획득하고자 하는 공통점을 가진다. 뮤텍스를 획득하면 이들은 카운터를 증가시키는데 단, 첫 번째 작업자만이 그 사실을 출력한다.

또한 첫 번째 작업자는 공유 카운터를 얻지 못할 때 로그를 남기지만 자신의 카운터는 증가시킨다.


``` shell
PS C:\C-language\Mutlithreading_CPP\Chapter05> ./ch05_mt_example_7
Exclusive (1)
Exclusive (2)
Exclusive (3)
Exclusive (4)
Exclusive (5)
Exclusive (6)
Exclusive (7)
Exclusive (8)
Shared (1)
```

### 타임드 뮤텍스

타임드 뮤텍스(timed mutex)는 락 획득 시도가 이뤄져야 할 기간에 대한 제어가 가능한 여러 함수(예를 들어 try_lock_for와 try_lock_until)를 가지는 일반 뮤텍스의 한 유형이다.

**try_lock_for** 함수는 결과(true 또는 false)를 ==반환하기 전에 지정 시간 동안(std::chrono::object)에 락 획득을 시도한다.== **try_lock_until** 함수는 결과를 반환하기 전에 미래의 특정 시점까지 대기한다.

이들 함수의 사용은 주로 일반 뮤텍스의 블록(락)과 비블록(try_lock) 메소드 간의 중간 경로를 제공하는데 있다. 태스크를 언제 사용할 수 있는지 알지 못해도 하나의 스레드만을 사용해  여러 태스크를 대기하거나 또는 대기하는 것이 더 이상 의미가 없는 특정 시점에 만료할 수 있는 태스크를 대기할 수도 있다.



### 락 가드


**락 가드**(lock guard)는 ==뮤텍스 객체에 대한 락 획득과 락 가드가 범위를 벗어날 때 락 해제를 다루는 단순한 뮤텍스 래퍼다.== 이것은 뮤텍스 락을 해제하는 것을 잊지 않도록 보장하고 또한 여러 위치에서 동일한 뮤텍스를 해제해야 하는 경우에 코드의 난잡함을 줄이는 데 도움이 되는 메커니즘이다.

예를 들어 커다란 if/else 블록을 리팩토링하면 뮤텍스 락의 해제 부분을 줄여 줄 수 있겠지만, ==이 락 가드 래퍼를 사용하면 훨씬 사용하면 훨씬 손쉽게 가능하고 세부적 사항을 염려할 필요가 없다.==

``` c++
#include <thread>
#include <mutex>
#include <iostream>

int counter=0;
std::mutex counter_mutex;

  

void worker(){

    std::lock_guard<std::mutex> lock(counter_mutex);

    if(counter==1){counter+=10;}
    else if (counter>=10){counter+=15;}
    else if(counter>=50){return;}
    else{++counter;}
  
    std::cout<<std::this_thread::get_id()<<": "<<counter<<std::endl;
}

  

int main(){

    std::cout<<__func__<<": "<<counter<<std::endl;
    std::thread t1(worker);
    std::thread t2(worker);
    t1.join();
    t2.join();
    std::cout<<__func__<<": "<<counter<<std::endl;

}
```

이 예제에서 작업자 함수가 즉시 복귀되도록 하는 조건이 있는 작은 크기의 if/else 블록을 볼 수 있다. ==락 가드가 없었다면 함수에서 복귀하기 전에 각 조건에서 뮤텍스를 언락시켜야 했을 것이다.==

하지만 락 가드 덕택에 이런 세부적 사항을 염려할 필요가 있다. 이로써 뮤텍스 관리를 염려하는 대신 비즈니스 로직에 전념할 수 있다.


### 고유 락

고유 락(unique lock)은 일종의 **범용 뮤텍스 래퍼**이다. ==타임드 뮤텍스와 유사하지만 추가적인 특징을 지닌다. 그중 하나가 소유권 개념이다.== ==다른 락 유형과 달리 고유 락은 뮤텍스를 포함한다고 하더라도 자신이 감싸는 뮤텍스를 소유할 필요가 없다.== 뮤텍스는 **swap()** 함수를 사용해 앞서 언급한 뮤텍스의 소유권과 함께 고유 락 인스턴스 간의 전송이 가능하다.


고유 락 인스턴스가 자신의 뮤텍스에 대한 소유권을 가질지 여부와 그 뮤텍스과 락 됐는지 아닌지 여부는 **락을 생성할 때 먼저 생성**된다. 다음에 그 예제가 있다.


``` c++

std::mutex m1, m2, m3;
std::unique_lock<std::mutex> lock1(m1,std::defer_lock);
std::unique_lock<std::mutex> lock2(m2,std::try_lock);
std::unique_lock<std::mutex> lock3(m3,std::adopt_lock);
```

이 코드의 첫 번째 생성자는 할당된 뮤텍스를 락하지 않는다(지연시킨다). 두 번째 생성자는 try_lock()을 이용해 뮤텍스 락을 시도한다. 끝으로 마지막 생성자는 제공된 뮤텍스를 이미 소유했다고 가정한다.

이외의 다른 생성자를 이용해 타임드 뮤텍스 기능을 활용할 수 있다. ==즉, 어떤 시간 지점에 도달할 때까지나 락이 획득될 때까지 특정 시간을 대기한다.==

마지막으로 락과 뮤텍스 간의 관계는 release() 함수를 이용해 해제되고 뮤텍스 객체에 대한 포인터가 반환된다. 그러고 나서 호출자는 뮤텍스에 남아 있는 락을 해제하고 추가적인 처리를 해야 할 처리를 가진다.

이 유형의 락은 일반적인 경우처럼 자체적으로 그렇게 자주 사용되는 락은 아니다. 그 밖의 다른 대부분의 뮤텍스와 락은 훨씬 덜 복잡하고 모든 경우에 99% 정도의 여구를 충족시켜 줄 것이다. 따라서 고유 락의 복잡성은 장점이기도 하고 위험하기도 하다.

하지만 잠시 뒤 보겠지만 이 락은 조건 변수와 같이 C++11 스레딩 API의 다른 부분에서 흔히 사용된다.

==고유 락이 유용한 한 사용처는 C++17 표준의 네이티브 범위 락에 의존하지 않고도 범위 락을 사용할 수 있는 **범위 락**이다.== 다음 예제를 보자.


``` c++
#include <mutex>
std::mutex my_mutex;
int count=0;

int function(){
std::unique_lock<mutex> lock(my_mutex);
count++;
}
```

이 함수에 진입할 때 전역 뮤텍스 인스턴스를 갖는 새로운 **unique_lock**을 생성한다. 이 뮤텍스는 이 시점에서 락이 되고 이후로 중요한 작업을 수행할 수 있다.

이 함수의 범위가 끝날 때 unique_lock의 소멸자가 호출돼 결과적으로 뮤텍스는 다시 언락 상태가 된다.


### 범위 락

2017 표준안에 처음 소개된 범위 락(scoped lock)은 제공된 뮤텍스에 대한 접근을 획득하는(즉, 락시키는) ==일종의 뮤텍스 래퍼로 범위 락이 범위를 벗어날 때 언락을 보장한다.== 이것은 하나가 아닌 ==여러 뮤텍스에 대한 래퍼==라는 점에서 락 가드와 다르다.

이것은 한 영역 내에서 여러 뮤텍스를 다룰 때 유용하다. 범위 락을 사용하는 이유는 우연히 발생하는 데드락과 그 밖의 달갑지 않는 복잡함(예를 들어 한 뮤텍스는 범위 락에 의해 락돼 있고 또 다른 락은 여전히 대기 중인 상황에서 정확히 이와 반대되는 상황을 가지는 또 다른 스레드 인스턴스가 존재하는 경우)을 방지하기 위함이다.

범위 락의 한 속성은 이런 상황을 방지하고자 하는 것이다. 즉, 이론적으로는 이 유형의 락을 **데드락 안전**(deadlock-safe)한 것으로 만든다.



### 재귀 뮤텍스

재귀 뮤텍스(recursive mutex)는 뮤텍스의 또 다른 서브유형이다. 이것은 일반 뮤텍스와 동일한 기능을 갖지만 최초에 ==뮤텍스를 락시킨 호출 스레드로 하여금 동일한 뮤텍스를 반복해서 락 시킬 수 있게 해 준다.== 이렇게 함으로서 ==뮤텍스는 소유자 스레드가 이 뮤텍스를 락시킨 횟수만큼 언락시킬 때까지 다른 스레드에 의해 이용될 수 없게 된다.==

예를 들어 재귀 함수를 사용하는 경우 **재귀 뮤텍스**를 사용할 수 있다. 일반적인 뮤텍스를 사용한다면 재귀 함수에 진입하기 전에 뮤텍스를 락시킬 일종의 진입점을 만들어야 한다.


재귀 뮤텍스 덕분에 재귀 함수의 각 반복 호출 때 재귀 뮤텍스를 다시 락 시키고 하나의 반복이 종료될 때 뮤텍스를 언락한다. 결과적으로 뮤텍스는 동일한 횟수만큼, 락, 언락 된다.


여기서 ==**잠재적 복잡성**은 재귀 뮤텍스가 락될 수 있는 최대 횟수가 표준안에 정의돼 있지 않다는 것이다.== ==구현의 제한치에 도달한 상태에서 뮤텍스를 락시키고자 한다면 std::system_error가 던져지거나 비블록 try_lock함수를 사용할 때 false가 반환된다.==


### 재귀 타임드 뮤텍스

재귀 타임드 뮤텍스(recursive timed mutex)는 이름이 말하듯 타임드 뮤텍스와 재귀 뮤텍스의 기능을 합친 것이다. 결과적으로 타임드 조건 함수를 사용해 뮤텍스를 재귀적으로 락 시킬 수 있다.

이것은 스레드가 락시킨 횟수만큼 언락되는 것을 보장하기 위해 어려움이 따르지만 앞서 언급한 **태스크-핸들러와 같은 좀 더 복잡한 알고리즘에 대한 가능성을 제공**한다.

---
# 공유 뮤텍스

`<shared_mutex>` 헤더는 2014년 표준안에 shared_timed_mutex 클래스를 추가하면서 처음 도입되었다. 2017년 표준안에도 shared_mutex 클래스가 추가됐다.

공유 뮤텍스(shared mutex) 헤더는 C++17 이후로 존재했었다. ==일반적인 상호 배제 접근과 더불어 이 뮤텍스 클래스는 뮤텍스에 대한 공유 접근을 제공하는 기능을 가진다.== 예를 들어 이 기능을 사용하면 하나의 쓰기 스레드가 여전히 배타적 접근을 획득할 수 있으면서도 복수의 스레드가 그 자원을 읽을 수 있다.

이것은 Pthreads의 읽기 쓰기 락과 비슷하다.


이 뮤텍스 유형에 추가된 함수는 다음과 같다.

- lock_shared()
- try_lock_shared()
- unlock_shared()

이 뮤텍스의 공유 기능 사용은 꽤나 명료하다. ==하나의 스레드만이 해당 자원에 언제라도 쓰기를 하면서도 이론적으로 제한이 없는 읽기 스스레드가 뮤텍스에 대한 읽기 접근을 획득할 수 있다.==


### 공유 타임드 뮤텍스

이 헤더는 C++14 이래로 존재했다. 이것은 다음과 같음 함수를 통해 타임드 뮤텍스에 공유 락 기능을 제공한다.

- lock_shared()
- try_lock_shared()
- try_lock_shared_for()
- try_lock_shared_until()
- unlock_shared()

이 클래스는 ==그 이름이 암시하듯이 기본적으로 **공유 뮤텍스** 와 **타임스 뮤텍스**를 혼합한 것이다.== 흥미로운 점은 이것은 기본 공유 뮤텍스보다 먼저 표준에 추가됐다는 것이다.


---

# 조건 변수

조건 변수는 기본적으로 다른 스레드에 의해 스레드 실행을 제어할 수 있는 메커니즘을 제공한다. 이것은 다른 스레드에 의해 시그널될 때까지 대기하는 공유 변수에 의해 이뤄진다.

4장 [[스레드 동기화와 통신]]에서 살펴봤던 [[스레드 동기화와 통신#스케줄러|스케줄러 구현]]의 핵심 부분이다.

C++11 API의 경우, 조건 변수와 이들의 관련 기능은 `<condition_variable>`헤더에 정의돼 있다.

조건 변수의 기본 사용법은 4장, '스레드 동기화와 통신'에서 사용한 스케줄러 코드에 요약돼 있다.

``` c++

#pragma once
#ifndef WORKER_H
#define WORKER_H
  

#include "Abstract_Request.h"

#include <condition_variable>
#include <mutex>

using namespace std;

class Worker

{

condition_variable cv;
mutex mtx;
unique_lock<mutex> ulock;
AbstractRequest* request;
bool running;
bool ready;

public:
    Worker(){
        running=true;
        ready=false;
        ulock=unique_lock<mutex>(mtx);
    }

    void run();
    void stop(){running=false;}
    void setRequest(AbstractRequest* request){
        this->request=request;
        ready=true;
    }

    void getCondition(condition_variable* &cv);
};


#endif
```

Worker 클래스 선언에 정의된 생성자에서 C++11 API의 조건 변수가 초기화되는 방법을 볼 수 있다. 그 단계는 다음과 같다.


1. condition_variable과 mutex 인스턴스를 생성한다.
2. 뮤텍스를 새로운 unique_lock 인스턴스에 할당한다. 락을 위해 여기서 사용하는 생성자를 통해 할당된 뮤텍스는 할당 시에 락이 된다.
3. 조건 변수는 이제 사용할 준비가 됐다.

```c++
#include "Worker.h"
#include "disptcher.h"

#include <chrono>

  

using namespace std;

void Worker::getCondition(condition_variable * &cv){
    cv=&(this)->cv;
}

  

void Worker::run(){
    while(running){
        if(ready){
            ready=false;
            request->process();
            request->finish();
        }

        if(Dispatcher::addWorker(this)){
            // Use the ready loop to deal with spurious wake-ups.
            while(!ready&& running){
                if(cv.wait_for(ulock,chrono::seconds(1))==cv_status::timeout){
                    // We timed out, but we keep waiting unless
                    // the worker is
                    // stopped by the dispatcher.
                }
            }
        }
    }
}
```

여기서 조건 변수의 wait_for() 함수를 사용하는데 ==이때 앞서 생성한 고유 락 인스턴스와 대기하고자 하는 시간을 전달한다.== 1초 동안 대기한다. 이 대기에서 타임아웃이 발생하면 반복 루프에서 대기로 재진입하거나 실행을 계속한다.

wait() 함수를 사용하거나 wait_for()로 특정 시점까지 대기할 수도 있다.

이 코드를 처음 봤을 때 언급한 것처럼 작업자 코드가 ==ready 불리언 변수를 사용하는 이유는 조건 변수를 시그널한 스레드가 가짜 웨이크-업이 아닌 실제로 다른 스레드였음을 검사하기 위한 것이다.== 이런 상황에 취약한 것은 C++11의 구현을 포함해 대부분의 조건 변수 구현의 복잡성으로 인한 것이다.


이러한 **임의의 깨우기 이벤트**로 인해 실제로 의도하여 깨어났음을 보장할 수단이 필요하다. ==스케줄러 코드에서 이것은 작업자 스레드를 깨우는 스레드로 하여금 작업자 스레드가 깨어날 수 있는 불리언 값을 설정하게 함으로서 이뤄진다.==

타임아웃이 됐는지 또는 통지를 받았는지, 가짜 깨우기에 의한 것인지는 cv_status 열거 유형으로 검사할 수 있다. 이 열거 유형은 다음과 같은 가능한 두 조건을 인지한다.

- timeout
- no_timeout

시그널 또는 통지는 그 자체로 매우 명료하다.

``` c++
void Dispatcher::addRequest(AbstractRequest* request){

    workersMutex.lock();

    if(!workers.empty()){
        Worker* worker=workers.front();
        worker->setRequest(request);
        condition_variable* cv;
        worker->getCondition(cv);
        cv->notify_one();
        workers.pop();
        workersMutex.unlock();
    }else

    {
        workersMutex.unlock();
        requestsMutex.lock();
        requests.push(request);
        requestsMutex.unlock();
    }

}
```

Dispatcher 클래스의 이 함수에서 이용 가능한 작업 스레드 인스턴스를 얻기 위한 시도를 한다. ==발견을 하면 다음과 같이 작업자  스레드의 조건 변수에 대한 참조를 갖게 된다.==


``` c++
void Worker::getCondition(condition_variable * &cv){

    cv=&(this)->cv;

}
```

작업자 스레드에 새로운 요청을 설정하게 되면 ready 변수의 값도 true로 변경한다. 이를 통해 작업자는 실제로 계속 진행할 수 있는지를 검사할 수 있다.

마지막으로 조건 변수는 notify_one()을 사용해 자신을 대기하고 있는 스레드가 이제 계속 진행할 것인지에 대해 통지를 받는다. 이 특별한 함수는 이 조건 변수에 대한 FIFO큐 내의 첫 번째 스레드를 시그널하여 재개하도록 한다. 여기서는 단 하나의 스레드만이 통지를 받는다. 하지만 동일한 조건 변수를 복수의 스레드가 대기하고 있다면 notify_all()을 통해 FIFO 큐 내의 모든 스레드가 재개되도록 할 수 있다.


### Condition_variable_any

condition_variable_any 클래스는 condition_variable 클래스를 일반화하는 것이다. 이것은 `unique_lock<mutex>` 이상의 다른 상호 배제 메커니즘의 사용을 허용한다는 점에서 condition_variable 클래스와는 다르다. ==유일한 요건은 사용되는 락이 BasicLockable 요구사항을 충족시켜야 한다는 것이다.== 즉, lock()과 unlock() 함수를 제공해야 한다.


### 스레드 종료 시점에 모두에게 통지하기

std::notify_all_at_thread_exit() 함수를 이용해 (분리된) 한 ==스레드가 자신이 완전히 마쳤고 자신의 영역에 속한(스레드-로컬) 모든 객체가 해제되고 있음을 나머지 모든 스레드에게 통지할 수 있다.== 이것은 제공된 조건 변수를 시그널하기 이전에 제공된 락을 내부 저장소로 이동시키면서 동작한다.

(동작은 하지 않지만) 기본 예제는 다음과 같다.


``` c++
#include <mutex>
#include <thread>
#include <condition_variable>

using namespace std;

mutex m;
condition_variable cv;
bool ready=false;
ThreadLocal result;

  
void worker(){
    unique_lock<mutex> ulock(m);
    result=thread_local_method();
    ready=true;
    std::notify_all_at_thread_exit(cv,std::move(ulock));
}

  

int main(){

    thread t(worker);
    t.detach();

    // Do work here.

    unique_lock<std::mutex> ulock(m);

    while(!ready){
        cv.wait(ulock);
    }

    // Process result

}
```

여기서 작업자 스레드는 스레드-로컬 객체를 생성하는 메소드를 생성한다. ==따라서 주 스레드는 분리된 작업자 스레드가 먼저 종료하기를 대기해야 한다.== ==주 스레드가 작업을 끝냈을 때 분리된 작업자 스레드가 아직 완료되지 않았다면 주 스레드는 전역 조건 변수를 사용해 대기 상태로 진입한다.== 작업자 스레드에서 ready 불리언을 설정한 이후에 std::notify_all_at_thread_exit()가 호출된다.

여기서 이루어지는 것은 두 가지다. std::notify_all_at_thread_exit()를 호출한 이후에 어떤 스레드라도 조건 변수를 대기할 수 없다. 또한 주 스레드는 분리된 작업자 스레드의 결과가 이용 가능하게 될 때까지 대기할 수 있다.


---
# 퓨처

C++11 스레드 지원 API에 대한 마지막 부분은 `<future>`에 정의되어 있다. 이것은 멀티스레드 아키텍쳐에 대한 구현이라기보다는 ==용이한 비동기적 처리를 목표로 한 좀 더 고수준의 멀티스레딩 개념을 구현하는 일련의 클래스를 제공한다.==

여기서 퓨처(future)와 프라미스(promise)에 대한 두 개념을 구분해야 한다. 전자는 구독자/소비자에 의해 사용될 **최종 결과물**(미래의 산출물)이다. 후자는 **제작자/생산자가 사용하는 개념**이다.

퓨처의 예제는 다음과 같다.

``` c++
#include <iostream>
#include <future>
#include <chrono>

  

bool is_prime(int x){

    for (int i = 0; i < 2; ++i)
    {
if (x%i==0)
{return false;}
  }
    return true;
}

  

int main(){

    std::future<bool> fut=std::async(is_prime,444444443);
    std::cout<<"Checking, please wait";
    std::chrono::milliseconds span(100);

    while (fut.wait_for(span)==std::future_status::timeout)
    {
        std::cout<<'.'<<std::flush;

    }

  

    bool x=fut.get();

    std::cout<<"\n444444443"<<(x?"is":"is not")<<"prime.\n";

    return 0;

}
```

==이 코드는 비동기적으로 한 함수를 호출하면서 인자(잠재적인 소수)를 전달한다.== 그러나 나서 비동기 함수 호출로부터 받은 퓨처가 종료하기를 대기하면서 활성 루프로 진입한다. 대기 함수에는 100ms 타임아웃이 설정돼 있다.

퓨처가 종료되면(대기 함수에서 타임아웃으로 반환하지 않고서) 결괏값을 구한다. 이 경우 함수에 제공했던 값이 실제로는 소수임을 알려준다.

5장의 [[네이티브 C++ 스레드와 기본 요소#async]] 절에서 비동기 함수 호출에 대해 더 살펴볼 예정이다.

### 프라미스

프라미스(promise)는 스레드 간의 상태 전송을 가능하게 해준다. 다음은 그 예제다.

``` c++

#include <iostream>
#include <functional>
#include <thread>
#include <future>

  

void print_int(std::future<int>& fut){
    int x=fut.get();
    std::cout<<"value: "<<x<<std::endl;
}

  

int main(){

    std::promise<int> prom;
    std::future<int> fut=prom.get_future();
    std::thread th1(print_int,std::ref(fut));
    prom.set_value(10);
    th1.join();
    return 0;
}
```

이 코드에서 ==다른 스레드에 값(여기서는 정수)을 전송하기 위해 작업자 스레드에 전달되는 **프라미스 인스턴스**를 사용==한다. 새로운 스레드는 프라미스에서 생성한 **퓨처**(작업을 완료하기 위해 주 스레드로부터 받은)를 대기한다.

프라미스에 값을 설정할 때 프라미스는 완료된다. 이것은 퓨처를 완료시키고 작업자 스레드는 종료하게 된다.

이 특별한 예제에서 퓨처 객체에 대한 **블록 대기**(blocking wait)를 사용했다. 하지만 퓨처에 관한 이전 예제에서 살펴본 것처럼 일정시간 동안이나 일정 시점까지 대기하기 위해 각각각 wait_for()이나 또는 wait_until()을 사용할 수도 있다.


### 공유 퓨처

shared_future는 일반 퓨처 객체와 유사하지만 복사가 될 수 있어서 복수의 스레드가 그 결과를 읽을 수 있다.

shared_future는 일반 퓨처와 유사한 방법으로 생성할 수 있다.

``` c++
std::promise<void> promise1;
std::shared_future<void> sFuture(promise1.get_future());
```

가장 큰 차이점은 일반 퓨처가 그 생성자로 전달된다는 점에 있다.

이후로 ==퓨처 객체에 접근하는 모든 스레드는 이를 대기할 수 있고 그 값을 구할 수 있다.==
또한 이것은 조건 변수와 유사한 방식으로 스레드를 시그널하는 데 사용될 수 있다.


### Packaged_task

packaged_task는 호출 가능한 대상(함수나 바인드, 람다, 그 밖의 함수 객체)에 대한 래퍼다. 이것은 퓨처 객체에서 사용할 수 있는 결과로 비동기 실행이 가능하게 한다. 이것은 std::function와 유사하지만 그 결과를 퓨처 객체에 자동으로 전달한다.

다음은 그 예제다.

``` c++
#include <iostream>
#include <future>
#include <chrono>
#include <thread>

  

using namespace std;

  

int countdown(int from,int to){

    for (int i = from; i < to; --i)
    {
        cout<<i<<endl;
        this_thread::sleep_for(chrono::seconds(1));
    }
    cout<<"Finished countdown.\n";
    return from -to;
}

  

int main(){

    packaged_task<int(int,int)> task(countdown);
    future<int> result=task.get_future(); // countdown의 반환값 그래서 future 탬플릿이 int 이다.
    thread t(std::move(task),10,0);
    // Other logic.
    int value=result.get();

  

    cout<<"The countdown lasted for "<<value<<"seconds. \n";
    t.join();
    return 0;
}
```

이 코드는 10부터 0까지 카운트다운하는 간단한 기능을 구현한다. ==태스크를 생성하고 그 퓨처 객체에 대한 참조를 구한 이후에 작업자 함수에 대한 인자와 태스크를 이용해 스레드 인스턴스를 생성한다.==

==카운트다운 작업자 스레드의 결과는 종료 즉시 이용 가능하다.== 여기서 프라미스의 경우와 동일한 방식으로 퓨처 객체의 대기 함수를 사용할 수 있다.


### Async

프라미스와 packaged_task의 좀 더 직관적인 버전은 std::async()에서 볼 수 있다. 이것은 호출 객체(함수와 바인드bind, 람다lambda)와 그 밖의 다른 인자를 가지며 퓨처 객체를 반환하는 간단한 함수다.

다음은 async()함수의 기본 예제다.


``` c++
#include <iostream>
#include <future>

using namespace std;

  

bool is_prime(int x){

    cout<<"Calculating prime ..."<<endl;
    for (int i = 2; i < x; ++i)
    {
        if(x%i==0){
            return false;
        }
    }
    return true;
}

  

int main(){

    future<bool> pFuture=std::async(is_prime,343321);
    cout<<"Checking whether 343321 is a prime number. \n";

    // wait for future object to be ready.

    bool result=pFuture.get();

    if (result)
    {
        cout<<"Prime found."<<endl;
    }else
    {
        cout<<"No prime found."<<endl;
    }
    return 0;

}
```

이 코드에서 worker 함수는 제공된 정수가 소수인지 아닌지 여부를 결정한다. 보다시피 최종 코드는 packaged_task나 프라미스의 경우보다 훨씬 단순해졌다.


### 시작 정책

std::async()의 기본 버전 외에 첫 번째 인자로 **시작 정책**(launch policy)을 지정할 수 있는 또 다른 버전이 존재한다. 이것은 다음과 같은 값을 가질 수 있는 std::launch 유형의 비트마스크 값이다.

``` c++
*launch::async
*launch::deferred
```

async 플래그는 **새로운 스레드**와 작업자 **함수의 실행 컨텍스트**가 즉시 생성됨을 의미한다. deferred 플래그는 퓨처 객체에 대해 wait() 또는 get()이 호출될 대까지 지연됨을 의미한다. 이들 ==두 플래그를 함께 지정하면 함수는 현재 시스템 상황에 따라 자동으로 해당 메소드를 선택한다.==

명시적으로 비트마스크 값을 지정하지 않은 std::async()는 기본으로 후자가 된다.


---
### 원자적 요소

멀티 스레딩에 있어서 원자적 요소의 사용 또한 매우 중요하다. C++11 STL은 이런 이유로 `<atomic>`헤더를 제공한다. 이 주제는 8장 '원자적 동작-하드웨어와 작업하기'에서 심도 있게 다룬다.