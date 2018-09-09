---

layout : single
title : "Reversecore Issue chap 17 - 실행 파일에서 .reloc 섹션 제거하기"
categories : [Reversing]
tags : [reversecore]
comment : true

---

### '리버싱 핵심 원리'의 내용 및 이슈들과 해결책을 다룹니다.

---

<br/>


PE 파일에서 .reloc 섹션을 수동으로 제거하는 실습을 해 보자.


---

### .reloc 섹션

EXE 형식의 PE 파일에서 Base Reloaction Table 항목은 실행에 큰 영향을 끼치지 않는다(DLL, SYS 형식의 파일은 거의 필수!).

VC++ 에서 생성된 PE 파일의 Relocation 섹션 이름은 '.reloc'이다. .reloc 섹션이 제거되면 PE 파일의 크기가 약간 줄어드는 효과가 있다. 이때, .reloc 섹션은 보통 마지막에 위치한다.

---

### reloc.exe


'.reloc' 섹션 제거는 아래의 4 단계의 작업을 통해 이루어진다.

1. .reloc 섹션 헤더 정리

2. .reloc 섹션 제거

3. IMAGE_FILE_HEADER 수정

4. IMAGE_OPTIONAL_HEADER 수정

---

- .reloc 섹션 헤더 정리

![17-1](https://user-images.githubusercontent.com/26838115/45260473-c6628300-b423-11e8-87b2-b21588a128f3.png)

PE View를 통해 .reloc 섹션 헤더는 파일 옵셋 270에서 시작된다는 것을 알 수 있다. 이제 Hex Editor을 이용해서 해당 섹션을 0으로 덮어 써 보자.

![17-2](https://user-images.githubusercontent.com/26838115/45260487-14778680-b424-11e8-8c96-0ca128eded03.png)


- .reloc 섹션 제거

앞의 PE View에서의 그림을 보면, pointer to Raw Data 값의 Data가 0000C000인 것을 일 수 있다. 즉, 파일에서 .reloc 섹션의 시작 옵셋이 C000이다(이곳부터 파일 끝까지 .reloc 섹션 영역임). 이제 Hex Editor로 C000 옵셋부터 파일 끝까지 삭제하면 된다.

하지만 다른 PE 헤더 정보들이 아직 수정되지 않아 파일이 정상적으로 실행되지 않는다. 따라서 관련 PE 헤더 정보를 수정해 정상 실행을 시켜보자!


- IMAE_FILE_HEADER 수정

섹션을 하나 제거했으니 IMAGE_FILE_HEADER - Number of Sections 항목을 수정해야 한다.

![3](https://user-images.githubusercontent.com/26838115/45260524-1726ab80-b425-11e8-8a11-afa9e01ee834.png)

위의 005 값을 004로 수정하자(Hex Editor 이용).

![4](https://user-images.githubusercontent.com/26838115/45260550-92885d00-b425-11e8-8bf7-0cc14e9d006b.png)


- IMAGE_OPTIONAL_HEADER 수정

.reloc 섹션이 제거되면서 (프로세스 가상 메모리에서) 섹션 크기만큼 전체 이미지 크기가 줄어들었다. 이미지 크기는 IMAGE_OPTIONAL_HEADER - Size of Image 값에 명시되어 있으므로 이를 수정해야 한다.

![5](https://user-images.githubusercontent.com/26838115/45260567-f3b03080-b425-11e8-9c4b-e334f3d2a200.png)

위의 그림을 보면 Size of Image가 11000인 것을 확인할 수 있다. .reloc 섹션의 VirtualSize 값은 E40이지만, 이를 Section Alignment에 맞게 확장하면 1000이 된다. 따라서 1000을 Size of Image에서 빼주면 된다.

수정하면 정상 실행됨을 확인할 수 있다!

![6](https://user-images.githubusercontent.com/26838115/45260588-6caf8800-b426-11e8-8742-33e8e6749a5b.png)




<br/>

---



### Issue #1

- 해결해야 할 이슈 : 

