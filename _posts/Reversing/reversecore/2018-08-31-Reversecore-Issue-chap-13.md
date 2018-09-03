---

layout : single
title : "Reversecore Issue chap 13 - PE File Format"
categories : [Reversing]
tags : [reversecore]
comment : true

---

### '리버싱 핵심 원리'의 내용 및 이슈들과 해결책을 다룹니다.

---

<br/>


**PE(Portable Executable)** 파일은 Windows 운영체제에서 사용되는 실행 파일 형식이다. 기존 UNIX에서 사용되는 COFF(Common Object File Format)를 기반으로 Microsoft에서 만들었다. 애초에는 다른 운영체제에 이식성을 좋게 하려고 만들었으나 현재는 Windows 계열의 OS에서만 사용되고 있다. <br/><br/>


1. **PE File Format**

본격적으로 PE 파일의 종류를 알아보자.

종류 | 주요 확장자
|:-----|:------|
실행 계열 | EXE, SCR
라이브러리 계열 | DLL, OCX, CPL, DRV
드라이버 계열 | SYS, VXD
오브젝트 파일 계열 | OBJ

엄밀히 얘기하면 OBJ(오브젝트) 파일을 제외한 모든 것은 실행 가능한 파일이다. 디버거, 서비스 등을 이용해 DLL, SYS 파일도 실행 가능!

![13-1](https://user-images.githubusercontent.com/26838115/44969169-d7744580-af86-11e8-8702-d03ce2dc3dc6.png)

> notepad.exe 파일이 메모리에 로딩되는 모습

DOS header부터 Section header까지를 PE 헤더, 그 밑의 Section들을 합쳐서 PE 바디(Body)라고 한다. 파일에서는 offset으로, 메모리에서는 VA(Virtual Address, 절대주소)로 위치를 표현한다. 파일이 메모리에 로딩되면 모양이 달라진다(Section의 크기, 위치 등). 파일의 내용은 보통 코드(.text), 데이터(.data), 리소스(.rsrc) 섹션에 나뉘어서 저장된다. <br/><br/>

PE 헤더의 끝부분과 각 섹션의 끝에는 NULL padding이라고 불리우는 영역이 존재한다. 컴퓨터에서 파일, 메모리, 네트워크 패킷 등을 처리할 때 효율을 높이기 위해 최소 기본 단위 개념을 사용하는데, PE 파일에도 같은 개념이 적용된 것이다. 파일/메모리에서 섹션의 시작 위치는 각 파일/메모리의 최소 기본 단위의 배수에 해당하는 위치여야 하고, 빈 공간은 NULL로 채워버린다. 위의 그림을 보면 각 섹션의 시작 주소가 어떤 규칙에 의해 딱딱 끊어지는 걸 볼 수 있다! <br/><br/>




<br/>

---



### Issue #1

해결해야 할 이슈 : 

