---

layout : single
title : "Reversecore Issue chap 4 - IA-32 Register 기본 설명"
categories : [Reversing]
tags : [reversecore]
comment : true

---

### '리버싱 핵심 원리'의 내용 및 이슈들과 해결책을 다룹니다.

---

<br/>


1. **CPU 레지스터란?**

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

2. 먼저 Basic program Execution registers에 대해 알아보자.

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

	- 참고 : 레지스터 이름에 E(Extended)가 붙은 경우는 예전 16비트 CPU인 IA-16시절부터 존재하던 16비트 크기위 레지스터들을 32비트 크기로 확장시켰다는 뜻임.

3. **범용 레지스터 (General Purpose Registers)**

	범용 레지스터 (General Purpose Registers)는 이름처럼 범용적으로 사용되는 레지스터들이다. IA-32에서 각각의 범용 레지스터들의 크기는 32비트(4바이트)이다. 보통은 상수/주소 등을 저장할 때 주로 사용되며, 특정 어셈블리 명령어에서는 특정 레지스터를 조작 하기도 한다. 또 어떤 레지스터들은 특수한 용도로 사용되기도 한다.

	![4-1](https://user-images.githubusercontent.com/26838115/44966263-22855d00-af75-11e8-8d36-928b352dee77.png)

	위의 그림을 보자. <br/><br/>

	각 레지스터들은 16비트 하위 호환을 위하여 몇 개의 구획으로 나뉘어진다. EAX를 기준으로 살펴보자.

	- EAX : (0 ~ 31) 32비트
	- AX : (0 ~ 15) EAX의 하위 16비트
	- AH : (8 ~ 15) AX의 상위 8비트
	- AL : (0 ~ 8) AX의 하위 8비트

	즉 4바이트(32비트)를 다 사용하고 싶을 때는 EAX를 사용하고, 2바이트(16비트)만 사용할 때는 EAX의 하위 16비트 부분인 AX를 사용하면 된다. 이런 식으로 하나의 32비트 레지스터를 상황에 맞게 8비트, 16비트, 32비트로 알뜰하게 사용할 수 있다! <br/><br/>

	각 레지스터의 이름은 아래와 같다. 

	- EAX : Accumulator for operands and results data
	- EBX : Pointer to data in the DS segment
	- ECX : Counter for string and loop operations
	- EDX : I/O pointer

	위의 4개의 레지스터들은 주로 산술연산(ADD, SUB, XOR, OR 등) 명령어에서 상수/변수 값의 저장 용도로 많이 사용된다. 어떤 어셈블리 명령어(MUL, DIV, LODS 등)들은 특정 레지스터를 직접 조작하기도 한다(이런 명령어가 실행된 이후에 특정 레지스터들의 값이 변경됨).<br/><br/>

	그리고 추가적으로 ECX와 EAX는 특수한 용도로도 사용된다. ECX는 반복문 명령어(LOOP)에서 반복 카운트(loop count)로 사용된다(루프를 돌 때마다 ECX를 1씩 감소시킨다). EAX는 일반적으로 함수 리턴 값에 사용된다. 모든 Win32 API 함수들은 리턴 값을 EAX에 저장한 후 리턴한다. <br/><br/>

	- Windows 어셈블리 프로그래밍에서 주의할 점!
		Win32 API 함수들은 내부에서 ECX와 EDX를 사용한다. 따라서 이런 API가 호출되면 ECX와 EDX의 값이 변경된다. 따라서 ECX와 EDX에 중요한 값이 저장되어 있다면 API 호출 전에 다른 레지스터나 스택에 백업해야 한다!

	나머지 범용 레지스터들의 이름은 아래와 같다.

	- EBP : Pointer to data on the stack (in the SS segment)
	- ESI : source pointer for string operations
	- EDI : destination pointer for string operations
	- ESP : Stack pointer (in the SS segment)

	위 4개의 레지스터들은 주로 메모리 주소를 저장하는 포인터로 사용된다. <br/>
	ESP는 스택 메모리 주소를 가리킨다. 어떤 명령어들(PUSH, POP, CALL, RET)은 ESP를 직접 조작하기도 한다(스택 메모리 관리는 프로그램에서 매우 중요하므로, ESP를 다른 용도로 사용하지 말아야 한다). <br/><br/>

	EBP는 함수가 호출되었을 때 그 순간의 ESP를 저장하고 있다가, 함수가 리턴하기 직전에 다시 ESP에 값을 되돌려줘서 스택이 깨지지 않도록 한다(스택 프레임 기법). ESI와 EDI는 특정 명령어들(LODS< STOS, REP MOVS 등)과 함께 주로 메모리 복사에 사용된다.

4. 





<br/>

---



### Issue #1

해결해야 할 이슈 : 

