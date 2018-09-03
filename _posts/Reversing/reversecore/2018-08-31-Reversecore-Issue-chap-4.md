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

범용 레지스터 (General Purpose Registers)는 이름처럼 범용적으로 사용되는 레지스터들이다. IA-32에서 각각의 범용 레지스터들의 크기는 32비트(4바이트)이다. 보통은 상수



<br/>

---



### Issue #1

해결해야 할 이슈 : 

