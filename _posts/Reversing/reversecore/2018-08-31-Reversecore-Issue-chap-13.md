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


**PE(Portable Executable)**파일은 Windows 운영체제에서 사용되는 실행 파일 형식이다. 기존 UNIX에서 사용되는 COFF(Common Object File Format)를 기반으로 Microsoft에서 만들었다. 애초에는 다른 운영체제에 이식성을 좋게 하려고 만들었으나 현재는 Windows 계열의 OS에서만 사용되고 있다. <br/><br/>


1. **PE File Format**

본격적으로 PE파일의 종류를 알아보자.

종류 | 주요 확장자
|:-----|:------|
실행 계열 | EXE, SCR
라이브러리 계열 | DLL, OCX, CPL, DRV
드라이버 계열 | SYS, VXD
오브젝트 파일 계열 | OBJ

엄밀히 얘기하면 OBJ(오브젝트) 파일을 제외한 모든 것은 실행 가능한 파일입니다. 디버거, 서비스 등을 이용해 DLL, SYS 파일도 실행 가능!

![13-1](https://user-images.githubusercontent.com/26838115/44969169-d7744580-af86-11e8-8702-d03ce2dc3dc6.png)

> notepad.exe 파일을 헥스 에디터 HxD로 로드





<br/>

---



### Issue #1

해결해야 할 이슈 : 

