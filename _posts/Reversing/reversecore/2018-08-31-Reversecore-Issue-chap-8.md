---

layout : single
title : "Reversecore Issue chap 8"
categories : [Reversing]
tags : [reversecore, unsolved]
comment : true

---

### '리버싱 핵심 원리'의 내용 및 이슈들과 해결책을 다룹니다.


---

<br/>


1. 챕터 8은 두 번째 crackme를 다룬다. (abex' 2nd crackme)

2. VB 파일은 MSVBVM60.dll(Microsoft Visual Basic Virtual Machine 6.0) 이라는 VB 전용 엔진을 사용한다. (The Thunder Runtime Engine 이라는 이름으로도 불리고 있음)

3. VB 엔진의 사용 예를 보자. 메시지 박스를 출력하고 싶을 때, VB 소스코드에서 MsgBox() 함수를 사용한다. VB 컴파일러는 실제로 MSVBVM60.dll!rtc MsgBox() 함수가 호출되도록 만들고, 이 함수 내부에서 Win32 API인 user32.dll!MessageBox() 함수를 호출해주는 방식으로 동작한다. (VB 소스코드에서 user32.dll!MessageBoxW() 함수를 직접 호출하는 것도 가능하다)

4. N(Native) code, P(Pseudo) code : VB 파일은 옵션에 따라 N code와 P code로 컴파일이 가능하다. N code는 일반적인 디버거에서 해석 가능한 IA-32 Instruction을 사용하는 반면, P code는 인터프리터 언어 개념으로서 VB 엔진으로 가상 머신을 구현하여 자체적으로 해석 가능한 명령어(바이트 코드)를 사용하는 것이다. P code는 간단히 말해서 Java 처럼 Java EVM 을 사용한다고 보면 이해하기 편할 것 같다.

5. **EP**(EntryPoint) 주변의 코드를 한 번 보자.

	00401232 | FF25 A0104000 | JMP DWORD PTR DS:[4010A0] | ; MSVBV60.ThunRTMain
	|:------|:--------|:---------|:--------|
	00401238 | 68 141E4000 | PUSH 401E14 | ; = EP
	0040123D | E8 F0FFFFFF | CALL 00401232 | ; JMP.&MSVBVM60.#100

	즉, **EP** 이후 바로 첫째 줄로 가는 것이 아닌, 3번째 줄의 **JMP**명령어를 거쳐 가는 것을 볼 수 있는데, 이 기법은 VC++, VB 컴파일러에서 많이 사용하는 간접호출 (Indirect Call) 기법이다.


6. 새로운 어셈블리 명령어를 배워보자.

- **TEST** : 논리 비교 (Logical Compare)

  bit-wise logical 'AND' 연산과 동일 (operand 값이 변경되지 않고 EFLAGS 레지스터만 변경됨)<br/>
  두 operand 중에 하나가 0이면 AND 연산 결과는 0 -> ZF = 1로 세팅됨.<br/>
  예를 하나 보도록 하자.<br/>

	  TEST AX, AX
	  JE 403408

  위 어셈블리 코드를 C 언어로 해석하면 아래와 같다. <br/>

	  If(AX == 0)
	      goto 403408
  <br/>
- **JE** : 조건 분기 (Jump if equal)

  ZF = 1 이면 점프

- **LEA** : Load Effective Address

  해당 명령어는 사실 **MOV**와 같은 기능을 한다. <br/>
  그럼 **LEA**와 **MOV**의 명령어의 차이점은 무엇일까? <br/>
  LEA는 OPCODE의 ModR/M 을 이용하여, SRC의 값을 연산한 뒤 REG에 옮길 수 있다.<br/>

  즉, **MOV**는 특정 메모리나 레지스터의 값을 옮기기만 하는 것이고, 
  옮겨지는 값에 연산을 할 수 없다.<br/>
  예를 들어, **LEA EAX, [EBP+1000]**라고 하면
  EBP의 값에 1000을 더한 뒤 EAX에 그 값이 대입된다.<br/>
  만약 EBP가 2000이었으면 EAX의 값은 3000이 된다.<br/>
  그런데 이것을 **MOV**로 표현을 하려면<br/>

	  ADD EBP, 1000
	  MOV EAX, EBP

  처럼, 2줄의 명령어를 사용해야 한다.<br/>
  따라서, LEA를 이용하여 좀 더 명령어를 줄일 수 있는 것이다!<br/>

  그럼 **MOV EAX, [EBP+8]**은 어떻게 작동할까?<br/>
  이것은 EBP+8의 위치에 있는 값을 EAX에 넣게 된다.<br/>
  즉 EBP가 100이라면, 108의 주소에 있는 특정 값(예를 들어 20)을 EAX에 넣게 된다.<br/>
  결론적으로, **LEA**만이 실행중에 계산된 주소를 얻을 수 있다고 알아두면 된다!<br/>

  *(**LEA**에 대한 설명은 **[enes's blog](http://enes.tistory.com/entry/MOVE와-LEA-명령의-차이점)**에서 가져옴.)*

  



<br/>


---



### Issue #1

해결해야 할 이슈 : 


앞서 ***5***을 보면, 간접호출(Indirect Call)기법이 쓰인 것을 볼 수 있다. 

그렇다면 간접호출을 사용하는 이유는 무엇일까?


---


### Issue #2

해결해야 할 이슈 : 

암호화 루프 부분 나중에 다시 시도해 보기. <br/>
책을 보고 해서 망정이지, 혼자 한다고 하면 loop부분을 제대로 해석해내기 힘들 것 같다.

