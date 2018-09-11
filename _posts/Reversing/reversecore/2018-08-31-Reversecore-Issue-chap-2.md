---

layout : single
title : "Reversecore chap 2 - Hello World! 리버싱"
categories : [Reversing]
tags : [reversecore]
comment : true

---

### '리버싱 핵심 원리'의 내용 및 이슈들과 해결책을 다룹니다.

---

<br/>


- **1.** **EP**(Entry Point)

**EP**(Entry Point)란 Windows 실행 파일 (EXE, DLL, SYS등)의 코드 시작점을 의미한다. 프로그램이 실행될 때 CPU에 의해 가장 먼저 실행되는 코드 시작 위차라고 생각하면 된다.

- **2.** **디버거 명령어**

명령어 |단축키 |설명 
|:------|:-------|:------|
Go to | [Ctrl+G] | 원하는 주소로 이동(코드/메모리를 확인할 때 사용. 실행되는 것은 아님)
Execute till Cursor | [F4] | cursor 위치까지 실행 (디버깅하고 싶은 주소까지 바로 갈 수 있음)
Set/Reset Breakpoint | [F2] | BP 설정 / 해제
Run | [F9] | 실행 (BP가 걸려있으면 그곳에서 실행이 정지됨)
Preview CALL/JMP address | [Enter] | 커서가 CALL?JMP 등의 명령어에 위치해 있다면, 해당 주소를 따라가서 보여줌. <br/> (실행되는 것이 아님. 간단히 함수 내용을 확인할 때 유용함)

- **3.** **원하는 코드를 빨리 찾는 방법**

- **문자열 검색 방법**

***마우스 우측 메뉴 - Search for - All referenced text strings*** <br/><br/>
해당 방법을 이용하여, 원하는 문자열이 어디에서 **CALL, JMP**되는지 등등을 파악할 수 있다.
VC++에서는 static 문자열을 기본적으로 유니코드(UNICODE)형식으로 저장한다. static 문자열이란 프로그램 내부에 하드코딩(Hard Coding)되어 있는 문자열을 의미한다.

- **API 검색 방법 (1) - 호출 코드에 BP**

***마우스 우측 메뉴 - Search for - All intermodular calls*** <br/><br/>
코드에서 사용된 API 호출 목록을 보고 프로그램에서 코드를 뽑아낸다!

- **API 검색 방법 (2) - API 코드에 직접 BP**

OllyDbg가 모든 실행 파일에 대해서 API 함수 호출 목록을 추출할 수 있는 것은 아님! Packer/Protector을 사용하여 실행파일을 압축(또는 보호)해버리면, 파일 구조가 변경되어 OllyDbg에서 API 호출 목록이 보이지 않는다.(디버깅이 어려워짐)<br/>

이런 경우에는 프로세스 메모리에서 로딩된 라이브러리(DLL 코드)에 직접 BP를 걸어 볼 수 있다. API라는 것은 OS에서 제공한 함수이고, 실제로 API는 **C:\Windows\system32** 폴더에 **XXX.dll** 파일 내부에 구현되어 있다. (예: kernel32.dll, user32.dll 등) 간단히 말해서 우리가 만든 프로그램이 어떤 의미 있는 일(각종 I/O)를 하려면 반드시 OS에서 제공된 API를 사용해서 OS에게 요청해야 하고, 그 API가 실제 구현된 시스템 DLL 파일들은 우리 프로그램의 프로세스 메모리에 로딩되어야 한다. <br/>

OllyDbg에서는 **View - Memory** (단축기 [Alt+M])메뉴에서 확인 가능하다.

> Packer(Run Time Packer)
>
>실행 압축 유틸리티. 실행 파일의 코드, 데이터, 리소스 등을 압축시킨다. 일반 압축 파일과 다른 점은 실행 압축된 파일 그 자체도 실행 파일이라는 것이다.

> Protector
>
>실행 압축 기능 외에 파일과 그 프로세스를 보호하려는 목적으로 anti-debugging, anti-emulating, anti-dump 등의 기능을 추가한 유틸리티이다. Protector를 상세 분석하려면 높은 수준의 리버싱 지식이 요구된다.

- **4.** **Assembly 기초 명령어**

명령어 | 설명
|:--------|:-------|
CALL XXXX | XXXX 주소의 함수를 호출
JMP XXXX | XXXX 주소로 점프
PUSH XXXX | 스택에 XXXX 저장
RETN | 스택에 저장된 복귀 주소로 점프


- **5.** **기초 용어**

용어 | 설명
|:------|:------|
VA(Virtual Address) | 프로세스의 가상 메모리
OP code(Operation code) | CPU 명령어 (바이트 code)
PE(Portable Executable) | Windows 실행 파일 (EXE, DLL, SYS 등)

- **6.** **Stub code**

Stub code는 사용자가 입력한 코드가 아니라 컴파일러가 임의로 추가시킨 코드이다. 컴파일러별로 자신의 특성에 맞게 정형화된 Stub code를 추가시킨다. <br/>
특히 EP 코드 영역에 Stub Code를 추가한다. (이것을 따로 StartUp Code라고도 부른다) 초보자 과정에서는 이 코드가 Stub Code인지 사용자 코드인지 구분만 할 줄 알면 된다!



<br/>

---



### Issue #1

해결해야 할 이슈 : 

