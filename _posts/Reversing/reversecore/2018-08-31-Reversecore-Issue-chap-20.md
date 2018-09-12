---

layout : single
title : "Reversecore chap 20 - 인라인 패치 실습"
categories : [Reversing]
tags : [reversecore]
comment : true

---

### '리버싱 핵심 원리'의 내용 및 이슈들과 해결책을 다룹니다.

---

<br/>


실행 압축되거나 암호화된 파일을 패치할 때 자주 사용되는 인라인 패치(Inline Patch) 기법을 실습해 보자.


---

### 인라인 패치

인라인 코드 패치(Inline Code Patch) 혹은 줄여서 인라인 패치(Inline Patch)라고 하는 기법은 원하는 코드를 직접 수정하기 어려울 때 간단히 코드 케이브(Code Cave)라고 하는 패치 코드를 삽입한 후 실행해 프로그램을 패치시키는 기법이다. 주로 대상 프로그램이 실행 압축(혹은 암호화)되어 있어서 파일을 직접 수정하기 어려운 경우 많이 사용하는 기법이다.



