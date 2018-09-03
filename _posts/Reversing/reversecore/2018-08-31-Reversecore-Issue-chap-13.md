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

	- **VA & RVA**

	VA(Virtual Address)는 프로세스 가상 메모리의 절대주소를 말하며, RVA(Relative Virtual Address)는 어느 기준 위치(ImageBase)에서부터의 상대주소를 말한다. VA와 RVA의 관계는 다음 식과 같다.

	> RVA + ImageBase = VA

	PE 헤더 내의 정보는 RVA 형태로 된 것이 많다. 그 이유는 PE 파일(주로 DLL)이 프로세스 가상 메모리의 특정 위치에 로딩되는 순간 이미 그 위치에 다른 PE 파일(DLL)이 로딩되어 있을 수 있기 때문이다. 그럴 때 재배치(Relocation)과정을 통해서 비어 있는 다른 위치에 로딩되어야 하는데, 만약 PE 헤더 정보들이 VA(Virtual Address, 절대주소)로 되어 있다면 정상적인 엑세스가 이루어지지 않을 것이다. 그러므로 RVA(Relative Virtual Address, 상대주소)로 해두면 Relocation이 발생해도 기준위치에 대한 상대주소가 변하지 않기 때문에 아무런 문제없이 원하는 정보에 엑세스할 수 있는 것이다.

	> 32비트 Windows OS에서 각 프로세스에게는 4GB 크기의 가상 메모리가 할당된다. 따라서 프로세스에서 VA 값의 범위는 00000000 ~ FFFFFFFF까지 이다.

2. PE 헤더




<br/>

---



### Issue #1

해결해야 할 이슈 : 

