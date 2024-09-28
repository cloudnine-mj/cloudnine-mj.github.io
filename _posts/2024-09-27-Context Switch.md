# 운영 체제 기본

## Context Switch

:point_right: 하나의 CPU는 동일한 시간에 하나의 작업만 수행할 수 있음. (여러 프로세스 동시 실행 불가)

* Context Switch란?
	* 하나의 CPU에서 여러 프로세스를 동시성으로 처리하기 위해서는 한 프로세스에서 다른 프로세스로 전환해야 하는데, 이것을 Context Switch라고 함.

<br>
## Context

* 프로세스 간 전환을 위해서는 이전에 어디까지 명령을 수행했는지, CPU 레지스터에는 어떤 값이 저장되어 있는지에 대한 정보가 필요

* Context란?
	* CPU가 해당 프로세스를 실행하기 위한 프로세스 정보를 의미
	* 이 정보들은 OS가 관리하는 PCB라고 하는 자료 구조의 공간에 저장됨.
	<br>
## PCB (Process Control Block)

* OS가 시스템 내의 프로세스들을 관리하기 위해 프로세스 마다 유지하는 정보를 담는 커널 내의 자료 구조

<br>
## Context Switching 이란?

* CPU가 프로세스 간 PCB 정보를 교체하고 캐시를 비우는 일련의 과정

<br>
## 프로세스 상태

| 상태       | 설명                                                         |
| ---------- | ------------------------------------------------------------ |
| New        | 프로세스를 생성하고 있는 단계 (커널 영역에 PCB가 만들어진 상태) |
| Ready      | 프로세스가 CPU를 할당받기 위해 기다리고 있는 상태            |
| Running    | 프로세스가 CPU를 할당받아 명령어를 실행 중인 상태            |
| Waiting    | 프로세스가 I/O 작업 완료 혹은 사건 발생을 기다리는 상태      |
| Terminated | 프로세스가 종료된 상태                                       |

<br>

## Context Switching 이 일어나는 조건

* 실행 중인 프로세스에서 I/O 호출이 일어나 해당 I/O 작업이 끝날 때까지 프로세스 상태가 Running에서 Waiting 상태로 전이된 경우

* Round Robin 스케줄링 등 운영체제의 CPU 스케줄러 알고리즘에 의해 현재 실행 중인 프로세스가 사용할 수 있는 시간 자원을 모두 사용했을 때, 해당 프로세스를 중지하고(Ready 상태로 전이) 다른 프로세스를 실행시켜주는 경우

:point_right: 프로세스 스케줄링 알고리즘에 의해서 현재 실행 중인 프로세스가 중지되고 다른 프로세스를 실행시켜줘야 하는 경우


> **Round Robin 스케줄링**
> * 선점적(Preemptive) 스케줄링 방식
> * Round Robin은 가슴이 붉은 새인 로빈에게서 유래됨. (로빈은 새에게 먹이를 줄 떄 순서대로 준다. 즉, 자식이 5마리면 먹이를 작게 잘라 순서대로 주고, 순서가 끝난다면 다시 처음부터 먹이를 또 순서대로 준다.)
> * 한 프로세스가 할당받은 시간(time slice, time quantum) 동안 작업을 하다가 작업을 완료하지 못하면 ready queue의 맨 뒤로 가서 자기 차례를 기다리는 방식. (프로세스들이 작업을 완료할 때까지 계속 순환을하면서 실행)
> * 선점형 방식 중 가장 단순

<br>
## Thread Context Switching

### TCB (Thread Control Block)

* Thread 상태 정보를 저장하는 자료 구조
* PC와 Register Set(CPU 정보), 그리고 PCB를 가리기는 포인터를 가짐
* 스레드가 하나 생성될 때마다 PCB내에서 TCB가 생성됨.
* Context Switching이 일어나면 기존의 Thread TCB를 저장하고 새로운 TCB를 가져와 실행한다.

<br>
## Process와 Thread 비교

* Process는 오버헤드가 큼.
	* Context Switching 할 때 메모리 주소 관련 여러가지 처리(CPU 캐시 초기화, TLB 초기화,  MMU 주소 체계 수정 등...) 를 하기 때문.

* Thread는 Process에 비해 Context Switching이 빠름.
	* Thread는 Process 내 메모리를 공유하기 때문에 메모리 주소 관련 추가적인 작업이 없어서 Process에 비해 오버헤드가 작기 때문.

* Thread는 생성하는 비용이 커서, Thread를 많이 생성하면 메모리 부족 현상이 발생하거나 빈번한 Context Switching으로 인해 애플리케이션의 성능이 저하될 수 있음.







