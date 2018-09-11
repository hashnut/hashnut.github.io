---

layout : single
title : "Reversecore chap 4 - IA-32 Register 기본 설명"
categories : [Reversing]
tags : [reversecore]
comment : true

---

### '리버싱 핵심 원리'의 내용 및 이슈들과 해결책을 다룹니다.

---

<br/>


- **1.** **CPU 레지스터란?**

레지스터(Register)란 CPU 내부에 존재하는 다목적 저장 공간이다. 우리가 일반적으로 메모리라고 얘기하는 RAM(Random Access Memory)과는 조금 성격이 다르다. CPU가 RAM에 있는 데이터를 엑세스(Access)하기 위해서는 물리적으로 먼 길을 돌아가야 하기 때문에 시간이 오래 걸린다. 하지만 레지스터는 CPU와 한 몸이기 때문에 고속으로 데이터를 처리할 수 있다.

| 레지스터의 종류 |
|:------|
|**Basic program execution registers**|
|x87 FPU registers|
|MMX registers|
|XMM registers|
|Control registers|
|Memory management registers|
|Debug registers|
|...|

- **2.** 먼저 **Basic program Execution registers**에 대해 알아보자.

Basic program execution registers | 구성 요소
|:-----|:------|
General Purpose Registers (32비트 - 8개) | EAX
| EBX
| ECX
| EDX
| ESI
| EDI
| EBP
| ESP
Segment Resiters (16비트 - 6개) | CS
| DS
| SS
| ES
| GS
| GS
Program Status and Control Register (32비트 - 1개) | EFLAGS
Instruction Pointer (32 비트 - 1개) | EIP

<br/>

> 참고 : 레지스터 이름에 E(Extended)가 붙은 경우는 예전 16비트 CPU인 IA-16시절부터 존재하던 16비트 크기의 레지스터들을 32비트 크기로 확장시켰다는 뜻임.

<br/>

- **3.** **범용 레지스터 (General Purpose Registers)**

범용 레지스터 (General Purpose Registers)는 이름처럼 범용적으로 사용되는 레지스터들이다. IA-32에서 각각의 범용 레지스터들의 크기는 32비트(4바이트)이다. 보통은 상수/주소 등을 저장할 때 주로 사용되며, 특정 어셈블리 명령어에서는 특정 레지스터를 조작 하기도 한다. 또 어떤 레지스터들은 특수한 용도로 사용되기도 한다.

![4-1](https://user-images.githubusercontent.com/26838115/44966263-22855d00-af75-11e8-8d36-928b352dee77.png)

위의 그림을 보자. <br/><br/>

각 레지스터들은 16비트 하위 호환을 위하여 몇 개의 구획으로 나뉘어진다. EAX를 기준으로 살펴보자.

- EAX : (0 ~ 31) 32비트
- AX : (0 ~ 15) EAX의 하위 16비트
- AH : (8 ~ 15) AX의 상위 8비트
- AL : (0 ~ 8) AX의 하위 8비트

<br/>

즉 4바이트(32비트)를 다 사용하고 싶을 때는 EAX를 사용하고, 2바이트(16비트)만 사용할 때는 EAX의 하위 16비트 부분인 AX를 사용하면 된다. 이런 식으로 하나의 32비트 레지스터를 상황에 맞게 8비트, 16비트, 32비트로 알뜰하게 사용할 수 있다! <br/><br/>

각 레지스터의 이름은 아래와 같다. 

- EAX : Accumulator for operands and results data
- EBX : Pointer to data in the DS segment
- ECX : Counter for string and loop operations
- EDX : I/O pointer

<br/>

위의 4개의 레지스터들은 주로 산술연산(ADD, SUB, XOR, OR 등) 명령어에서 상수/변수 값의 저장 용도로 많이 사용된다. 어떤 어셈블리 명령어(MUL, DIV, LODS 등)들은 특정 레지스터를 직접 조작하기도 한다(이런 명령어가 실행된 이후에 특정 레지스터들의 값이 변경됨).<br/><br/>

그리고 추가적으로 ECX와 EAX는 특수한 용도로도 사용된다. ECX는 반복문 명령어(LOOP)에서 반복 카운트(loop count)로 사용된다(루프를 돌 때마다 ECX를 1씩 감소시킨다). EAX는 일반적으로 함수 리턴 값에 사용된다. 모든 Win32 API 함수들은 리턴 값을 EAX에 저장한 후 리턴한다. <br/><br/>

> Windows 어셈블리 프로그래밍에서 주의할 점!
>
>Win32 API 함수들은 내부에서 ECX와 EDX를 사용한다. 따라서 이런 API가 호출되면 ECX와 EDX의 값이 변경된다. 따라서 ECX와 EDX에 중요한 값이 저장되어 있다면 API 호출 전에 다른 레지스터나 스택에 백업해야 한다!

<br/>

나머지 범용 레지스터들의 이름은 아래와 같다.

- EBP : Pointer to data on the stack (in the SS segment)
- ESI : source pointer for string operations
- EDI : destination pointer for string operations
- ESP : Stack pointer (in the SS segment)

<br/>

위 4개의 레지스터들은 주로 메모리 주소를 저장하는 포인터로 사용된다. <br/>
ESP는 스택 메모리 주소를 가리킨다. 어떤 명령어들(PUSH, POP, CALL, RET)은 ESP를 직접 조작하기도 한다(스택 메모리 관리는 프로그램에서 매우 중요하므로, ESP를 다른 용도로 사용하지 말아야 한다). <br/>

EBP는 함수가 호출되었을 때 그 순간의 ESP를 저장하고 있다가, 함수가 리턴하기 직전에 다시 ESP에 값을 되돌려줘서 스택이 깨지지 않도록 한다(스택 프레임 기법). ESI와 EDI는 특정 명령어들(LODS< STOS, REP MOVS 등)과 함께 주로 메모리 복사에 사용된다.

- **4.** **세그먼트 레지스터**

세그먼트(Segment)란 IA-32의 메모리 관리 모델에서 나오는 용어이다. IA-32 보호 모드에서 세그먼트란 메모리를 조각내어 각 조각마다 시작 주소, 범위, 접근 권한 등을 부여해서 메모리를 보호하는 기법을 말한다. 또한 세그먼트는 페이징(Paging) 기법과 함께 가상 메모리를 실제 물리 메모리로 변경할 때 사용된다. 세그먼트 메모리는 Segment Descriptor Table(SDT)이라고 하는 곳에 기술되어 있는데, 세그먼트 레지스터는 바로 이 SDT의 index를 가지고 있다.

![4-2](https://user-images.githubusercontent.com/26838115/44966492-7775a300-af76-11e8-9ab6-c0508eaef6d7.png)

위 그림을 보면, 각 세그먼트 레지스터가 가리키는 세그먼트 디스크립터(Segment Descriptor)와 가상 메모리가 조합되어 선형주소(Linear Adderss)가 되며, 페이징 기법에 의해서 선형 주소가 최종적으로 물리주소(Physical Address)로 변환됩니다. 만약 OS에서 페이징을 사용하지 않는다면 선형주소는 그대로 물리주소가 된다.<br/><br/>

각 세그먼트 레지스터의 이름은 아래와 같다.

S / R | Name
|:----|:----|
CS | Code Segment
SS | Stack Segment
DS | Data Segment
ES | Extra(Data) Segment
FS | Data Segment
GS | Data Segment

- **5.** **프로그램 상태와 컨트롤 레지스터**

S / R | Name
|:----|:----|
EFLAGS | Flag Register

플래그(Flag) 레지스터의 이름은 EFLAGS이며 32비트(4바이트) 크기이다(EFLAGS 레지스터 역시 16비트의 FLAGS 레지스터의 32비트 확장 형태이다). <br/><br/>

EFLAGS 레지스터는 각각의 비트마다 의미를 가지고 있다. 각 비트는 1 또는 0의 값을 가지는데, 이는 On/Off 혹은 True/False를 의미한다. 리버싱 입문 단계에서는 애플리케이션 디버깅에 필요한 3가지 flag(ZF, OF, CF)에 대해서만 잘 이해하면 된다. 이들 3개의 플래그가 중요한 이유는 특히 조건 분기 명령어(Jcc)에서 이들 Flag의 값을 확인하고 그에 따라 동작 수행 여부를 결정하기 때문이다.

![4-3](https://user-images.githubusercontent.com/26838115/44966754-020ad200-af78-11e8-861a-cd2859ee8516.png)

- Zero Flag(ZF)
	연산 명령 후에 결과 값이 0이 되면 ZF가 1(True)로 세팅된다.

- Overflow Flag(OF)
	부호 있는 수(signed integer)의 오버플로가 발생했을 때 1 로 세팅된다. 그리고 MSB(Most Significant Bit)가 변경되었을 때 1로 세팅된다.

- Carry Flag(CF)
	부호 없는 수(unsigned integer)의 오버플로가 발생했을 때 1로 세팅된다.

- **6.** **Instruction Pointer**

- EIP : Instruction pointer

Instruction Pointer는 CPU가 처리할 명령어의 주소를 나타내는 레지스터이며, 크기는 32비트(4바이트)이다(16비트의 IP 레지스터의 확장 형태). CPU는 EIP에 저장된 메모리 주소의 명령어(instruction)을 하나 처리하고 난 후 자동으로 그 명령어 길이만큼 EIP를 증가시킨다. 이런 식으로 계속 명령어를 처리해 나간다. <br/><br/>

범용 레지스터들과는 다르게 EIP는 그 값을 직접 변경할 수 없도록 되어 있어서 다른 명령어를 통해 간접적으로 변경해야 한다. EIP를 변경하고 싶을 때는 특정 명령어(JMP, Jcc, CALL, RET)를 사용하거나 인터럽트(interrupt), 예외(exception)을 발생시켜야 한다.

<br/>

---



### Issue #1

해결해야 할 이슈 : 

