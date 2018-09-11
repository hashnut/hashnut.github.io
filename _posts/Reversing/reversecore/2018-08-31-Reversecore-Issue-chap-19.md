---

layout : single
title : "Reversecore chap 19 - UPack 디버깅, OEP 찾기!"
categories : [Reversing]
tags : [reversecore]
comment : true

---

### '리버싱 핵심 원리'의 내용 및 이슈들과 해결책을 다룹니다.

---

<br/>


UPack 패커를 이용하여 압축한 notepad.exe 파일의 EP(Entry Point)를 찾아 디버깅 해보자.


---

### OllyDbg 디버깅 에러

Stud_PE를 이용하여 EP의 VA를 알아낼 수 있다.

![1](https://user-images.githubusercontent.com/26838115/45330036-4f052e80-b59e-11e8-8c8a-216c151263f1.png)

RVA 값이 1018이고 ImageBase가 01000000이므로, 01001018이 실제 EP의 주소값이라는 것을 알 수 있다.

이제 OllyDbg를 실행하여 01001018 주소를 EP로 설정하면 된다!

![2](https://user-images.githubusercontent.com/26838115/45330114-cd61d080-b59e-11e8-8e54-8af805f98907.png)

---

### 디코딩 루프

EP부터 차근차근 디코딩을 해 보자.

![3](https://user-images.githubusercontent.com/26838115/45330166-1023a880-b59f-11e8-87fd-baf9df4dba04.png)

처음 두 명령은 010011B0 주소에서 4 바이트를 읽어 EAX에 저장하는 명령이다. EAX는 0100739D 값을 가지는데, 이는 원본 notepad의 OEP(Original Entry Point)이다. LODS DWORD PTR DS:[ESI] 명령어를 해석해 보면 ESI가 가리키는 주소에서 4 바이트 값을 읽어서 EAX레지스터에 저장하라는 의미이다. 사실 이 값이 OEP인 것을 알고 있다면, 하드웨어 BP를 설치한 후 실행[F9]하면 정확히 OEP에서 멈춘다.

일단 다시 정석대로 진행하면, 다음과 같은 호출 코드가 나타난다.

![3-1](https://user-images.githubusercontent.com/26838115/45350961-f9f20880-b5ef-11e8-9189-69d7dae41eb3.png)

이 때 ESI 값은 0102718C이고, 0102718C가 가리키는 값을 Hex Dump에서 찾아보면, 0101FCCB의 값을 가진다는 것을 알 수 있다. 이게 바로 decode() 함수의 주소이다. 앞으로 이 함수가 반복적으로 호출된다. decode() 함수(0101FCCB)를 살짝 살펴보자.


![5](https://user-images.githubusercontent.com/26838115/45351126-5b19dc00-b5f0-11e8-8af7-8b38d91c0e2d.png)

[F7] 명령어(Step into)로 계속 트레이싱을 진행하면 다음과 같은 코드를 만나게 된다.  

![6](https://user-images.githubusercontent.com/26838115/45353288-6a4f5880-b5f5-11e8-99f2-e62f70565175.png)

0101FE57과 0101FE5D 주소에는 EDI 값이 가리키는 곳에 뭔가를 쓰는 명령어(MOVS, STOS)가 있다. 이때 EDI 값은 첫 번째 섹션 내의 주소를 가리킨다. 즉 압축을 해제한 후 실제 메모리에 쓰는 명령어들이다. 0101FE5E와 0101FE61 주소에는 CMP/JB 명령어를 통해서 EDI 값이 01014B5A가 될 때까지 계속 루프를 돌게 된다([ESI+34] = 01014B5A). 0101FE61 주소가 바로 디코딩 루프의 끝부분이다. 실제로 이 루프를 반복해서 트레이싱하면 EDI가 가리키는 주소에 어떤 값들이 쓰여지는 것을 볼 수 있다.

넘어가기 전에, 위 그림에서 나온 어셈블리 명령어를 조금 알아보자. <br/>

> REP MOVS BYTE PTR ES:[EDI], BYTE PTR DS:[ESI]

~~~
REP MOVS DWORD PTR ES:[EDI], DWORD PTR [ESI] is a synonym for REP MOVSD; and REP MOVS BYTE PTR ES:[EDI], BYTE PTR[ESI] is a synonym of REP MOVSB.

There are the following MOVS commands, based on data sizes:

MOVSB (byte, 8-bit)
MOVSW (word, 16-bit)
MOVSD (dword, 32-bit)
MOVSQ (qword, 64 bit) - only available in 64-bit mode

The MOVS command copies data from DS:(SI/ESI/RSI) to ES:(DI/EDI/RDI) -- the size of SI/DI register is based on your current mode - 16-bit, 32-bit or 64-bit. It also increases (decreases) SI and DI registers (based on the D flag, set CLD to increase the registers).

The MOVS command cannot use other registers than SI/DI, so it is not necessary to specify them.

If the MOVS command is prefixed by REP, it is repeated to copy CX(ECX/RCX) number of bytes, decreasing CX, so at the end CX becomes zero.
~~~

> LODS DWORD PTR DS:[ESI]

~~~
The lods instruction is unique among the string instructions.
You will never use a repeat prefix with this instruction.

The lods instruction copies the byte or word pointed at by ds:si into the al, ax, or eax register, after which it increments or decrements the si register by one, two, or four. 

The size of the operation determines which register is targeted and how far the ESI register is advanced. For LODS DWORD, a double-word (32-bit) datum is loaded, which means the 32-bit EAX register. LODS WORD would be 16-bit into the 16-bit AX register, and LODS BYTE would be the 8-bit AL.
~~~

> STOS BYTE PTR DS:[ESI]

~~~
The STOS instruction copies the data item from AL (for bytes - STOSB), AX (for words - STOSW) or EAX (for doublewords - STOSD) to the destination string, pointed to by ES:DI in memory.
~~~

---

### IAT 세팅

일반적인 패커에서는 디코딩 루프가 끝나면 원본 파일에 맞게 IAT를 새롭게 구성한다. UPack도 같은 과정을 거친다.

![7](https://user-images.githubusercontent.com/26838115/45355533-966dd800-b5fb-11e8-85ca-53c44c785b00.png)


UPack이 임포트하는 2개의 함수 LoadLibrary와 GetProcAddress를 이용하여 루프를 돌면서 원본 notepad의 IAT를 구성한다(notepad에서 임포트하는 함수들의 실제 메모리 주소를 얻어서 원본 IAT 영역에 쓰는 것이다). 이 과정이 끝나면 0101FEAF 주소의 RETN 명령어에 의해서 드디어 OEP로 가게 된다.

![8](https://user-images.githubusercontent.com/26838115/45355834-74288a00-b5fc-11e8-9108-8b5a34030eeb.png)



<br/>

---



### Issue #1

- 해결해야 할 이슈 : 

위의 **섹션 겹쳐쓰기** 파트에서, 두 번째 섹션에 압축된 파일 이미지를 첫 번째 섹션에 그대로 압축해제를 한다. 이때, 메모리에서 첫 번째 섹션은 원본 파일(notepad.exe)의 Size of Image와 같다. 그렇다면, 2nd, 3rd Section도 메모리에 로딩될텐데, 이는 불필요하게 메모리를 많이 소모하는 문제를 발생시킬 수 있는 것인가? 즉, UPack에서 파일 크기를 줄이는 기능의 trade-off는 메모리 사용량의 증가인가? 아니면 해당 메모리 영역은 실제로 아예 사용을 하지 않는 것이고, Offset만 표기된 것인가?

---


### Issue #2

아래와 같은 어셈블리 명령어가 있다고 치자

~~~
CMP EDI,DWORD PTR DS:[ESI+34]
JB 0101FD13
~~~

이는 ESI+34에 담긴 DWORD 바이트 만큼의 값(4바이트, 32비트이니 Hex Dump로 따지면 4칸이다)이 EDI와 같은지를 비교해서 ZF를 설정하는 것일텐데, JB의 설명을 찾아보면 이렇게 나온다.

> JB : Jump near if below (CF=1)

즉, Carry Flag가 1이면 Jump 한다는 것인데, Carry Flag는 아래와 같은 역할을 한다 :

> Carry Flag : 부호 없는 수(unsigned integer)의 오버플로가 발생했을 때 1로 세팅된다.

CMP 명령어를 통해 두 값을 비교하면 ZF의 값만 바뀌는 것이 아니라 CF 값도 변하는 것인가?

그런데 ESI+34의 Hex Dump 값과 EDI의 값이 같을때는, JB문에서 점프가 실행되지 않는다.

그렇다면 JB를 JNE 처럼 CMP한 두 값이 같지 않을 때 점프를 수행한다고 알고 있어도 좋을 것인가?

애초에 CMP가 Carry Flag 값을 변경할 이유가 있는가?

이슈 해결 바람.

---

### Issue #2 Solution

CMP 명령이 어떻게 수행되는지 자세히 살펴보자.

CMP는 앞의 변수에서 뒤의 변수를 뺀 다음, 그 값을 통해 대수 비교를 진행한다(다른 프로그래밍 언어의 compare 함수들과 원리가 비슷함). 따라서, 만약 뒤의 값이 앞의 값보다 클 경우, unsigned - unsigned 연산에서 음수가 나오므로, overflow가 발생한다! 즉, 뒤의 변수가 앞의 변수보다 크면, CF = 1로 세팅이 되는 것이다.

위의 경우에서, **CMP EDI,DWORD PTR DS:[ESI+34]**를 보면 EDI가 ESI+34 주소에 저장된 값보다 작아 CF가 1로 세팅이 되면, JB 명령어는 CF가 1인 것을 확인하고 점프를 수행하는 것이다.

EFL 레지스터의 Flag들의 값이 언제 변하는지는 숙지하고 있어야 한다.

~~~
Assume result = op1 - op2

CF - 1 if unsigned op2 > unsigned op1
OF - 1 if sign bit of OP1 != sign bit of result
SF - 1 if MSB (aka sign bit) of result = 1
ZF - 1 if Result = 0 (i.e. op1=op2)
AF - 1 if Carry in the low nibble of result
PF - 1 if Parity of Least significant byte is even
~~~

OF와 CF에 대한 자세한 글은 [여기](http://teaching.idallen.com/dat2343/10f/notes/040_overflow.txt)에서 읽어볼 수 있다. 