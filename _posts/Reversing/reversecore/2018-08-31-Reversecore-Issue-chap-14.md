---

layout : single
title : "Reversecore Issue chap 14 - 실행 압축"
categories : [Reversing]
tags : [reversecore]
comment : true

---

### '리버싱 핵심 원리'의 내용 및 이슈들과 해결책을 다룹니다.

---

<br/>

**실행 압축**이란 말 그대로 실행(PE: Portable Executable)파일을 대상으로 파일 내부에 압축해제 코드를 포함하고 있어서 실행되는 순간에 메모리에서 압축을 해제시킨 수 실행시키는 기술이다. 실행 압축된 파일 역시 PE 파일이며, 내부에 원본 PE 파일과 decoding 루틴이 존재한다. EP(Entry Point) 코드에 decoding 루틴이 실행되면서 메모리에서 압축을 해제시킨 후 실행된다.


---

- 일반적인 ZIP 압축과 실행 압축의 차이

항목 | 일반 압축 | 실행 압축
|:-----|:--------|:------|
대상 파일 | 모든 파일 | PE 파일(exe, dll, sys)
압축 결과물 | 압축(zip, rar) 파일 | PE 파일(exe, dll, sys)
압축해제 방식 | 전용 압축해제 프로그램 사용 | 내부의 decoding 루틴
파일 실행 여부 | 자체 실행 불가 | 자체 실행 가능
장점 | 모든 파일에 대해 높은 압축율 | 별도의 해제 프로그램 없이 바로 실행 가능
단점 | 전용 압축해제 프로그램이 없으면 해당 압축 파일을 사용할 수 없음 | 실행할 때마다 decoding 루틴이 호출되기 때문에 실행시간이 아주 미세하게 느려짐


일반 PE 파일을 실행 압축 파일로 만들어 주는 유틸리티를 패커, 좀 더 Anti-Reversing 기법에 특화된 패커를 프로텍터라고 한다.


### 패커 

PE 패커(Packer)란 실행 파일 압축기를 말한다. 정식 명칭은 Run-Time 패커로, PE 파일 전문 압축기이다. PE 패커는 PE 파일의 크기를 줄이거나, PE 파일의 내부 코드와 리소스를 감추기 위한 목적으로 쓰인다.


### 프로텍터

PE 프로텍더(Protector)란 PE 파일을 'Reverse Code Engineering'으로부터 보호하기 위한 유틸리티이다. 리버싱을 막기위해 다양한 기법이 추가되어 오히려 원본 PE 파일보다 커지는 경향이 있다. 목적은 크래킹(Cracking) 방지와 코드 및 리소스 보호 등이 있다.

---

- notepad.exe를 통한 실습

notepad.exe를 upx 패커를 이용해 압축해 보자.

![14-1](https://user-images.githubusercontent.com/26838115/45084127-77250580-b138-11e8-83ce-667b8c43a998.png)

notepad.exe와 notepad_upx.exe를 비교하면, ".text" 섹션의 RawDataSize가 0이 되고, EP가 두 번째 섹션으로 이동한 것을 관찰할 수 있다.

그런데 PEview를 통해 notepad_upx.exe를 보면(위의 비교 그림은 단순 참고용이므로, 세세한 값은 다르다),

![14-2](https://user-images.githubusercontent.com/26838115/45084239-d256f800-b138-11e8-9948-fff3a286bd52.png)

Size of Raw Data 값이 0인데도 불구하고, Virtual Size는 31000인 것을 확인할 수 있다. 즉, UPX로 실행 압축된 파일은 실행되는 순간에 파일에 있는 압축된 코드를 메모리에 있는 첫 번째 섹션에 풀어 버린다는 것을 알 수 있다. 압축해제 코드와 압축된 원본 코드는 두 번째 섹션에 존재하고, 파일이 실행되면 먼저 압축해제 코드가 실행되어 압축된 원본 코드를 첫 번째 섹션에 해제시키는 것이다. 압축해제 과정이 끝나면 원본 EP 코드를 실행하는 것이다!


<br/>

---



### Issue #1

- 해결해야 할 이슈 : 

