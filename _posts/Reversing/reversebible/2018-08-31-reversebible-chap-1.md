---

layout : single
title : "Reversebible #1 - 리버스 엔지니어링만을 위한 어셈블리"
categories : [Reversing]
tags : [reversebible]
comment : true

---

### '리버스 엔지니어링 바이블'의 핵심 내용 및 이슈들과 해결책을 다룹니다.

---

<br/>

### C 언어로 어셈블리 사용해 보기!

어셈블리로 두 변수의 값을 더하는 코드를 짜 보자!

~~~
#include <windows.h>
#include <stdio.h>

int Plus(int a, int b) {
    return a +b;
}

_declspec(naked) PlusAsm(int a, int b) {

    _asm {
        mov ebx, dword ptr ss:[esp+8]
        mov edx, dword ptr ss:[esp+4]
        add edx, ebx
        mov eax, edx
        retn
    };
}

void main(int argc, char *argv[]) {
    int value = Plus(3, 4);
    printf("value: %d\n", value);

    int value2 = PlusAsm(3,4);
    printf("value2: %d\n", value2);

    return;
}
~~~

> _declspec(naked) <br/>
함수의 엔트리 포인트에서는 개발자가 작성한 코드의 첫 줄이 등장하는 것이 아니라 컴파일러가 자체적으로 생성한 스택을 확보하는 작업에 대한 코드부터 등장한다. naked는 그것을 방지하기 위한 접두어다. naked를 사용하면 이제부터 이 함수 안에서는 부수적인 코드를 전혀 사용하지 않을 것이라고 지정하게 되며, 컴파일러는 이 함수 안에 어떤 자체적인 코드도 생성하지 않는다. 심지어 리턴값조차 컴파일러가 생성하지 않는다! 따라서 naked 함수를 만들려면 개발자가 스택이나 변수 할당, 레지스터 사용 등의 모든 처리 내용을 모두 작성해야 한다. 위의 예제처럼 순수 어셈블리로 만들어진 코드를 작성할 때 이와 같이 naked를 사용한다!







<br/>

---



### Issue #1

해결해야 할 이슈 : 

