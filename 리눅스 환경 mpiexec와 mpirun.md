

만약 그냥 `mpirun -np 4 ./ch09_2`로 실행 파일을 실행하면 이런 결과가 나온다.


``` bash
┌──(kali㉿kali)-[~/Documents/C-programming/Multithreading_CPP/Chapter09] └─$ mpirun -np 4 ./ch09_2 [kali][[9403,1],1][../../../../../../opal/mca/btl/tcp/btl_tcp_proc.c:266:mca_btl_tcp_proc_create_interface_graph] Unable to find reachable pairing between local and remote interfaces [kali][[9403,1],3][../../../../../../opal/mca/btl/tcp/btl_tcp_proc.c:266:mca_btl_tcp_proc_create_interface_graph] Unable to find reachable pairing between local and remote interfaces [kali][[9403,1],1][../../../../../../opal/mca/btl/tcp/btl_tcp_proc.c:266:mca_btl_tcp_proc_create_interface_graph] Unable to find reachable pairing between local and remote interfaces [kali][[9403,1],3][../../../../../../opal/mca/btl/tcp/btl_tcp_proc.c:266:mca_btl_tcp_proc_create_interface_graph] Unable to find reachable pairing between local and remote interfaces [kali][[9403,1],3][../../../../../../opal/mca/btl/tcp/btl_tcp_proc.c:266:mca_btl_tcp_proc_create_interface_graph] Unable to find reachable pairing between local and remote interfaces [kali][[9403,1],1][../../../../../../opal/mca/btl/tcp/btl_tcp_proc.c:266:mca_btl_tcp_proc_create_interface_graph] Unable to find reachable pairing between local and remote interfaces [kali][[9403,1],2][../../../../../../opal/mca/btl/tcp/btl_tcp_proc.c:266:mca_btl_tcp_proc_create_interface_graph] Unable to find reachable pairing between local and remote interfaces [kali][[9403,1],0][../../../../../../opal/mca/btl/tcp/btl_tcp_proc.c:266:mca_btl_tcp_proc_create_interface_graph] Unable to find reachable pairing between local and remote interfaces [kali][[9403,1],2][../../../../../../opal/mca/btl/tcp/btl_tcp_proc.c:266:mca_btl_tcp_proc_create_interface_graph] Unable to find reachable pairing between local and remote interfaces [kali][[9403,1],0][../../../../../../opal/mca/btl/tcp/btl_tcp_proc.c:266:mca_btl_tcp_proc_create_interface_graph] Unable to find reachable pairing between local and remote interfaces [kali][[9403,1],2][../../../../../../opal/mca/btl/tcp/btl_tcp_proc.c:266:mca_btl_tcp_proc_create_interface_graph] Unable to find reachable pairing between local and remote interfaces [kali][[9403,1],0][../../../../../../opal/mca/btl/tcp/btl_tcp_proc.c:266:mca_btl_tcp_proc_create_interface_graph] Unable to find reachable pairing between local and remote interfaces 
Hell world from processor kali, rank 1out of 4 processors 
Hell world from processor kali, rank 0out of 4 processors 
Hell world from processor kali, rank 3out of 4 processors 
Hell world from processor kali, rank 2out of 4 processors
```

출력된 메시지는 `Unable to find reachable pairing between local and remote interfaces` 경고와 함께 프로그램이 실행되었다. 경고에도 불구하고 프로그램이 성공적으로 실행되어 모든 프로세스가 "Hello World" 메시지를 출력한 것을 확인할 수 있다.

이 경고는 OpenMPI가 네트워크 인터페이스 간의 연결을 설정하려 시도했지만, 적절한 인터페이스를 찾지 못했음을 나타낸다. 이는 로컬 머신에서 실행하거나, 잘못된 네트워크 설정 또는 OpenMPI의 기본 TCP 설정과 관련이 있다.

---
# 경고 해결 방법

#### 1. **Loopback** 인터페이스만 사용하도록 강제 설정

OpenMPI에서 loopback 인터페이스(`lo`)만 사용하도록 명시적으로 설정한다.

``` bash
mpirun --mca btl_tcp_if_include lo -np 4 ./ch09_2
```

실행결과

``` bash
┌──(kali㉿kali)-[~/Documents/C-programming/Multithreading_CPP/Chapter09]
└─$ mpirun --mca btl_tcp_if_include lo -np 4 ./ch09_2

Hell world from processor kali, rank 3out of 4 processors
Hell world from processor kali, rank 2out of 4 processors
Hell world from processor kali, rank 0out of 4 processors
Hell world from processor kali, rank 1out of 4 processors
                                                   
```

내용을 보면 `processor kali`라고 나와 있다. 현재 운영체제가 칼리 리눅스이며, 이름 또한 그대로 kali이기 때문에 정확히 나온 것이다.
#### 2. **TCP 대신 공유 메모리 및 자체 통신 사용**

**로컬 시스템에서 실행 중**이라면, TCP 대신 공유 메모리(`sm`)와 자체 통신(`self`) 방식을 사용하도록 설정합니다.

``` bash
mpirun --mca btl self,sm -np 4 ./ch09_2
```

#### 3. **환경 변수 설정**

OpenMPI 실행 시 사용 가능한 네트워크 인터페이스를 제어하는 환경 변수를 설정합니다. 예를 들어, loopback 인터페이스를 기본으로 지정합니다.

``` bash

export OMPI_MCA_btl_tcp_if_include=lo
export OMPI_MCA_btl=self,sm mpirun -np 4 ./ch09_2

```

---
# 경고 무시

이 경고는 단일 노드에서 실행할 때 MPI의 기본 네트워크 설정이 필요 이상으로 엄격하게 동작한 결과일 가능성이 큽니다. 프로그램이 제대로 실행되었다면 이 경고를 무시해도 괜찮습니다.

---
# 추가 확인

- 네트워크 인터페이스 상태를 확인합니다:
    
    `ip addr`
    
    네트워크 인터페이스(`lo` 포함)가 제대로 설정되어 있는지 확인하세요.
    
- OpenMPI 설정 정보 확인:
    
    
    `ompi_info --all | grep btl`
    
- OpenMPI의 최신 버전을 설치하거나 재설치합니다:
    
    
    `sudo apt update sudo apt install --reinstall libopenmpi-dev openmpi-bin`