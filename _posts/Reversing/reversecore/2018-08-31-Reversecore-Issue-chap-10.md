---

layout : single
title : "Reversecore Issue chap 10"
categories : [Reversing]
tags : [reversecore]
comment : true

---

### '리버싱 핵심 원리'의 내용 및 이슈들과 해결책을 다룹니다.

---

<br/>


1. 챕터 10은 함수 호출 규약(Calling Convention)에 대해 다룬다. 이는 '함수를 호출할 때 파라미터를 어떤 식으로 전달하는지'에 대한 일종의 약속이다.

2. 함수 호출 규약은 **cdecl, stdcall, fastcall**으로 크게 3 가지로 나뉜다.

3. 간단한 용어 설명

   - Caller(호출자) - 함수를 호출한 쪽
   - Callee(피호출자) - 호출을 당한 함수

4. **cdecl**

   cdecl방식은 주로 C 언어에서 사용되는 방식이며, Caller에서 스택을 정리하는 특징을 가지고 있다.

	   #include "studio.h"
	   
	   int add(int a, int b)
	   
	   {
	   
	   return (a + b);
	   
	   }
	   
	   int main (int argc, char * argv[])
	   
	   {
	   
	   return add(1, 2);
	   
	   }


	![10-1](https://user-images.githubusercontent.com/26838115/44953107-4dff3d80-aeca-11e8-9aec-cebad8f1db3f.png)

	0040101C 주소를 보면, **ESP**에 8을 더하여 Caller인 main()함수가 자신이 스택에 입력한 함수 파라미터를 직접 정리	 하고 있다. 
	이 방식이 바로 cdecl 방식으로, 장점은 C 언어의 printf() 함수와 같이 가변 길이 파라미터를 전달할 수 있다는 것이다. 

5. **stdcall**

   **stdcall**방식은 Win32 API 에서 사용되며, Callee에서 스택을 정리하는 것이 특징이다. (C 언어는 기본적으로 cdecl방식이다.) **stdcall**방식으로 컴파일 하고 싶을 때는 '_stdcall'키워드를 붙여주면 된다.


	   #include "studio.h"
	   
	   int _stdcall add(int a, int b)
	   
	   {
	   
	   return (a + b);
	   
	   }
	   
	   int main (int argc, char * argv[])
	   
	   {
	   
	   return add(1, 2);
	   
	   }


    ![10-2](https://user-images.githubusercontent.com/26838115/44953114-7d15af00-aeca-11e8-905a-1de6283f51ee.png)

	0040100A의 주소를 보면 **RETN 8**라는 명령어를 통해 호출된 함수(Callee)내부에서 스택을 정리해준다는 것을 알 수 있다.

6. Win32 API는 C 언어로 된 라이브러리이지만 기본 cdecl 방식이 아닌 stdcall 방식을 사용한다. 이는 C 이외의 다른 언어 (Delphi(Pascal), Visual Basic 등)에서 API를 직접 호출할 때 호환성을 좋게 하기 위한 것이다!

7. **fastcall**

   **fastcall**방식은 기본적으로 stdcall 방식과 같지만, 함수에 전달하는 파라미터 일부(2개까지)를 스택 메모리가 아닌 레지스터를 이용하여 전달한다는 것이 특징이다. 예를 들어, 어떤 함수의 파라미터가 4개이면 앞의 두 개의 파라미터는 각각 **ECX**,**EDX** 파라미터를 이용하여 전달한다

   **fastcall**의 장점은 좀 더 빠른 함수 호출이 가능하다는 것이다. 하지만 **ECX**, **EDX**레지스터 관리를 위해 추가적인 오버헤드가 필요한 경우가 있다. 또한 앞의 레지스터들을 다른 용도로 쓸 경우 따로 백업을 해야하는 절차가 필요하다.

<br/>

---



### Issue #1

해결해야 할 이슈 : 

