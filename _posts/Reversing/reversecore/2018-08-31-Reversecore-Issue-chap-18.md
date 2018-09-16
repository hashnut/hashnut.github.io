---

layout : single
title : "Reversecore chap 18 - UPack PE 헤더 상세 분석"
categories : [Reversing]
tags : [reversecore, unsolved]
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

UPack은 IMAGE_OPTIONAL_HEADER.NumberOfRvaAndSizes 값 또한 변경한다. 이 값의 의미는 바로 뒤에 이어지는 IMAGE_DATA_DIRECTORY 구조체 배열의 원소 개수를 나타낸다. 정상적인 파일에서는 IMAGE_DATA_DIRECTORY 배열의 원소 개수는 10개 이지만, UPack에서는 A개로 변경된다.


![4](https://user-images.githubusercontent.com/26838115/45294184-7bd12b80-b535-11e8-90f3-bbb4447976a9.png)

> NumberOfRvaAndSizes

IMAGE_DATA_DIRECTORY 구조체 배열의 원소 개수는 이미 10개로 정해져 있지만, PE 스펙에 따르면 NumberOfRvaAndSizes 값을 배열의 원소 개수로 인정하도록 되어 있다. 따라서 UPack의 경우 IMAGE_DATA_DIRECTORY 구조체 배열의 뒤쪽 6개 원소들은 무시된다.

UPack은 IMAGE_OPTIONAL_HEADER.NumberOfRvaAndSizes 값을 A로 변경하여 LOAD_CONFIG 항목(파일 옵셋 D8 이후)부터는 사용하지 않는다. 그리고 바로 그 무시된 IMAGE_DATA_DIRECTORY 영역에 자신의 코드를 덮어써버린다!

![5](https://user-images.githubusercontent.com/26838115/45296981-c5724400-b53e-11e8-9e8e-45cc82a8912a.png)

![6](https://user-images.githubusercontent.com/26838115/45297012-e20e7c00-b53e-11e8-8176-fc8c57eed37e.png)

> 앞의 사진은 IMAGE_DATA_DIRECTORY 구조체 배열 전체 영역이고, 아래 사진은 UPack에서 무시되는 부분이다.

---

### IMAGE_SECTION_HEADER

IMAGE_SECTION_HEADER 구조체에서 프로그램 실행에 사용되지 않는 항목들에게 UPack 자신의 데이터를 기록한다. 이 기법 역시 PE 헤더에서 쓰이지 않는 영역에 자신의 코드와 데이터를 덮어쓰는 기법이다. 

---

### 섹션 겹쳐쓰기

UPack의 주요 특징 중 하나가 바로 섹션과 헤더를 마구 겹쳐쓰는 것이다. Stud_PE의 'Sections'탭을 통해 IMAGE_SECTION_HEADER를 살펴보자.

![7](https://user-images.githubusercontent.com/26838115/45297646-02d7d100-b541-11e8-8c99-cee49b86b0f8.png)

위의 그림을 보면 첫 번째 섹션과 세 번째 섹션의 파일 시작 옵셋(RawOffset) 값이 10으로 되어 있다. 옵셋 10은 헤더 영역인데 UPack에서는 이곳에서부터 섹션이 시작된다. 

그 다음 눈에 띄는 내용은 첫 번째 섹션과 세 번째 섹션의 파일 시작 옵셋(RawOffset)과 파일에서의 크기(RawSize)가 완전히 동일하다는 것이다. 단, 섹션의 메모리 시작(VirtualOffset) 항목과 메모리 크기(VirtualSize) 값이 서로 다르다. PE 스펙에 이렇게 하면 안 된다는 내용이 없으므로, PE 파일의 스펙에 어긋나는 것은 아니다!

위 사실을 종합하면, UPack은 PE 헤더, 첫 번째 섹션, 세 번쨰 섹션이 겹쳐 있다. 그림을 통해 자세히 알아보자.

![8](https://user-images.githubusercontent.com/26838115/45298219-d3c25f00-b542-11e8-997a-7c079ada053f.png)

섹션 헤더(IMAGE_SECTION_HEADER)에 정의된 값에 의해 PE 로더는 파일 옵셋 0 ~ 1FF 영역을 3군데 다른 메모리 위치(헤더, 첫 번째 섹션, 세 번째 섹션)에 각각 매핑한다. 이때, 같은 파일 이미지를 가지고 각각 다른 위치와 다른 크기의 메모리 이미지를 만들 수 있다는 사실에 주목해야 한다!

파일의 헤더(첫째/셋째 섹션)영역의 크기는 200이다. 반면, 두 번째 섹션(2nd Section) 영역의 크기(AE28)는 파일의 대부분을 차지할 정도로 크다. 바로 이곳에 원본 파일(notepad.exe)이 압축되어 있다. 

또 하나 주목해야 하는 부분은 메모리에서의 첫 번째 섹션 영역이다. 섹션의 메모리 크기는 14000이다. 이는 원본 파일(notepad.exe)의 Size of Image와 같은 값이다. 즉 두 번째 섹션에 압축된 파일 이미지를 첫 번째 섹션에 (notepad의 메모리 이미지) 그대로 압축해제하는 것이다. 참고로 notepad.exe 원본은 3 개의 섹션이 있다. 이를 하나의 섹션에 풀어내는 것이다.

![9](https://user-images.githubusercontent.com/26838115/45298224-da50d680-b542-11e8-9625-c4261a864203.png)

다시 한 번 정리하면 메모리 두 번쨰 섹션 영역에 압축된 notepad가 들어 있고, 압축이 풀리면서 첫 번째 섹션 영역에 기록된다. 중요한 건 notepad.exe(원본 파일)의 메모리 이미지가 통쨰로 풀리기 때문에 프로그램이 정상적으로 실행될 수 있다(주소가 정확히 일치하게 된다).

---

### RVA to RAW

각종 PE 유틸리티들이 UPack 파일의 RVA -> RAW 변환에 어려움을 겪었다('잘못된 메모리 참조에 의한 비정상 종료'를 당함). 

![10](https://user-images.githubusercontent.com/26838115/45298781-80e9a700-b544-11e8-9b68-63a646db0cfa.png)

위 사진을 보면, AddressOfEntryPoint의 값이 00001018임을 알 수 있다. 변환 공식을 이용하면(RAW - PointerToRawData = RVA - VirtualAddress), RAW(파일 옵셋) 값은 28임을 알 수 있다.

![11](https://user-images.githubusercontent.com/26838115/45298883-ce661400-b544-11e8-832c-908e308c9f90.png)

Hex Editor로 해당 영역을 살펴보면, 코드가 아니라 (ordinal:010B) "LoadLibraryA" 문자열 영역임을 알 수 있다. 이는 UPack의 트릭인데(UPack은 미끼를 던져분 것이고.. 초짜 해커는 그것을..), 비밀은 바로 첫 번째 섹션의 PointerToRawData 값 10에 있다.

일반적으로 섹션 시작의 파일 옵셋을 가리키는 PointerToRawData 값은 FileAlignment의 배수가 되어야 한다. UPack의 FileAlignment는 200이므로 PointerToRawData 값은 0, 200, 400, 600 등의 값을 가져야 한다. 따라서 PE 로더는 첫 번째 섹션의 PointerToRawData(10)가 FileAlignment(200)의 배수가 아니므로 강제로 배수에 맞춰서 인식한다(이 경우는 0). 이것이 바로 UPack 파일이 정상적으로 실행은 되지만, 많은 PE 관련 유틸리티에서 에러가 발생했던 이유이다.

정상적인 RVA -> RAW 변환은 아래와 같다.

~~~
RAW = 1018 - 1000 + 0 = 18
* PointerToRawData = 0으로 인식
~~~

디버거로 해당 영역의 코드를 살펴보자.

![12](https://user-images.githubusercontent.com/26838115/45299231-dd999180-b545-11e8-815f-fae33c749c82.png)

이제 정상적인 RVA -> RAW 변환을 할 수 있게 되었다!


---

### Import Table(IMAGE_IMPORT_DESCRIPTOR array)

UPack의 Import Table은 매우 특이하다. Hex Editor를 이용해서 IMAGE_IMPORT_DESCRIPTOR 구조체를 살펴보자. 먼저 Directory Table에서 Import Directory Table(IMAGE_IMPORT_DESCRIPTOR 구조체 배열) 주소를 얻어야 한다. 

![13](https://user-images.githubusercontent.com/26838115/45299395-70d2c700-b546-11e8-8781-97f62820543f.png)

위의 그림은 Data_Directory의 Import Table인데, 8바이트의 data 중 앞의 4 바이트가 Import Table의 주소(RVA), 뒤의 4바이트가 Import Table의 크기(Size)를 나타낸다. 따라서 위의 그림을 보면 RVA는 000271EE이다.

RVA -> RAW 변환을 거치면, RAW(파일 옵셋)의 값이 ;; 임을 알 수 있다.

~~~
RVA - VirtualAddress = RAW - PointerToRawData
RAW = RVA(271EE) - VirtualOffset(27000) + RawOffset(0) = 1EE
// 주의 : 3rd Section의 RawOffset 값이 10이 아니라 0으로 강제 변환되는 사실을 꼭 이해해야 한다! Section의 크기는 FileAlignment의 배수값을 가져야 하므로, 이에 맞게 강제로 0으로 조정되는 것임!
~~~

Hex Editor로 파일 옵셋 1EE를 살펴보자.

![14](https://user-images.githubusercontent.com/26838115/45299790-914f5100-b547-11e8-99c9-235a83d4ceba.png)

여기에 바로 트릭이 숨겨져 있다!

먼저 IMAGE_IMPORT_DESCRIPTOR 구조체 정의를 보고 난후 계속 진행해 보자(구조체 크기는 14 바이트).

~~~
typedef struct _IMAGE_IMPORT_DESCRIPTOR {
    union {
        DWORD   Characteristics;            // 0 for terminating null import descriptor
        DWORD   OriginalFirstThunk;         // RVA to original unbound IAT (PIMAGE_THUNK_DATA)
    } DUMMYUNIONNAME;
    DWORD   TimeDateStamp;                  // 0 if not bound,
                                            // -1 if bound, and real date\time stamp
                                            //     in IMAGE_DIRECTORY_ENTRY_BOUND_IMPORT (new BIND)
                                            // O.W. date/time stamp of DLL bound to (Old BIND)

    DWORD   ForwarderChain;                 // -1 if no forwarders
    DWORD   Name;
    DWORD   FirstThunk;                     // RVA to IAT (if bound this IAT has actual addresses)
} IMAGE_IMPORT_DESCRIPTOR;
~~~

PE 스펙에 따르면 Import Table은 IMAGE_IMPORT_DESCRIPTOR 구조체 배열로 이루어지고 마지막은 NULL 구조체로 끝나야 한다.

바로 위의 그림을 보면 선택된 영역이 Import Table을 나타내는 IMAGE_IMPORT_DESCRIPTOR 구조체 배열이다. 1EE ~ 201 옵셋까지 첫 번째 구조체이며, 그 뒤는 두 번째 구조체도 아니고, 그렇다고(Import Table의 끝을 나타내는) NULL 구조체도 아니다.


얼핏 보면 이는 분명 PE 스펙에 어긋난 듯이 보인다. 하지만 위의 그림에서 200 옵셋 이상은 세 번째 섹션 메모리에 매핑되지 않는다.

![16](https://user-images.githubusercontent.com/26838115/45300174-9cef4780-b548-11e8-844c-a071ef342047.png)

> 세 번쨰 섹션의 RawOffset은 10, RawSize는 1FF임을 알아두자.


![15](https://user-images.githubusercontent.com/26838115/45300145-877a1d80-b548-11e8-917a-bc8b9f0a8b90.png)

세 번째 섹션이 메모리에 로딩될 때 파일 옵셋 0 ~ 1FF 영역이 메모리 주소 27000 ~ 271FF에 매핑되고 (세 번째 섹션의 남은 메모리 영역인) 27200 ~ 28000 영역은 NULL로 채워진다. 해당 영역을 디버거로 확인해 보자.

![17](https://user-images.githubusercontent.com/26838115/45300359-15ee9f00-b549-11e8-84f8-ca040a843c64.png)

정확하게 010271FF까지만 매핑되고, 01027200 이후부터는 NULL로 채워진 걸 확인할 수 있다.

다시 PE 스펙의 Import Table 조건으로 돌아가서 01027102 주소 이후부터 NULL 구조체가 나타난다고 본다면 PE 스펙에 어긋나지 않는 셈이다. 이것이 바로 섹션을 이용한 UPack의 트릭이다. 파일로 볼 떄는 Import Table이 깨진 것처럼 보이지만, 실제 메모리에서는 Import Table이 정확히 나타나는 것이다.

대부분의 PE 유틸리티들이 파일에서 Import Table을 읽을 때 이트릭에 걸려 잘못된 주소를 쫓아가다가 메모리 참조 에러로 죽어버린다(굉장한 트릭임!).

---

### IAT(Import Address Table)

UPack이 어떤 DLL에서 어떤 API를 임포트하는지 실제로 IAT를 따라가서 확인해 보자. IMAGE_IMPORT_DESCRIPTOR 구조체와 위의 Hex Editor에서 확인한 정보를 조합하면 다음과 같다.

offset | member | RVA
|:----|:----|:----|
1EE | OriginalFirstThunk(INT) | 0
1FA | Name | 2
1FE | FirstThunk(IAT) | 11E8

먼저 Name의 RVA 값은 2 이고, 이는 Header 영역에 속해 있다(첫 번째 섹션이 RVA 1000에서 시작하기 때문).

헤더 영역에서는 그냥 RVA와 RAW 값이 같으므로 Hex Editor로 파일 옵셋(RAW) 2를 살펴보자.


![18](https://user-images.githubusercontent.com/26838115/45300936-a5e11880-b54a-11e8-999f-4eecfb8f7cde.png)

"KERNEL32.DLL" 문자열을 확인할 수 있다. 이 위치는 DOS 헤더(IMAGE_DOS_HEADER)의 사용되지 않는 영역이라서 UPack이 이곳에 Import DLL 이름을 써 둔 것이다(알뜰한 Upack ^^). DLL 이름을 알았으니 어떤 API를 임포트하는지 알아보자.

보통은 OriginalFirstThunk(INT)를 쫓아가면 API 이름 문자열이 나타나지만, UPack과 같이 OriginalFirstThunk(INT)가 0일 때는 FirstThunk(IAT) 값을 따라가도 상관 없다(INT, IAT 둘 중 어느 한쪽에서만 API 이름 문자열이 나타나면 된다). Hex Editor에 의하면 IAT 값 11E8은 첫 번째 섹션 영역이므로, 변환을 하면 RAW가 1E8임을 알 수 있다.

![19](https://user-images.githubusercontent.com/26838115/45301202-6bc44680-b54b-11e8-8e25-8d44f7635fa6.png)

위 그림의 영역은 IAT이면서 동시에 INT 역할도 하고 있다. 즉, 이곳은 Name Pointer(RVA) 배열이고 배열의 끝은 NULL이다. 또 2개의 API를 임포트하는 것을 알 수 있다. 각각 RVA 28과 BE이다.

이 RVA 위치에 Import 함수의 [ordinal + 이름 문자열]이 있다. 모두 헤더 영역이므로 RVA 값은 RAW 값과 동일하다.

![20](https://user-images.githubusercontent.com/26838115/45301323-b0e87880-b54b-11e8-8691-c8830c8c4f8a.png)

![21](https://user-images.githubusercontent.com/26838115/45301366-c52c7580-b54b-11e8-879c-96fdeff23d64.png)


위 그림을 통해 'LoadLibraryA'와 'GetProcAddress' API를 임포트한다는 것을 알 수 있다. 이 두 함수는 원본 파일의 IAT를 구성할 때 편리하기 때문에 일반적인 패커에서도 많이 임포트해서 사용되고 있다.



<br/>

---



### Issue #1

- 해결해야 할 이슈 : 

위의 **섹션 겹쳐쓰기** 파트에서, 두 번째 섹션에 압축된 파일 이미지를 첫 번째 섹션에 그대로 압축해제를 한다. 이때, 메모리에서 첫 번째 섹션은 원본 파일(notepad.exe)의 Size of Image와 같다. 그렇다면, 2nd, 3rd Section도 메모리에 로딩될텐데, 이는 불필요하게 메모리를 많이 소모하는 문제를 발생시킬 수 있는 것인가? 즉, UPack에서 파일 크기를 줄이는 기능의 trade-off는 메모리 사용량의 증가인가? 아니면 해당 메모리 영역은 실제로 아예 사용을 하지 않는 것이고, Offset만 표기된 것인가?

