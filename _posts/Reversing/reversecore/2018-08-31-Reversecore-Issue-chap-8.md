---

layout : single
title : "Reversecore chap 8 - abex' crackme #2"
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

4. **N(Native) code, P(Pseudo) code** : <br/>
  VB 파일은 옵션에 따라 N code와 P code로 컴파일이 가능하다. N code는 일반적인 디버거에서 해석 가능한 IA-32 Instruction을 사용하는 반면, P code는 인터프리터 언어 개념으로서 VB 엔진으로 가상 머신을 구현하여 자체적으로 해석 가능한 명령어(바이트 코드)를 사용하는 것이다. P code는 간단히 말해서 Java 처럼 Java EVM 을 사용한다고 보면 이해하기 편할 것 같다.

5. **EP**(EntryPoint) 주변의 코드를 한 번 보자.

  주소 | Hex | Assembly Code | Comment
  |:------|:--------|:---------|:--------|
	00401232 | FF25 A0104000 | JMP DWORD PTR DS:[4010A0] | ; MSVBV60.ThunRTMain
	00401238 | 68 141E4000 | PUSH 401E14 | ; = EP
	0040123D | E8 F0FFFFFF | CALL 00401232 | ; JMP.&MSVBVM60.#100

	즉, **EP** 이후 바로 첫째 줄로 가는 것이 아닌, 3번째 줄의 **JMP**명령어를 거쳐 가는 것을 볼 수 있는데, 이 기법은 VC++, VB 컴파일러에서 많이 사용하는 간접호출 (Indirect Call) 기법이다.


6. 새로운 어셈블리 명령어를 배워보자.

- **TEST** : 논리 비교 (Logical Compare)

bit-wise logical 'AND' 연산과 동일(operand 값이 변경되지 않고 EFLAGS 레지스터만 변경됨).

두 operand 중에 하나가 0이면 AND 연산 결과는 0 -> ZF = 1로 세팅됨.

예를 하나 보도록 하자.

~~~
TEST AX, AX
JE 403408
~~~

위 어셈블리 코드를 C 언어로 해석하면 아래와 같다. 

~~~
If(AX == 0)
  goto 403408
~~~

<br/>

**JE** : 조건 분기 (Jump if equal)

ZF = 1 이면 점프

<br/>

**LEA** : Load Effective Address

해당 명령어는 사실 **MOV**와 비슷한 기능을 한다.

그럼 **LEA**와 **MOV**의 명령어의 차이점은 무엇일까?

LEA는 사실 MOV보다 더 간단한데, MOV는 메모리가 가리키는 값을 가져오고, LEA는 그냥 메모리의 주소값을 가져온다. 예시 코드를 한 번 보자.

가령 레지스터와 메모리에 다음과 같은 값이 들어 있다고 가정해 보자.

esi : 0x401000 (esi에는 0x401000이라는 값이 들어 있다)
*esi : 5640EC83 (esi가 가리키는 번지에는 5604EC93라는 값이 들어 있다)

이때 코드의 의미를 살펴보면,

~~~
lea eax, dword ptr ds:[esi]
// esi가 0x401000이므로 eax에는 0x401000이 들어온다.
mov eax, dword ptr ds:[esi]
// esi가 0x401000이므로 eax에는 0x401000 번지가 가리키는 5640EC83이라는 값이 들어온다!
~~~

즉 eax는 그냥 번지값, mov는 번지가 가리키는 값을 가져온다는 사실을 한 눈에 파악할 수 있다!

(출처는 '리버스 엔지니어링 바이블'에서 가져옴)
 

  



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

