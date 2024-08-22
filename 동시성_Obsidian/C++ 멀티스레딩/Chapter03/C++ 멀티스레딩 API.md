
C++는 표준 템플릿 라이브러리(STL, Standard Template Library)에 네이티브 멀티스레딩 구현을 가지고 있지만 OS 수준과 프레임워크 기반의 멀티스레딩 API가 여전히 매우 일반적이다. 이들 API의 예로는 윈도우와 이식형 운영체제 인터페이스(POSIX,Portable Operating System Interface) 스레드 그리고 Qt, Boost, POCO 라이브러리에 의해 제공되는 것들이 있다.


3장에서 다루려는 주제는 다음과 같다.


- 이용 가능한 멀티스레딩 API 비교
- 이들 API 각각에 대한 사용 예제



---

### API 개요
[[POSIX 스레드]](Pthread)는 실질적으로 리눅스 기반과 BSD 기반의 OS, OS  X(macOS), 솔라리스 같은 UNIX 계열 OS의 표준이 되었다.


플랫폼 간의 개발을 용이하게 하기 위해 다수의 라이브러리가 개발됐다. Pthreads는 소프트웨어를 모든 중요 운영체제에 이식 가능하도록 하는데 UNIX 계열의 OS를 필요한 전제 조건 중의 하나로 만들기 위해 다소간의 도움을 주기 하지만 일반적인 스레딩 API가 필요하다. 이것이 Boost와 POCO ,Qt와 같은 라이브러리가 만들어진 이유이다.


---

### [[POSIX 스레드]]


---

### 윈도우 지원


다음과 같이 POSIX API를 제한된 방식으로 사용할 수 있다.


| 이름                 | 준수                                                                                              |
| ------------------ | ----------------------------------------------------------------------------------------------- |
| Cygwin             | 거의 완료됨. 일반 윈도우 애프리케이션으로 배포될 수 있는 POSIX 애플리케이션을 위한 완전한 런타임 환경을 제공한다.                             |
| MinGW              | MinGW-w64(MinGW의 재개발 버전)의 경우, Pthreads 지원은 거의 완료됬지만 일부 기능을 빠져 있을 수 있다.                          |
| 리눅스 용도의 윈도우 서브 시스템 | WSL은 윈도우 10의 기능이다. 이 기능을 사용하려면 현재 위도우 10 애니버서리 업데이트를 실행하고 마이크로소프트가 제공하는 지침을 따라 직접 WSL을 설치해야 한다. |



윈도우에서 POSIX는 일반적으로 권장되지 않는다. POSIX를 사용할 합당한 이유(예를 들어 커다란 기존의 코드 기반)가 없다면 플랫폼 문제를 해결할 수 있는 크로스-플랫폼 API(3장 뒷부분에서 설명)를 사용하는 것이 훨씬 용이하다.


다음 절에서 Pthreads API가 제공하는 기능을 살펴본다.

> [!note]
> 여기서 pthread.h를 메이커파일에서 실행시키기 위해 속성> VC++로 들어감. 그 후에 외부 include 디렉터리를 설정, 다음에 라이브러리 디렉터리를 수정하여 lib 파일과 연결함.

---

### Pthreads 스레드 관리

pthread_ 또는 pthread_attr_로 시작하는 모든 함수가 여기해 해당한다. 이들 함수 모두는 스레드 자체와 스레드 속성 객체에 적용된다.

``` C++

#include <pthread.h>
#include <stdlib.h>
#define NUM_THREADS 5

```


가장 중요한  Pthreads 헤더는 pthread.h이다. 이것은 세마포어를 제외한 모든 것에 대한 접근을 제공한다. 여기서 시작하고자 하는 스레드의 개수를 상수로 정의한다.

``` c++
void* worker(void* arg) {
	int value = *((int*)arg);
	// More business logic.

	return 0;
}

```

간단한 Worker 함수를 정의하고 이 함수를 잠시 후에 새로운 스레드로 전달할 것이다. 새로운 스레드로 전달된 값을 출력하기 위해 예시와 디버깅 용도로 비즈니스 로직 부분에 cout 또는 printf 기반의 작업을 추가할 수 있다.

그러고 나서 다음과 같이 main 함수를 정의한다.


``` c++
int main(int argc, char** argv) {
	pthread_t threads[NUM_THREADS];
	int thread_args[NUM_THREADS];

	int result_code;

	for (unsigned int i = 0; i < NUM_THREADS; i++)
	{
		thread_args[i] = i;
		result_code = pthread_create(&threads[i], 0, worker, (void*)&thread_args[i]);
	}
}
```


main 함수 내의 루프에서 모든 스레드를 생성한다. 각 스레드 인스턴스는 생성될 때 할당된 스레드 ID(첫 번째 인자)와 ==pthread_create() 함수가 반환하는 결과 코드(성공 시 0)를 갖는다.== 스레드 ID는 이후의 호출에서 스레드를 참조하기 위한 핸들이다.

pthread_create 함수의 두 번째 인자는 pthread_attr_t 구조체 인스턴스이거나, 없으면 0이다. ==이것은 초기 스택 크기와 같이 새로운 스레드의 특성 구성을 가능하게 한다.== 0이 전달되면 플랫폼과 구성별로 다른 기본 인자가 된다.

pthread_create 함수의 세 번째 인자는 새로운 스레드가 시작할 함수에 대한 포인터다. 이 함수 포인터는 void 데이터 포인터(즉, 커스텀 데이터)를 반환하고 void 데이터 포인터를 받는 함수로 정의돼 있다. ==다음과 같이 인자에 의해 새로운 스레드로 전달되는 데이터는 스레드ID이다.==


``` c++
	for (int i = 0; i < NUM_THREADS; i++)
	{
		result_code = pthread_join(threads[i], 0);
	}
	exit(0);
```

이제 pthread_join() 함수를 통해 각 작업자 스레드가 마치기를 대기한다. 이 함수는 두 인자를 가지는 대기할 스레드의 ID와 worker 함수의 반환값을 위한 버퍼(또는 0)가 그것이다.


스레드를 관리하는 그 밖의 함수로는 다음과 같은 것이 있다.


---


스레드는 관리하는 그 밖의 함수로는 다음과 같은 것이 있다.

- `void pthread_exit(void *value_ptr)`

이 함수는 호출하는 스레드를 종료시키고 제공된 인자의 값을 pthread_join()를 호출하는 모든 스레드가 사용할 수 있도록 한다.


- `int pthread_cancel(pthread_t thread)`

명시된 스레드가 취소되도록 요청한다. 대상 스레드의 상태에 따라 이 함수는 자신의 취소 핸들러를 호출한다.


이외에도 `pthread_attr_t` 구조체에 관한 정보를 조작하고 구하기 위한 `pthread_attr_*` 함수들이 존재한다.


---

### 뮤텍스

이들 함수에는 `pthread_mutex_` 또는 `pthread_mutexattr_` 접두어가 붙는다. 이들은 뮤텍스와 그 속성 객체에 작동한다.

Ptheads에서 뮤텍스는 초기화되고, 해제, 락, 언락이 이루어진다. 이들은 `pthread_mutexattr_t` 구조체를 사용해 커스텀화된 동작을 한다. 이 구조체는 또한 자신에 대한 속성을 초기화하고 해제하는 해당 `pthread_mutexattr_*` 함수를 가진다.


정적 초기화를 사용하는 Pthread 뮤텍스의 기본적 사용법은 다음과 같다.


``` c
static pthread_mutex_t func_mutex = PTHREAD_MUTEX_INITIALIZER;


void func() {
	pthread_mutex_lock(&func_mutex);

	// Do something that's not thread-safe.

	pthread_mutex_unlock(&func_mutex);
}
```

이 코드에서 PTHREAD_MUTEX_INITIALIZER 매크로를 사용해 ==메번 뮤텍스에 대한 코드를 입력하지 않고서도 뮤텍스를 초기화한다.== 매크로의 사용이 도움이 되긴 하지만 다른 API와 비교해 수동으로 뮤텍스를 초기화하고 해제해야 한다.

초기화 코드 다음은 뮤텍스를 락하고 언락하는 부분이다. 일반적인 락 버전과 유사한 ***pthread_mutex_trylock()*** 이 존재하지만 이 함수는 참조 뮤텍스가 이미 락 되어 있다면 언락되기를 대기하는 대신 즉시 리턴한다.


이 예제에서 뮤텍스는 명시적으로 해제되지 않는다. 이것은 Pthreads 기반의 애플리케이션에서 일반적인 메모리 관리의 일부분이다.



---

### 조건 변수


조건 변수 `pthread_cond_` 또는 `pthread_condattr_` 접두어가 붙은 함수들이다. 이들 함수는 조건 변수와 그 속성 객체에 작동한다.


Pthreads에서 조건 변수는 초기화와 해제 함수를 가지는 경우와 동일한 패턴을 따르며 또한 pthread_condattr_t 속성 구조체를 관리하는 것과 동일한 패턴을 따른다.


이 예제는 Pthreads 조건 변수의 기본적 사용법을 다룬다.


``` c
#include <pthread.h>
#include <stdlib.h>
#include <unistd.h>


#define COUNT_TIRGGER 10
#define COUNT_LIMIT 12
  
int count=0;

int thread_ids[3]={0,1,2};

pthread_mutex_t count_mutex;

pthread_cond_t count_cv;
```

이전 코드에는 표준 헤더 파일이 보이고 그 용도는 잠시 뒤 보게 될 COUNT_TRIGGER와 COUNT_LIMIT가 정의돼 있다. count 변수와 생성하고자 하는 스레드 ID, 뮤텍스와 조건 변수 같은 전역 변수도 정의돼 있다.

``` c++

void* add_count(void* t){
    int tid=(long)t;
    for (int i = 0; i < COUNT_TIRGGER; i++)
    {
        pthread_mutex_lock(&count_mutex);
        count++;

if(count==COUNT_TIRGGER){

    pthread_cond_signal(&count_cv);

}
pthread_mutex_unlock(&count_mutex);
sleep(1);
    }
    pthread_exit(0);

}

```

이 함수는 기본적으로 count_mutex로서 전역 타운터 변수에 대한 배타적 접근을 획득한 후 이 변수의 값을 증가시킨다.

이 함수는 실행하는 두 번째 스레드에게 뮤텍스를 획득할 기회를 주기 위해 루프를 돌 때 1초마다 슬립한다.


``` C++
void* watch_count(void* t){

    int tid=(int)t;
    pthread_mutex_lock(&count_mutex);
    if(count<COUNT_LIMIT){
        pthread_cond_wait(&count_cv,&count_mutex);
    }
    pthread_mutex_unlock(&count_mutex);

    pthread_exit(0);
}
```

==두 번째 이 함수에서는 카운트 제한에 도달했는지 검사하기 전에 전역 뮤텍스를 락 시킨다.== 이것은 카운트가 제한 값이 도달하기 전에 이 함수를 실행하는 스레드가 호출되지 않는 경우를 대비한 것이다.


그렇지 않다면 조건 변수와 락된 뮤텍스를 제공하는 조건 변수를 대기한다. 시그널이 되면 전역 뮤텍스를 언락시키고 스레드는 종료한다.


여기서 주목해야 할 점은 이 예제는 가짜 깨우기(wake-ups)를 고려하지 않았다는 것이다. Pthreads 조건 변수는 루프를 통해 어떤 종류의 조건이 충족이 됐는지 검사하는 데 필요한 깨우기 작업에 민감하다.


``` c
int main(int argc,char* argv[]){
    int tid1=1,tid2=2,tid3=3;
    pthread_t threads[3];
    pthread_attr_t attr;
    
    pthread_mutex_init(&count_mutex,0);
    pthread_cond_init(&count_cv,0);

    pthread_attr_init(&attr);

    pthread_attr_setdetachstate(&attr,PTHREAD_CREATE_JOINABLE);

    pthread_create(&threads[0],&attr,watch_count,(void*)tid1);

    pthread_create(&threads[1],&attr,add_count,(void*)tid2);

    pthread_create(&threads[2],&attr,add_count,(void*)tid3);

  

for (int i = 0; i < 3; ++i)
{
    pthread_join(threads[i],0);     // 마무리
}

  

pthread_attr_destroy(&attr);
pthread_mutex_destroy(&count_mutex);
pthread_cond_destroy(&count_cv);

return 0;

}
```


마지막으로 main 함수에서 3개의 스레드를 생성한다. 이들 중 두 스레드는 카운터를 증가시키는 함수를 실행하고 나머지 한==스레드는 조건 변수가 시그널되기를 대기하는 함수를 실행한다.==


이 메소드에서 또한 전역 뮤텍스와 조건 변수를 초기화한다. 생성되는 스레드는 "합류 가능(joinable)" 속성이 명시적으로 설정된다.

끝으로 각각의 스레드가 마치기를 대기한다. 그리고 종료 전에 속성 구조체 인스턴스와 뮤텍스, 조건 변수를 해제해야 정리 작업을 수행한다.


***pthread_cond_broadcast()*** 함수를 사용하면 큐에 있는 첫 번째 스레드만 시그널 시키는 것이 아니라==조건 변수를 대기하는 모든 스레드를 시그널시킬 수 있다.== 이를 통해 새로운 데이터 설정을 대기하는 작업자 스레드가 많은 경우 개별 스레드별로 통지하지 않아도 되는 방식으로 일부 애플리케이션에는 좀 더 세련되게 조건 변수를 사용할 수 있다.


---

### 동기화

==동기화를 구현하는 함수==는 `pthread_rwlock_` 또는 `pthread_barrier_` 접두어를 가진다. 이들은 읽기-쓰기 락과 동기화 장벽(synchronization barriers)을 구현한다.

읽기-쓰기 락(rwlock)은 스레드 수에 제한 없이 동시에 읽을 수 있고 쓰기 접근은 한 스레드로 접근을 제한한다는 점을 제외하면 뮤텍스와 유사하다.

``` c++
#include <pthread.h>
int pthread_rwlock_init(pthread_rwlock_t* rwlock,const pthread_rwlock_t* attr);
pthread_rwlock_t rwlock=PTHREAD_RWLOCK_INITIALIZER;
```

이전 코드에서 동일한 헤더 파일이 보이고 초기화 함수와 일반적인 매크로를 볼 수 있다. 흥미로운 부분은 rwlock을 락시킬 때 읽기 접근 용도로 이루어진다는 것이다.

``` c++

int pthread_rwlock_rdlock(pthread_rwlock_t* rwlock);

int pthread_rwlock_tryrdlock(pthread_rwlock_t* rwlock);
```


여기서 두 번째 다른 점은 락이 이미 락돼 있다면 즉시 복귀한다는 것이다. 쓰기 접근을 위해 락을 락시킬 수 도 있다.

``` c++

int pthread_rwlock_wrlock(pthread_rwlock_t* rwlock);
int pthread_rwlock_trywrlock(pthread_rwlock_t* rwlock);
```

복수의 리더(reader)가 읽기 전용 락을 획득할 수 있을지라도 단 하나의 라이터(writer)가 특정 시점에 허용된다는 점을 제외하면 이들 함수는 기본적으로 동일하게 동작한다.

장벽(Barriers)은 Pthreads의 또 다른 개념이다. 이들은 다수의 스레드에 대해 장벽처럼 동작하는 동기화 객체다. 이들 모든 스레드는 장벽을 지나 계속 진행하면 장벽에 도달해야 한다. 장벽 초기화 함수에서 스레드 카운트가 명시된다. 이들 모든 스레드가 pthread_barrier_wait() 함수를 사용해 장벽 객체를 호출해야만 실행을 계속할 수 있다.


---

### 세마포어

앞서 언급했듯이 세마포어(semaphores)는 POSIX 사양에서 원래 Pthreads 확장의 일부분이 아니었다.

본질적으로 세마포어는 자원 카운트로 사용되는 단순한 정수다. 세마포어를 이용해 스레드 안전(thread-safe)하게 만들기 위해 원자적 동작(검사와 락)이 사용된다. POSIX 세마포어는 세마포어의 초기화와 해제, 증가 감소를 지원하여 세마포어가 논-제로 값에 도달하기를 대기하는 것도 지원한다.


---

### 스레드 로컬 스토리지

Pthreads처럼 스레드 로컬 스토리지(TLS,Thread Local Storage)는 스레드 한정적인 데이터를 설정하기 위해 키와 메소드를 사용 작업을 수행한다.


```c
pthread_key_t global_var_key;
void* worker(void* arg){
    int *p=new int;
    *p=1;

    pthread_setspecific(global_var_key,p);
    int* global_spec_var=(int*)pthread_getspecific(global_var_key);
    *global_spec_var+=1;

    pthread_setspecific(global_var_key,0);
    delete p;
    pthread_exit(0);
}
```


작업자 스레드에서 힙에 새로운 정수를 할당하고 그 값으로 전역 키를 설정한다. 다른 스레드가 무엇을 하든가에 관계없이 전역 변수를 1씩 증가시켜서 그 값은 2가 된다. 이 스레드에 대해 작업을 완료했으면 전역 변수를 0으로 설정하고 할당된 변수를 삭제한다.

``` c
int main(void){
    pthread_t threads[5];
    pthread_key_create(&global_var_key,0);

    for (int i = 0; i < 5; i++)
    {
        pthread_create(&threads[i],0,worker,0);
    }

  

    for (int i = 0; i < 5; i++)
    {
        pthread_join(threads[i],0);
    }

    return 0;

}
```

전역 키가 설정돼 TLS 변수를 참조하는데 사용되지만 생성한 각 스레드는 이 키에 대해 자체 값을 설정할 수 있다.

스레드는 자체적인 키를 생성할 수 있지만, TLS를 처리하는 이 메소드는 3장에서 살펴보는 API와 비교할 때 꽤나 복잡하다.


---

### 윈도우 스레드

Pthreads에 비해 윈도우 스레드는 윈도우 운영체제와 그 유사한 운영체제에 제한적인다. 윈도우 스레드는 이를 지원하는 윈도우 버전에 의해 손쉽게 정의된 꽤나 일관적인 구현을 제공한다.

윈도우 비스타 이전에는 Pthreads에는 없는 기능을 가지고 있었지만 조건 변수 같은 기능이 스레딩 자원에서 빠져 있었다. 관점에 따라 윈도우 헤더에 정의된 무수한 "typedef" 유형을 사용해야 한다는 것은 꽤나 성가신 일이다.



##### 스레드 관리

공식적인 MSDN 문서의 예제 코드에서 발췌한 윈도우 스레드 사용에 관한 기본 예는 다음과 같다.

``` c
#include <windows.h>
#include <tchar.h>
#include <strsafe.h>
  

#define MAX_THREADS 3
#define BUF_SIZE 255
```


스레드 함수와 문자열 등과 관련된 윈도우 한정적인 헤더 파일을 포함하고 생성하고자 하는 스레드의 개수와 Worker 함수 내의 메시지 버퍼 크기를 정의한다.


각 작업자 스레드에 전달할 간단한 데이터를 포함하는 struct 유형(void 포인터(LPVOID)로 전달되는)을 또한 정의한다.

``` c
typedef struct MyData{
    int val1;
    int val2;
}MYDATA, *PMYDATA;

  
  

DWORD WINAPI worker(LPVOID lpParam){
    HANDLE hStdout=GetStdHandle(STD_OUTPUT_HANDLE);
    if(hStdout==INVALID_HANDLE_VALUE){
       return 1;
    }

  

    PMYDATA pDataArray=(PMYDATA) lpParam;

  

TCHAR msgBuf[BUF_SIZE];
size_t cchStringSize;
DWORD dwChars;

  
StringCchPrintf(msgBuf,BUF_SIZE,TEXT("Parameters=%d %dn"),pDataArray->val1,pDataArray->val2);
StringCchLength(msgBuf,BUF_SIZE,&cchStringSize);
WriteConsole(hStdout,msgBuf,(DWORD)cchStringSize,&dwChars,NULL);


return 0;

}
```

Worker 함수에서 제공된 인자를 사용하기 전에 우리 용도의 struct 유형으로 형변환을 하고서 그 값을 문자열로 콘솔에 출력된다.

활성 표준 출력(콘솔이나 그 유사한 것)이 존재한다는 것을 확인한다. 문자열에 출력에 사용된 함수는 모두 스레드 안전하다.


``` c
  

void errorHandler(LPTSTR lpszFunction){

    LPVOID lPMsgBuf;
    LPVOID lPDisplayBuf;
    DWORD dw=GetLastError();

    FormatMessage(
        FORMAT_MESSAGE_ALLOCATE_BUFFER|
        FORMAT_MESSAGE_FROM_SYSTEM|
        FORMAT_MESSAGE_IGNORE_INSERTS,
        NULL,
        dw,
        MAKELANGID(LANG_NEUTRAL,SUBLANG_DEFAULT),
        (LPTSTR)&lPMsgBuf,
        0,NULL
    );

  

    lPDisplayBuf=(LPVOID)LocalAlloc(LMEM_ZEROINIT,(lstrlen((LPCTSTR)lPMsgBuf)+lstrlen((LPCTSTR)lpszFunction)+40)*sizeof(TCHAR));

  

    StringCchPrintf((LPTSTR)lPDisplayBuf,LocalSize(lPDisplayBuf)/sizeof(TCHAR),TEXT("%s failed with error %d: %s"),lpszFunction,
    dw,lPMsgBuf);

  

    MessageBox(NULL,(LPCTSTR)lPDisplayBuf,TEXT("Error"),MB_OK);

  

    LocalFree(lPMsgBuf);
    LocalFree(lPDisplayBuf);

}
```

이 코드는 마지막 오류 코드에 대한 시스템 오류 메시지를 구하는 오류 핸들 함수를 정의한다. 마지막 오류에 대한 코드를 구한 이후에 메시지 박스로 통해 보여질 출력 오류 메시지를 포맷화한다. 마지막으로 할당된 메모리 버퍼가 해제된다.


끝으로 메인 함수는 다음과 같다.

``` c

int _tmain(){
    PMYDATA pDataArray[MAX_THREADS];
    DWORD dwThreadIdArray[MAX_THREADS];
    HANDLE hThreadArray[MAX_THREADS];

    for (int i = 0; i < MAX_THREADS; i++)
    {
       pDataArray[i]=(PMYDATA)HeapAlloc(GetProcessHeap(),HEAP_ZERO_MEMORY,sizeof(MYDATA));

       if(pDataArray[i]==0){
        ExitProcess(2);
       }

    pDataArray[i]->val1=i;
    pDataArray[i]->val2=i+100;

    hThreadArray[i]=CreateThread(
        NULL,
        0,
        worker,
        pDataArray[i],
        0,
        &dwThreadIdArray[i]
    );

if(hThreadArray[i]==0){
    errorHandler(TEXT("CreateThread"));
    ExitProcess(3);
}

    }

  

WaitForMultipleObjects(MAX_THREADS,hThreadArray,TRUE,INFINITE);

for (int i = 0; i < MAX_THREADS; i++)
{
    CloseHandle(hThreadArray[i]);
    if(pDataArray[i]!=0){
        HeapFree(GetProcessHeap(),0,pDataArray[i]);
    }
   

}

  
 return 0;  

}
```


main 함수에서 루프 내에서 스레드를 생성하고 스레드 데이터 용도의 메모리를 할당하고, 스레드를 시작하기 전에 각 스레드에 고유한 데이터를 생성한다. 각 스레드 인스턴스에는 고유한 인자가 전달된다.

그러고 나서 스레드가 종료되고 재합류하기를 대기한다. 이것은 Pthreads에서 단일 스레드에 대한 join 함수를 호출하는 것과 기본적으로 동일하다. 여기서는 단지 한 함수의 호출만으로도 충분하다.

최종적으로 각 스레드의 핸들이 닫히고 이전에 할당된 메모리를 정리한다.


---
##### 고급 관리

윈도우 스레드에 있어 고급 스레드 관리는 잡(job)과 파이버(fiber), 스레드 풀을 포함한다. 잡은 기본적으로 다수의 스레드를 단일 단위로 묶어서 이들 모든 스레드의 속성과 상태를 한 번에 변경할 수 있게 해 준다.

파이버는 가벼운 스레드로 자신들을 생성한 스레드 컨텍스트 내에서 실행한다. 생성 스레드가 이들 파이버 자체를 스케줄한다. 파이버는 TLS와 유사한 파이버 로컬 스토리지(FLS,Fiber Local Storage)를 가진다.


윈도우 스레드 API는 스레드 풀 API를 제공해 애플리케이션에서 이런 스레드 풀을 손쉽게 사용할 수 있게 해 준다. 각 프로세스에는 또한 기본 스레드 풀이 제공된다.


---

##### 동기화

윈도우 스레드에서 상호 배체와 동기화는 임계 영역과 뮤텍스, 세마포어, 슬림 읽기-쓰기(SRW), 장벽 등에 의해 이루어진다.

동기화 객체는 다음 표를 참고하라.


| 이름                      | 설명                                                                          |
| ----------------------- | --------------------------------------------------------------------------- |
| 이벤트                     | 네임드 객체를 사용해 스레드와 프로세스 간의 이벤트를 시그널한다.                                        |
| 뮤텍스                     | 공유 자원에 대한 접근을 조율하기 위한 스레드와 프로세스 간의 동기화 용도로 사용된다.                            |
| 세마포어                    | 스레드와 프로세스 간의 동기화 용도로 사용되는 표준 세마포어 카운터 객체                                    |
| 대기 기능 타이머               | 다양한 사용 모드에서 복수의 프로세스가 사용할 수 있는 타이머 객체                                       |
| 임계 영역                   | 임계 영역은 기본적으로 단일 프로세스에 제한적인 뮤텍스로서 커널 공간으로의 호출이 없음으로 인해 그 사용 속도가 빠르다.         |
| 슬림(slim) 읽기-쓰기 락        | SRW는 Phtreads의 읽기-쓰기 락과 유사하면 복수의 읽기 스레드 또는 단일 쓰기 스레드에게 공유 자원 접근을 허용한다.      |
| 인터락드(Interlocked) 변수 접그 | 어떤 변수 범위에 대해 원자적 접근을 가능하게 한다. 이를 이용해 스레드는 뮤텍스를 사용하지 않고서도 한 변수를 공유할 수 있게 한다. |

---
##### 조건 변수

윈도우 스레드에서 조건 변수의 구현은 매우 간단핟. 특정 조건 변수를 대기하거나 이를 시그널하기 위해 조건 변수 함수와 함께 임계 영역(CRITICAL_SECITON)과 조건 변수 (CONDITION_VARIABLE)를 사용한다.


---

##### 스레드 로컬 스토리지

윈도우 스레드에서 스레드 로컬 스토리지(TLS)는 중심 키(TLS 인덱스)를 먼저 생성해야만 각 스레드가 전역 인덱스를 사용해 로컬 값을 받아들여서 저장할 수 있다는 점에서 Pthreads와 유사하다.

Pthreads처럼 TLS는 TLS 값을 수동으로 할당하고 삭제하는 것과 같이 일정 부분 수동 메모리 관리를 해야 한다.


---
### 상승

상승(boost) 스레드는 상승 라이브러리 모음의 비교적 작은 일부분이다. 하지만 이것은 그 밖의 상승 라이브러리가 궁극적으로 완전하게 또는 일부분으로 새로운 C++표준이 된 것과 유사하게 C++11에서 멀티스레딩 구현의 기본으로 사용된다. 멀티스레딩 API에 관한 세부적 사항은 3장의 'C++ 스레드'절을 참조한다.

상승 스레드에서는 이용 가능하지만 C++11 표준에 없는 기능을 다음과 같다.

- (윈도우 잡과 같은) 스레드 그룹
- 스레드 인터럽션(취소)
- 타임아웃이 있는 스레드 합류
- (C++14에서 개선된) 추가적인 상호 배제 락 유형


이런 기능이 절대적으로 필요하지 않거나(STL 스레드를 포함해) C++11 표준을 지원하는 컴파일러를 사용하지 않는다면 c++11 구현에 **상승 스레드**를 사용할 이유가 거의 없다.

**상승**은 네이티브 OS 기능에 래퍼(wrapper)를 제공하기 때문에 네이티브 C++스레드를 사용하면 STL 구현의 품질 정도에 따라 오버헤드를 줄일 수 있다.


---
### Qt

Qt는 상대적으로 고수준의 프레임워크로 멀티스레딩 API에서도 반영돼 있다. Qt의 또 다른 특징은 **메타-컴파일러**(qmake)를 이용해 자신의 코드를 감싸서 시그널-슬록 아키텍처와 프레임워크의 다른 정의 기능을 구현한다는 것이다.

결과적으로 Qt 스레딩 지원은 기존의 코드에 그대로 추가될 수 없고 프레임워크에 맞게 코드에 수정해야 한다.



##### Qthread

Qt의 Qthread 클래스는 스레드가 아니고 스레드 인스턴스 주변을 감싸는 광범위한 래퍼로서 시그널-슬롯 통신과 런타임 지원, 그 밖의 기능을 추가한다. 이것은 다음 코드에서 보듯이 Qthread의 기본 방법에 반영돼 있다.

``` c++
class Worker: public QObject{
Q_OBJECT
public:
	Worker();
	~Worker();

public slots:
	void process();

signals:
	void finished();
	void error(QString err);

privateL
}
```


(Qt는 문제가 있어서 잠시 보류하겠다.)


---
### POCO

POCO 라이브러리는 운영체제 주변을 감싸는 꽤 가벼운 래퍼다. 이는 C++11 호환 컴파일러나 어떤 종류의 선행-컴파일(pre-compiling), 메타 컴파일(metacompiling)을 필요로 하지 않는다.


##### Thread 클래스


Thread 클래스는 OS 수준의 스레드를 감싸는 간단한 래퍼이다. 이것은 Runnable 클래스로 부터 상속받는 Worker 클래스 인스턴스를 가진다. 공식 문서에는 다음과 같이 기본적 예제가 있다.


```

```