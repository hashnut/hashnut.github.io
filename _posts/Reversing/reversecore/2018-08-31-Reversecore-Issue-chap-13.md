---

layout : single
title : "Reversecore chap 13 - PE File Format"
categories : [Reversing]
tags : [reversecore]
comment : true

---

### '리버싱 핵심 원리'의 내용 및 이슈들과 해결책을 다룹니다.

---

<br/>


**PE(Portable Executable)** 파일은 Windows 운영체제에서 사용되는 실행 파일 형식이다. 기존 UNIX에서 사용되는 COFF(Common Object File Format)를 기반으로 Microsoft에서 만들었다. 애초에는 다른 운영체제에 이식성을 좋게 하려고 만들었으나 현재는 Windows 계열의 OS에서만 사용되고 있다. <br/><br/>


---

## **PE File Format**

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


---

## PE 헤더

- **DOS Header**

Microsoft는 PE File Format을 만들 때 당시에 널리 사용되던 DOS 파일에 대한 하위 호환성을 고려해서 만들었다. 그 결과로 PE 헤더의 제일 앞부분에는 기존 DOS EXE Header를 확장시킨 IMAGE_DOS_HEADER 구조체가 존재한다.

IMAGE_DOS_HEADER 구조체	

~~~
typedef struct_IMAGE_DOS_HEADER {
	WORD	e_magic;		// DOS signature :4D5A ("MZ")
	WORD	e_cblp;
	WORD	e_cp;
	WORD	e_crlc;
	WORD	e_parhdr;
	WORD	e_minalloc;
	WORD	e_maxalloc;
	WORD	e_ss;
	WORD	e_sp;
	WORD	e_csum;
	WORD	e_ip;
	WORD	e_cs;
	WORD	e_lfarlc;
	WORD	e_ovno;
	WORD	e_res[4];
	WORD	e_oemid;
	WORD	e_res2[10];
	WORD	e_lfanew;		// offset to NT header
		} IMAGE_DOS_HEADER, *PIMAGE_DOS_HEADER;
~~~

IMAGE_DOS_HEADER 구조체의 크기는 40h(64byte)이다. 이 구조체에서 꼭 알아두어야 할 중요한 멤버는 **e_magic**과 **e_lfanew**이다.

> e_magic : DOS signature (4D5A => ASCII 값 "MZ")
> e_lfanew : NT header의 옵셋을 표시 (파일에 따라 가변적인 값을 가짐)

모든 PE 파일은 시작 부분(e_magic)에 DOS signature("MZ")가 존재하고, e_lfanew 값이 가리키는 위치에 NT Header 구조체가 존재해야 한다(NT Header 구조체의 이름은 IMAGE_NT_HEADERS이며 나중에 소개된다).

> MZ는 Microsoft에서 DOS 실행 파일을 설계한 마크 주비코브스키라는 사람의 영문 이니셜이다! ~~전 세계에 자신의 이름을 새기신 갓 마크 센세...~~

![13-2](https://user-images.githubusercontent.com/26838115/44982233-25ea0a00-afb0-11e8-9a41-bc8cd9451b38.png)

IMAGE_DOS_HEADER 부분을 보면, PE 스펙에 맞게 파일 시작 2바이트는 4D5A이며, e_lfanew 값은 000000E8이라는 것을 알 수 있다. (리틀 엔디언 방식으로 저장했으므로, E8000000이 아닌 것을 숙지!) 시험 삼아 이 값들을 변경한 후 실행하면, 정상 실행이 되지 않는다.

- **DOS Stub**

DOS HEADER 밑에는 DOS stub이 존재한다. DOS Stub의 존재 여부는 옵션이며 크기도 일정하지 않다. (DOS Stub이 없어도 파일 실행에는 문제가 없음). DOS Stub은 코드와 데이터의 혼합으로 이루어져 있다. notepad.exe의 DOS Stub 코드이다.

![13-3](https://user-images.githubusercontent.com/26838115/44982279-529e2180-afb0-11e8-9e46-631ee9e786ba.png)

위 그림에서 파일 옵셋 40 ~ 4D 영역은 16비트 어셈블리 명령어이다. 32비트 Windows OS에서는 이쪽 명령어가 실행되지 않는다(PE 파일로 인식하기 떄문에 아예 이쪽 코드를 무시한다). notepad.exe 파일을 DOS 환경에서 실행하거나, DOS용 디버거(debug.exe)를 이용해서 실행하면 저 코드를 실행시킬 수 있다(DOS EXE 파일로 인식한다. 이들은 PE File Format을 모르므로...).

- **NT Header**

NT header 구조체 IMAGE_NT_DEADERS이다.

~~~
typedef struct _IMAGE_NT_HEADERS {
	DWORD Signature;			// PE Signature : 50450000 ("PE"00)
	IMAGE_FILE_HEADER FileHeader;
	IMAGE_OPTIONAL_HEADER32 OptionalHeader;
} IMAGE_NT_HEADERS32, *PIMAGE_NT_HEADERS32;
~~~

IMAGE_NT_HEADERS 구조체는 3개의 멤버로 되어 있는데, 제일 첫 멤버는 **Signature**로 5045000h("PE"00)값을 가진다. 그리고 FileHeader와 Optional Header 구조체 멤버가 있다. notepad.exe의 IMAGE_NT_HEADERS의 내용을 hex editor로 살펴보자.

![13-4](https://user-images.githubusercontent.com/26838115/44982793-005e0000-afb2-11e8-9883-8787073aa8e1.png)

IMAGE_NT_HEADERS 구조체의 크기는 F8이다(D0 8번째 칸까지).

- **NT Header - File Header**

파일의 개략적인 속성을 나타내는 IMAGE_FILE_HEADER 구조체이다.

~~~
typedef struct _IMAGE_FILE_HEADER {
	WORD	Machine;
	WORD 	NumbeOfSections;
	DWORD	TimeDateStamp;
	DWORD	PointerToSymbolTable;
	DWORD	NumberOfSymbols;
	WORD	SizeOfOptionalHeader;
	WORD	Characteristics;
} IMAGE_FILE_HEADER, *PIMAGE_FILE_HEADER;
~~~

IMAGE_FILE_HEADER 구조체에서 아래 4가지 멤버가 중요하다!

- **A. Machine**

Machine 넘버는 CPU별로 고유한 값이며 32비트 Intel x86 호환 칩은 14C의 값을 가진다. 아래는 winnt.h 파일에 정의된 Machine 넘버의 값들이다.

~~~
#define IMAGE_FILE_MACHINE_UNKNOWN           0
#define IMAGE_FILE_MACHINE_I386              0x014c  // Intel 386.
#define IMAGE_FILE_MACHINE_R3000             0x0162  // MIPS little-endian, 0x160 big-endian
#define IMAGE_FILE_MACHINE_R4000             0x0166  // MIPS little-endian
#define IMAGE_FILE_MACHINE_R10000            0x0168  // MIPS little-endian
#define IMAGE_FILE_MACHINE_WCEMIPSV2         0x0169  // MIPS little-endian WCE v2
#define IMAGE_FILE_MACHINE_ALPHA             0x0184  // Alpha_AXP
#define IMAGE_FILE_MACHINE_SH3               0x01a2  // SH3 little-endian
#define IMAGE_FILE_MACHINE_SH3DSP            0x01a3
#define IMAGE_FILE_MACHINE_SH3E              0x01a4  // SH3E little-endian
#define IMAGE_FILE_MACHINE_SH4               0x01a6  // SH4 little-endian
#define IMAGE_FILE_MACHINE_SH5               0x01a8  // SH5
#define IMAGE_FILE_MACHINE_ARM               0x01c0  // ARM Little-Endian
#define IMAGE_FILE_MACHINE_THUMB             0x01c2  // ARM Thumb/Thumb-2 Little-Endian
#define IMAGE_FILE_MACHINE_ARMNT             0x01c4  // ARM Thumb-2 Little-Endian
#define IMAGE_FILE_MACHINE_AM33              0x01d3
#define IMAGE_FILE_MACHINE_POWERPC           0x01F0  // IBM PowerPC Little-Endian
#define IMAGE_FILE_MACHINE_POWERPCFP         0x01f1
#define IMAGE_FILE_MACHINE_IA64              0x0200  // Intel 64
#define IMAGE_FILE_MACHINE_MIPS16            0x0266  // MIPS
#define IMAGE_FILE_MACHINE_ALPHA64           0x0284  // ALPHA64
#define IMAGE_FILE_MACHINE_MIPSFPU           0x0366  // MIPS
#define IMAGE_FILE_MACHINE_MIPSFPU16         0x0466  // MIPS
#define IMAGE_FILE_MACHINE_AXP64             IMAGE_FILE_MACHINE_ALPHA64
#define IMAGE_FILE_MACHINE_TRICORE           0x0520  // Infineon
#define IMAGE_FILE_MACHINE_CEF               0x0CEF
#define IMAGE_FILE_MACHINE_EBC               0x0EBC  // EFI Byte Code
#define IMAGE_FILE_MACHINE_AMD64             0x8664  // AMD64 (K8)
#define IMAGE_FILE_MACHINE_M32R              0x9041  // M32R little-endian
#define IMAGE_FILE_MACHINE_CEE               0xC0EE
~~~

- **B. NumberOfSections**

PE 파일은 코드, 데이터, 리소스 등이 각각의 섹션에 나뉘어서 저장된다. 이 때 NumberOfSections는 바로 그 섹션의 개수를 나타낸다. 그런데 이 값은 반드시 0보다 커야 한다. 또 정의된 섹션 개수와 실제 섹션이 다르면 실행 에러가 발생한다.

- **C. SizeOfOptionalHeader**

SizeOfOptionHeader 멤버는 바로 이 IMAGE_OPTIOANL_HEADER32 구조체의 크기를 나타낸다. IMAGE_OPTIOANL_HEADER32는 C언어의 구조체이기 떄문에 이미 그 크기가 결정되어 있다. 그런데 Windows의 PE 로더는 IMAGE_FILE_HEADER의 SizeOfOptionalHeader 값을 보고 IMAGE_OPTIOANL_HEADER32 구조체의 크기를 인식한다. <br/><br/>

PE32+ 형태의 파일인 경우에는 IMAGE_OPTIOANL_HEADER32 구조체 대신 IMAGE_OPTIOANL_HEADER64 구조체를 사용한다. 두 구조체의 크기는 다르기 때문에 SizeOfOptionalHeader 멤버에 구조체 크기를 명시하는 것이다.

> IMAGE_DOS_HEADER의 e_lfanew 멤버와 IMAGE_FILE_HEADER의 SizeOfOptioanlHeader의 멤버 때문에 일반적인(상식적인) PE 파일 형식을 벗어나는 일명 '꽈배기' PE 파일(PE Patch)을 만들 수 있다.

- **D. Characteristic**

파일의 속성을 나타내는 값으로, 실행이 가능한 형태인지(executable or not) 혹은 DLL 파일인지 등의 정보들이 bit OR 형식으로 조합된다.

~~~
#define IMAGE_FILE_RELOCS_STRIPPED           0x0001  // Relocation info stripped from file.
#define IMAGE_FILE_EXECUTABLE_IMAGE          0x0002  // File is executable  (i.e. no unresolved externel references).
#define IMAGE_FILE_LINE_NUMS_STRIPPED        0x0004  // Line nunbers stripped from file.
#define IMAGE_FILE_LOCAL_SYMS_STRIPPED       0x0008  // Local symbols stripped from file.
#define IMAGE_FILE_AGGRESIVE_WS_TRIM         0x0010  // Agressively trim working set
#define IMAGE_FILE_LARGE_ADDRESS_AWARE       0x0020  // App can handle >2gb addresses
#define IMAGE_FILE_BYTES_REVERSED_LO         0x0080  // Bytes of machine word are reversed.
#define IMAGE_FILE_32BIT_MACHINE             0x0100  // 32 bit word machine.
#define IMAGE_FILE_DEBUG_STRIPPED            0x0200  // Debugging info stripped from file in .DBG file
#define IMAGE_FILE_REMOVABLE_RUN_FROM_SWAP   0x0400  // If Image is on removable media, copy and run from the swap file.
#define IMAGE_FILE_NET_RUN_FROM_SWAP         0x0800  // If Image is on Net, copy and run from the swap file.
#define IMAGE_FILE_SYSTEM                    0x1000  // System File.
#define IMAGE_FILE_DLL                       0x2000  // File is a DLL.
#define IMAGE_FILE_UP_SYSTEM_ONLY            0x4000  // File should only be run on a UP machine
#define IMAGE_FILE_BYTES_REVERSED_HI         0x8000  // Bytes of machine word are reversed.
~~~

그 중, 0002h와 2000h의 값을 유심히 보자. PE 파일 중에 Characteristics 값에 0002h가 없는 경우(not executable)가 있을까? 답은..<br/><br/>

있다! 예를 들어 xxx.obj와 같은 object 파일 및 resource DLL 같은 파일을 들 수 있다. <br/><br/>

마지막으로 IMAGE_FILE_HEADER의 TimeDateStamp 멤버는 파일의 실행에 영향을 미치지 않는 값으로, 해당 파일의 빌드 시간을 나타낸 값이다. 개발 도구와 옵션에 따라 표기가 될 수도 있고 안 될 수도 있다. <br/><br/>

그럼 이제 Hex Editor 에서 notepad.exe의 IMAGE_FILE_HEADER 구조체를 확인해 보자.

![13-5](https://user-images.githubusercontent.com/26838115/44987060-d19b5600-afc0-11e8-901b-ae4ce4a02358.png)

위의 그림을 알아보기 쉽게 구조체 멤버로 표현해 보자.

~~~
[ IMAGE_FILE_HEADER ] - notepad.exe

offset		value		description
--------------------------------------------
000000EC		8664	machine
000000EE 		0006	number of sections
000000F0  	9D4727C2	time date stamp
000000F4	00000000	offset to symbol table
000000F8 	00000000	number of symbols
000000FC 		00F0 	size of optioanl header
000000FE 		0022 	characteristics
						IMAGE_FILE_RELOCS_STRIPPED
						IMAGE_FILE_EXECUTABLE_IMAGE
						IMAGE_FILE_LINE_NUMS_STRIPPED
						IMAGE_FILE_LOCAL_SYMS_STRIPPED
						IMAGE_FILE_32BIT_MACHINE
~~~


- **NT Header - Optional Header**

PE 헤더 구조체 중에서 가장 크기가 큰 IMAGE_OPTIONAL_HEADER32이다.

~~~
typedef struct _IMAGE_DATA_DIRECTORY {
    DWORD   VirtualAddress;
    DWORD   Size;
} IMAGE_DATA_DIRECTORY, *PIMAGE_DATA_DIRECTORY;

#define IMAGE_NUMBEROF_DIRECTORY_ENTRIES    16

//
// Optional header format.
//

typedef struct _IMAGE_OPTIONAL_HEADER {
    //
    // Standard fields.
    //

    WORD    Magic;
    BYTE    MajorLinkerVersion;
    BYTE    MinorLinkerVersion;
    DWORD   SizeOfCode;
    DWORD   SizeOfInitializedData;
    DWORD   SizeOfUninitializedData;
    DWORD   AddressOfEntryPoint;
    DWORD   BaseOfCode;
    DWORD   BaseOfData;

    //
    // NT additional fields.
    //

    DWORD   ImageBase;
    DWORD   SectionAlignment;
    DWORD   FileAlignment;
    WORD    MajorOperatingSystemVersion;
    WORD    MinorOperatingSystemVersion;
    WORD    MajorImageVersion;
    WORD    MinorImageVersion;
    WORD    MajorSubsystemVersion;
    WORD    MinorSubsystemVersion;
    DWORD   Win32VersionValue;
    DWORD   SizeOfImage;
    DWORD   SizeOfHeaders;
    DWORD   CheckSum;
    WORD    Subsystem;
    WORD    DllCharacteristics;
    DWORD   SizeOfStackReserve;
    DWORD   SizeOfStackCommit;
    DWORD   SizeOfHeapReserve;
    DWORD   SizeOfHeapCommit;
    DWORD   LoaderFlags;
    DWORD   NumberOfRvaAndSizes;
    IMAGE_DATA_DIRECTORY DataDirectory[IMAGE_NUMBEROF_DIRECTORY_ENTRIES];
} IMAGE_OPTIONAL_HEADER32, *PIMAGE_OPTIONAL_HEADER32;
~~~

아래에 후술될 값들은 파일 실행에 필수적이라서 잘못 세팅되면 파일이 정상 실행 되지 않는다!


- **1. Magic**

Magic 넘버는 IMAGE_OPTIONAL_HEADER32 구조체인 경우 10B, IMAGE_OPTIONAL_HEADER64 구조체인 경우 20B의 값을 가진다.

- **2. AddressOfEntryPoint**

AddressOfEntryPoint는 EP(Entry Point)의 RVA(Relative Virtual Address) 값을 가지고 있다. 이 값이야말로 프로그램에서 최초로 실행되는 코드의 시작 주소로, 매우 중요한 값이다.

- **3. ImageBase**

프로세스의 가상 메모리는 0 ~ FFFFFFFF 범위이다(32비트의 경우). ImageBase는 이렇게 광활한 메모리에서 PE 파일이 로딩되는 시작 주소를 나타낸다. <br/>
EXE, DLL 파일은 user memory 영역인 0~7FFFFFFF 범위에 로딩되고, SYS 파일은 kernel memory 영역인 80000000 ~ FFFFFFFF 범위에 로딩된다. 일반적으로 개발 도구들이 만들어내는 EXE 파일의 ImageBase 값은 00400000이고, DLL 파일의 ImageBase 값은 1000000이다(물론 다른 값도 지정 가능). PE 로더는 PE 파일을 실행시키기 위해 프로세스를 생성하고 파일을 메모리에 로딩한 후 EIP 레지스터 값을 ImageBase + AddressOfEntryPoint 값으로 세팅합니다.

- **4. SectionAlignment, FileAlignment**

PE 파일의 Body 부분은 섹션(Section)으로 나뉘어져 있다. 파일에서 섹션의 최소단위를 나타내는 것이 FileAlignment이고 메모리에서 섹션의 최소단위를 나타내는 것이 SectionAlignment이다(하나의 파일에서 FileAlignment와 SectionAlignment의 값은 같을 수도 있고 다를 수도 있음). 파일/메모리의 섹션 크기는 반드시 각각 FileAlignment/SectionAlignment의 배수가 되어야 한다.

- **5. SizeOfImage**

SizeOfImage는 PE 파일이 메모리에 로딩되었을 때 가상 메모리에서 PE Image가 차지하는 크기를 나타낸다. 일반적으로 파일의 크기와 메모리에 로딩된 크기는 다르다.

- **6. SizeOfHeader**

SizeOfHeader는 PE 헤더의 전체 크기를 나타냅니다. 이 값 역시 FileAlignment의 배수여야 한다. 파일 시작에서 SizeOfHeader 옵셋만큼 떨어진 위치에 첫 번째 섹션이 위치함.

- **7. Subsystem**

이 Subsystem 값을 보고 시스템 드라이버 파일(*.sys)인지, 일반 실행 파일(*.exe, *.dll)인지 구분할 수 있다. Subsystem 멤버는 아래 값을 가질 수 있다.

값 | 의미 | 비고
|:----|:----|:----|
1 | Driver file | 시스템 드라이버(예: ntfs.sys)
2 | GUI(Graphic User Interface) 파일 | 창 기반 애플리케이션(예: notepad.exe)
3 | CUI(Console User Interface) 파일 | 콘솔 기반 애플리케이션(예: cmd.exe)

- **8. NumberOfRvaAndSizes**

NumberOfRvaAndSizes는 IMAGE_OPTIONAL_HEADER32 구조체의 마지막 멤버인 DataDirectory 배열의 개수를 나타낸다. 구조체 정의에 분명히 배열 개수가 IMAGE_NUMBEROF_DIRECTORY_ENTRIES (16)이라고 명시되어 있지만, PE 로더는 NumberOfRvaAndSizes의 값을 보고 배열의 크기를 인식한다. 즉 16이 아닐 수도 있다는 뜻이다.

- **9. DataDirectory**

DataDirectory는 IMAGE_DATA_DIRECTORY 구조체의 배열로, 배열의 각 항목마다 정의된 값을 가진다(정의에는 16개로 정의됨). 

~~~
DataDirectory[0] = EXPORT Directory
DataDirectory[1] = IMPORT Directory
DataDirectory[2] = RESOURCE Directory
DataDirectory[3] = EXCEPTION Directory
DataDirectory[4] = SECURITY Directory
DataDirectory[5] = BASERELOC Directory
DataDirectory[6] = DEBUG Directory
DataDirectory[7] = COPYRIGHT Directory
DataDirectory[8] = GLOBALPTR Directory
DataDirectory[9] = TLS Directory
... 등등
~~~

여기서 말하는 Directory란 그냥 어떤 구조체의 배열이라고 생각하면 된다. 여기서 EXPORT, IMPORT, RESOURCE, TLS Directory는 중요하다. 나중에 다시 다룸.

- **IMAGE_OPTIONAL_HEADER 구조체 확인하기!**

![13-6](https://user-images.githubusercontent.com/26838115/45003738-700ad400-b020-11e8-95d8-4b76ba21d868.png)

00000100 주소에서 magic 값이 020B의 값을 가지는 것을 확인해 볼 수 있다!


---


- **섹션 헤더**

각 섹션의 속성(property)를 정의한 것이 섹션 헤더이다. PE 파일을 여러 개의 섹션 구조로 만들었을 때의 장점은 바로 프로그램의 안정성이다. 이를 위해서는 code/data/resouce마다 각각의 특성, 접근 권한 등을 다르게 설정할 필요가 있다.

종류 | 엑세스 권한
|:-----|:-----|
code | 실행, 읽기 권한
data | 비실행, 읽기, 쓰기 권한
resource | 비실행, 읽기 권한

- **IMAGE_SECTION_HEADER**

섹션 헤더는 각 섹션별 IMAGE_SECTION_HEADER 구조체의 배열로 되어 있다.

~~~
#define IMAGE_SIZEOF_SHORT_NAME              8

typedef struct _IMAGE_SECTION_HEADER {
   BYTE    Name[IMAGE_SIZEOF_SHORT_NAME];
    union {
            DWORD   PhysicalAddress;
            DWORD   VirtualSize;
    } Misc;
    DWORD   VirtualAddress;
    DWORD   SizeOfRawData;
    DWORD   PointerToRawData;
    DWORD   PointerToRelocations;
    DWORD   PointerToLinenumbers;
    WORD    NumberOfRelocations;
    WORD    NumberOfLinenumbers;
    DWORD   Characteristics;
} IMAGE_SECTION_HEADER, *PIMAGE_SECTION_HEADER;
~~~

IMAGE_SECTION_HEADER 구조체에서 알아야 할 중요 멤버는 다음과 같다(나머지는 사용되지 않음).

항목 | 의미
VirtualSize | 메모리에서 섹션이 차지하는 크기
VirtualAddress | 메모리에서 섹션의 시작 주소(RVA)
SizeOfRawData | 파일에서 섹션이 차지하는 크기
PointerToRawData | 파일에서 섹션의 시작 위치
Characteristic | 섹션의 속성(bit OR)

VirtualAddress와 PointerToRawData는 아무 값이나 가질 수 없고, 각각 (IMAGE_OPTIOANL_HEADER32에 정의된) SectionAlignment와 FileAlignment에 맞게 결정된다.

<br/>

VirtualSize와 SizeOfRawData는 일반적으로 서로 다른 값을 가진다. 이는 파일에서의 섹션 크기와 메모리에 로딩된 섹션의 크기는 다르다는 것을 의미한다!

<br/>

Characteristics는 다음과 같은 코드에 표시된 값들의 조합(bit OR)으로 이루어진다.

~~~
#define IMAGE_SCN_CNT_CODE                   0x00000020  // Section contains code.
	
#define IMAGE_SCN_CNT_INITIALIZED_DATA       0x00000040  // Section contains initialized data.
		
#define IMAGE_SCN_CNT_UNINITIALIZED_DATA     0x00000080  // Section contains uninitialized data.

#define IMAGE_SCN_MEM_EXECUTE                0x20000000  // Section is executable.

#define IMAGE_SCN_MEM_READ                   0x40000000  // Section is readable.

#define IMAGE_SCN_MEM_WRITE                  0x80000000  // Section is writeable.
~~~

마지막으로 Name 항목에 대해 얘기해 보자. Name 멤버는 C 언어의 문자열처럼 NULL로 끝나지 않는다. 또한 ASCII 값만 와야한다는 제한도 없다. PE 스펙에는 섹션 Name에 대한 어떠한 명시적인 규칙이 없기 때문에 어떠한 값을 넣어도 되고 NULL로 채워도 된다. 따라서 섹션의 Name은 그냥 참고용일 뿐 어떤 정보로써 활용하기에는 100% 장담할 수 없다. (데이터 섹션 이름을 '.code'로 해도 됨!) <br/>

notepad.exe의 IMAGE_DEXTION_HEADER 구조체 배열을 직접 보자.

![13-7](https://user-images.githubusercontent.com/26838115/45024530-32379b00-b074-11e8-9cb9-a4bdb937425c.png)


> PE 파일 설명에서 자주 등장하는 이미지(Image)라는 용어를 잘 알아두자. PE 파일이 메모리에 로딩될 때 파일이 그대로 올라가는 것이 아니라, 섹션 헤더에 정의된 대로 섹션 시작 주소, 섹션 크기 등에 맞춰서 올라간다. 따라서 파일에서의 PE와 메모리에서의 PE는 서로 다른 모양을 가진다. 이를 구별하기 위해서 메모리에 로딩된 상태를 이미지라는 용어를 사용해서 구별하는 것이다!

---

## RVA to RAW

섹션 헤더를 잘 이해했다면, PE 파일이 메모리에 로딩되었을 때 각 섹션에서 메모리의 주소(RVA)와 파일 옵셋을 잘 매핑할 수 있어야 한다. 이러한 매핑을 일반적으로 'RVA to RAW'라고 부른다. 방법은 아래와 같다.

1. RVA가 속해 있는 섹션을 찾는다.
2. 간단한 비례식을 사용해서 파일 옵셋(RAW)을 계산한다.

IMAGE_SECTION_HEADER 구조체에 의하면 비례식은 아래와 같다.

~~~
RAW - PointerToRawData = RVA - VirtualAddress
				   RAW = RVA - VirtualAddress + PointerToRawData
~~~

그럼 이제 퀴즈를 풀어보자.

![13-1](https://user-images.githubusercontent.com/26838115/44969169-d7744580-af86-11e8-8702-d03ce2dc3dc6.png)

**Q1.** RVA = 5000일 때 File Offset = ? <br/><br/>

A1. <br/>
RVA 5000은 첫 번째 섹션(.text)에 속해 있음(ImageBase 01000000를 고려함). <br/>
RAW = 5000(RVA) - 1000(VirtualAddres) + 400(PointerToRawData) = 4400

**Q2.** RVA = ABA8일 때 File Offset = ? <br/><br/>

A2. <br/>

해당 RVA 값이 속해 있는 섹션은 두 번째 섹션(.data)이다. <br/>
RAW = ABA8(RVA) - 9000(VA) + 7C00(PointerToRawData) = 97A8 (X) <br/>
계산 결과로 나온 옵셋은 세 번째 섹션에 속해 있다. RVA는 두 번째 섹션이고, RAW는 세 번째 섹션이라면 말이 안 된다! 따라서 이 경우에는 "해당 RVA(ABA8)에 대한 RAW값은 정의할 수 없다"고 해야 한다. 이런 이상한 결과가 나온 이유는 위 경우에 두 번째 섹션의 VirtualSize 값이 SizeOfRawData 값보다 크기 때문이다.

---

## IAT (Important Address Table)

IAT란 쉽게 말해서 프로그램이 어떤 라이브러리에서 어떤 함수를 사용하고 있는지를 기술한 테이블이다.

- **DLL**

IAT를 알아보기에 앞서 Windows OS의 근간을 이루는 DLL(Dynamic Lined Library) 개념을 짚고 넘어가자. DLL은 우리말로 '동적 연결 라이브러리'라고 한다. <br/>

이는 16비트 DOS 시절, 예를 들어 C 언어에서 printf() 함수를 사용할 때 컴파일러는 C 라이브러리에서 해당 함수의 binary 코드를 그대로 가져와서 프로그램에 삽입시켰다. 즉 실행 파일에 printf() 함수의 바이너리 코드를 가지고 있는 것인데, 이것이 매우 비효율적이었으므로 DLL 개념이 탄생했다(이 시기에는 그냥 'Library'만 존재함 ~~암흑의 시기~~). <br/>

- 프로그램에 라이브러리를 포함시키지 말고 별도의 파일(DLL)로 구성하여 필요할 때 마다 불러 쓰자.

- 일단 한 번 로딩된 DLL의 코드, 리소스는 Memory Mapping 기술로 여러 Process에서 공유해 쓰자.

- 라이브러리가 업데이트되었을 때 해당 DLL 파일만 교체하면 되니 쉽고 편해서 좋다.

실제 DLL 로딩 방식은 2가지이다. 프로그램에서 사용되는 순간에 로딩하고 사용이 끝나면 메모리에서 해제되는 방법(Explicit Linking)과 프로그램 시작할 때 같이 로딩되어 프로그램 종료할 때 메모리에서 해제되는 방법(Implicit Linking)이 있다. IAT는 바로 Implicit Linking에 대한 메커니즘을 제공하는 역할을 한다. IAT의 확인을 위해 OllyDbg로 notepad.exe를 열어보자.

![13-8](https://user-images.githubusercontent.com/26838115/45028505-51d4c080-b080-11e8-9b69-9f1b471484d3.png)

위 그림은 knernel32.dll의 CreateFileW를 호출하는 코드가 나와 있다. CreateFileW를 호출할 때 직접 호출하지 않고 01001104 주소에 있는 값을 가져와서 호출한다(모든 API 호출은 이런 방식으로 되어 있다). <br/>

01001104 주소는 notepad.exe에서 '.text' 섹션의 메모리 영역이다(더 정확히는 IAT 메모리 영역). 01001104 주소의 값은 7C8107F0이며, 7C8107F0 주소가 바로 notepad.exe 프로세스 메모리에 로딩된 kernel32.dll의 CreateFileW 함수 주소입니다. 물론 이렇게 쓰면 되지 않냐는 의문이 생길 수도 있다.

> CALL 7C810F0

하지만 이건 앞에서 설명했던 DOS 시절의 방식이다! <br/>

notepad.exe 제작자가 프로그램을 컴파일(생성)하는 순간에는 이 notepad.exe 프로그램이 어떤 Windows(Xp, 7 등), 어떤 언어(KOR, ENG 등), 어떤 Service Pack에서 실행될지 알 수 없다. 위에서 열거한 모든 환경에서 kernel32.dll의 버전이 달라지고, CreateFileW 함수의 위치(주소)가 달라진다. CreateFileW 함수 호출을 보장하기 위해서 컴파일러는 CreateFileW의 실제 주소가 저장될 위치(01001104)를 준비하고 CALL DWORD PTR DS:[1001104] 형식의 명령어를 적어두기만 한다. 그러다 파일이 실행되는 순간 PE 로더가 01001104의 위치에 CreateFileW의 주소를 입력해 준다. <br/>

컴파일러가 CALL 7C8107F0라고 쓰지 못하는 또 다른 이유는 DLL Relocation 때문이다. 일반적인 DLL 파일의 ImageBase 값은 10000000이다. 예를 들어 어떤 프로그램이 a.dll과 b.dll을 사용한다고 했을 때, PE 로더는 먼저 a.dll을 ImageBase 값인 메모리 10000000에 잘 로딩한다. 그 다음 b.dll을 ImageBase 값인 메모리 1000000에 로딩하려고 보면, 이미 그 주소는 a.dll이 사용하고 있다! 그래서 PE 로더는 다른 비어있는 메모리 공간(ex:3E000000)을 찾아서 b.dll을 로딩시켜 준다. <br/>

이것이 **DLL Relocation**이며 실제 주소를 하드코딩할 수 없는 이유이다. 또한 PE 헤더에서 주소를 나타낼 때 VA를 쓰지 못하고 RVA를 써야 하는 이유이기도 하다.

> DLL은 PE 헤더에 명시된 ImageBase에 로딩된다고 보장할 수 없다. 반면 process 생성 주체가 되는 EXE 파일은 자신의 ImageBase에 정확히 로딩된다(자신만의 가상 메모리 공간을 가지기 때문).


- **IMAGE_IMPORT_DESCRIPTOR**

PE 파일은 자신이 어떤 라이브러리를 임포트(Import)하고 있는지 IMAGE_IMPORT_DESCRIPTOR 구조체에 명시하고 있다.

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

typedef struct _IMAGE_IMPORT_BY_NAME {
    WORD    Hint;
    CHAR   Name[1];
} IMAGE_IMPORT_BY_NAME, *PIMAGE_IMPORT_BY_NAME;
~~~

일반적인 프로그램에서는 보통 여러 개의 라이브러리를 임포트하기 때문에 라이브러리의 개수만큼 위 구조체의 배열 형식으로 존재하며, 구조체 배열의 마지막은 NULL 구조체로 끝나게 된다. IMAGE_IMPORT_DESCRIPTOR 구조체에서 중요한 멤버는 다음과 같다.

항목 | 의미
|:----|:----|
OriginalFirstThunk | INT(Import Name Table)의 주소(RVA)
Name | Library 이름 문자열의 주소(RVA)
FirstThunk | IAT(Import Address Table)의 주소(RVA)

> PE 헤더에서 'Table'이라고 하면 '배열'을 뜻한다.
> 
> INT와 IAT는 long type(4바이트 자료형) 배열이고 NULL로 끝난다(크기가 따로 명시되어 있지 않다).
>
> INT에서 각 원소의 값은 IMAGE_IMPORT_BY_NAME 구조체 포인터이다(IAT도 같은 값을 가지는 경우가 있음).
>
> INT와 IAT의 크기는 같아야 한다.
>

아래 그림은 notepad exe의 kernel32.dll에 대한 IMAGE_IMPORT_DESCRIPTOR 구조를 표시하고 있다.

![13-9](https://user-images.githubusercontent.com/26838115/45029665-64e98f80-b084-11e8-9480-91ad4e36c9b3.png)

그럼 이제 PE 로더가 임포트 함수 주소를 IAT에 입력하는 기본적인 순서를 알아보자.

~~~
IAT 입력 순서

1. IID(_IMAGE_IMPORT_DESCRIPTOR)의 Name 멤버를 읽어서 라이브러리의 이름 문자열("kernel32.dll")을 얻는다.
2. 해당 라이브러리를 로딩한다.
	-> LoadLibrary("kernel32.dll")
3. IDD의 OriginalFirstThunk멤버를 읽어서 INT 주소를 얻는다.
4. INT에서 배열의 값을 하나씩 읽어 해당 IMAGE_IMPORT_BY_NAME 주소(RVA)를 얻는다.
5. IMAGE_IMPORT_BY_NAME의 Hint(ordinal) 또는 Name 항목을 이용하여 해당 함수의 시작 주소를 얻는다.
	-> GetProcAddress("GetCurrentThreadId")
	(ordianl은 라이브러리에서 함수의 고유번호를 의미한다.)
6. IDD의 FirstThnk(IAT) 멤버를 읽어서 IAT 주소를 얻는다.
7. 해당 IAT 배열 값에 위에서 구한 함수 주소를 입력한다.
8. INT가 끝날 때까지 (NULL을 만날 때까지) 위 4 ~ 7 과정을 반복한다.

~~~

위 그림에서는 INT와 IAT의 각 원소가 동시에 같은 주소를 가리키고 있지만, 그렇지 않은 경우도 많다(변칙적인 PE 파일에 대해서는 하나씩 직접 해보자).

- notepad.exe를 이용한 실습

IMAGE_IMPORT_DESCRIPTOR 구조체 배열은 PE 헤더가 아닌 PE 바디에 위치한다. 그곳을 찾아가기 위한 정보는 역시 PE 헤더에 있는데, IMAGE_OPTIONAL_HEADER32.DataDirectory[1].VirtualAddress 값이 실제 IMAGE_IMPORT_DESCRIPTOR 구조체 배열의 시작 주소이다(RVA 값). IMAGE_IMPORT_DESCRIPTOR 구조체 배열을 다른 용어로는 IMPORT Directory Table이라고 한다. <br/>

실습의 자세한 내용은 http://reversecore.com/23 에서 확인해 보도록 하자. PE 헤더 파트에 대한 자세한 내용이 해당 블로그에 담겨있다! (파일이 달라서 Hex Editor로 요소들을 정확히 읽어내기가 힘듦..) <br/>






> 위에 올려놓은 구조체들의 코드는 [winnt.h 파일](https://www.codemachine.com/downloads/win80/winnt.h)에서 직접 찾아볼 수 있다!





<br/>

---



### Issue #1

- 해결해야 할 이슈 : 

IMAGE_IMPORT_DESCRIPTOR에서 나온 구조체 멤버 중, 라이브러리의 주소(RVA 형식)를 가리키는 구조체 배열 INT와 IAT의 차이는 무엇일까?

<br/><br/>

- 답 :

FirstThunk와 OriginalFirstThunk의 차이? IAT와 INT는 동일하게 IMAGE_THUNK_DATA32 구조체 배열로 구성되어 있다. 하지만 바인딩 과정이라는걸 거치면서 INT에 값이 들어가면서 IAT가 된다. 즉, OriginalFirstThunk == INT, FirstThunk == IAT <br/>

---

### Issue #2

- 해결해야 할 이슈 : 

IAT 입력 순서에서 ***5.***번 부분이 어떻게 이루어지는지 잘 모르겠음.

