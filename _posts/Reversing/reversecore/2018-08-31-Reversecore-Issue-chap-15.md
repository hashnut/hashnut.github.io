---

layout : single
title : "Reversecore Issue chap 14 - 실행 압축"
categories : [Reversing]
tags : [reversecore]
comment : true

---

### '리버싱 핵심 원리'의 내용 및 이슈들과 해결책을 다룹니다.

---

<br/>


해당 챕터는 UPX를 이용해 notepad.exe를 압축한 후, 원본 EP (OEP; Original Entry Point)를 찾는 것이 핵심이다. 꼭 다시 실습 해볼 것!


---

### UPX 패커의 특징 몇 가지를 알아보자.

UPX 패커는 PUSHAD 명령으로 EAX~EDI 레지스터 값을 스택에 저장하고, ESI와 EDI 레지스터를 각각 두 번째 섹션 시작 주소(010110000)와 첫 번째 섹션 시작 주소(01001000)로 세팅한다.

디버깅할 때 이처럼 ESI와 EDI가 동시에 세팅되면 ESI가 가리키는 버퍼에서 EDI가 가리키는 버퍼로 메모리 복사가 일어날 거라고 예측할 수 있다.

> 트레이스(Trace)란 코드를 하나하나 실행하면서 쫓아가는 것을 말한다. 


### 방대한 코드를 트레이싱 할 때의 원칙 : 

**"루프를 만나면 그 역할을 살펴본 후 탈출한다"**

- OllyDbg의 트레이싱 명령어를 알아보자.

명령어 | 단축키 | 설명
|:----|:-----|:-----|
Animate Into | Ctrl+F7 | Step Into 명령 반복(화면 표시 됨)
Animate Over | Ctrl+F8 | Step Over 명령 반복(화면 표시 됨)
Trace Into | Ctrl+F11 | Step Into 명령 반복(화면 표시 안 됨)
Trace Over | Ctrl+F12 | Step OVer 명령 반복(화면 표시 안됨)
* 트레이싱을 멈추고 싶을 때는 [F7]을 누르면 된다!||


루프를 빠져나오다 보면 POPAD 코드 아래의, OEP로 가는 JMP 명령어를 찾을 수 있다! (직접 해 볼 것)

---

### UPX의 OEP를 빨리 찾는 방법

- POPAD 명령어 이후의 JMP 명령어에 BP 설치

UPX 패커의 특징을 이용해서, POPAD 명령어 이후의 JMP문에 BP를 설치하면, 바로 OEP로 갈 수 있다.

- 스택에 하드웨어 브레이크 포인트(Hardware Break Point)설치

![15-1](https://user-images.githubusercontent.com/26838115/45200985-57175280-b2ae-11e8-85d5-91f4d9c932bd.png)

> 이 그림에서 스택의 가장 위에 있는 주소로 가면 된다.

PUSHAD 실행 이후, 스택에 "return point"가 적혀있는 스택 주소로 간다(dump로 감). 그후 해당 주소 위치에 하드웨어 BP를 설치하면, 압축이 해제되면서 코드가 실행되고, POPAD가 호출되는 순간에 하드웨어 BP가 설치된 주소를 엑세스하고, 그 때 제어가 멈출 것이다! 그 아래에 OEP로 가는 JMP 명령어를 찾아 볼 수 있다.


<br/>

---



### Issue #1

- 해결해야 할 이슈 : 

