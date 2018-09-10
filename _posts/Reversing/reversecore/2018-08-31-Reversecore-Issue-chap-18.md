---

layout : single
title : "Reversecore Issue chap 18 - UPack PE 헤더 상세 분석"
categories : [Reversing]
tags : [reversecore]
comment : true

---

### '리버싱 핵심 원리'의 내용 및 이슈들과 해결책을 다룹니다.

---

<br/>


UPack 패커를 이용한 notepad.exe의 PE 헤더 구조를 파헤쳐 보자.


---

### 준비물

- UPack

- Stud_PE

- notepad.exe (32bit ver.)

---


### UPack의 PE 헤더 분석


### 헤더 겹쳐쓰기

이는 다른 패커에서도 많이 쓰이는 기법이다. 이는 MZ 헤더(IMAGE_DOS_HEADER)와 PE 헤더(IMAGE_NT_HEADERS)를 교묘하게 겹쳐쓰는 것이다. 헤더를 겹쳐씀으로 해서 헤더 공간을 절약할 수 있다. 부가적으로 복잡성을 증가시켜 분석을 어렵게 만드는 효과도 있다.

Stud_PE를 이용해서 MZ 헤더를 살펴보자. ('Headers' 탭의 [Basic HEADERS tree view in hexeditor] 버튼)


MZ 헤더(IMAGE_DOS_HEADER)에서는 아래 2가지 멤버가 중요하다.

~~~
e_magic   : Magic number = 4D5A('MZ')
e_lfanew  : File address of new exe header
~~~

그 외 나머지는 프로그램 실행에 아무 의미가 없다.

e_lfanew의 값에 따라서 IMAGE_NT_HEADERS의 시작 위치가 결정된다. 보통 정상적인 프로그램에서는 e_lfanew의 값이(빌드 환경마다 다르긴 함) 아래와 같다.

~~~
e_lfanew = MZ 헤더 크기(40) + DOS Stub 크기(가변 : VC++의 경우 보통 A0) = E0
~~~

UPack에서는 e_lfanew 값이 10이다. PE 스펙에 어긋나진 않고, 스펙 자체의 허술함을 이용한 것이다. 이런 식으로 MZ 헤더와 PE 헤더의 겹쳐쓰기가 가능해진다.

![1](https://user-images.githubusercontent.com/26838115/45293028-f7c97480-b531-11e8-82fe-7606243a5572.png)

> e_lfanew의 값이 00000010 임을 확인할 수 있다.

---

### IMAGE_FILE_HEADER.SizeOfOptionalHeader

IMAGE_FILE_HEADER.SizeOfOptionalHeader의 값을 변경한다. 이 값을 조작하여 헤더 안에 디코딩 코드를 삽입하기 위해서이다.

값의 의미는 PE 헤더에서 바로 뒤따르는 IMAGE_OPTIONAL_HEADER 구조체의 크기이다(E0). UPack은 이 값을 0148로 변경한다.

![2](https://user-images.githubusercontent.com/26838115/45293205-71616280-b532-11e8-9c75-52ee173c37a7.png)

IMAGE_OPTIONAL_HEADER는 말 그대로 '구조체'이기 때문에 PE 32 파일 포맷에서 크기는 이미 E0로 결정되어 있다. 이때, PE File Format 설계자들이 IMAGE_OPTIONAL_HEADER 구조체의 크기를 따로 입력하게 한 이유는, PE 파일의 형태에 따라 각각 다른 IMAGE_OPTIONAL_HEADER 형태의 구조체를 바꿔 낄 수 있도록 하기 위해서이다. 따라서, IMAGE_OPTIONAL_HEADER의 종류가 여러 개이므로 구조체의 크기를 따로 입력할 필요가 있는 것이다(예: 64비트용 PE32+의 IMAGE_OPTIONAL_HEADER 구조체 크기는 F0).

SizeOfOptionalHeader의 또 다른 의미는 섹션 헤더(IMAGE_SECTION_HEADER)의 시작 옵셋을 결정하는 것이다.

PE 헤더를 그냥 보면 IMAGE_OPTIONAL_HEADER에 이어서 IMAGE_SECTION_HEADER가 나타나는 듯이 보인다. 하지만 실제로는 IMAGE_OPTIONAL_HEADER 시작 옵셋에 SizeOfOptionalHeader 값을 더한 위치(옵셋)부터 IMAGE_SECTION_HEADER가 나타난다.

UPack에서는 SizeOfOptionalHeader의 값을 148로 설정함으로써 IMAGE_SECTION_HEADER가 옵셋 170부터 시작하게 된다(IMAGE_OPTIONAL_HEADER 시작 옵셋(28) + SizeOfOptionalHeader(148) = 170).

UPack은 왜 SizeOfOptionalHeader 값을 변경했을까? SizeOfOptionalHeader 값을 늘리면 IMAGE_OPTIONAL_HEADER와 IMAGE_SECTION_HEADER 사이에 추가적인 공간을 확보할 수 있는데, 바로 이 영역에 디코딩 코드를 추가 하기 위함이다.

![3](https://user-images.githubusercontent.com/26838115/45294075-2006a280-b535-11e8-894e-59f7df74fac9.png)

> 디코딩 코드

---

### IMAGE_OPTIONAL_HEADER.NumberOfRvaAndSizes








<br/>

---



### Issue #1

- 해결해야 할 이슈 : 

