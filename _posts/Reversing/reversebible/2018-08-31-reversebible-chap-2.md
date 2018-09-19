---

layout : single
title : "Reversebible #2 - C 문법과 디스어셈블링"
categories : [Reversing]
tags : [reversebible]
comment : true

---

### '리버스 엔지니어링 바이블'의 핵심 내용 및 이슈들과 해결책을 다룹니다.

---

<br/>

### Windows programming 맛보기

~~~
#include <windows.h>
#include <stdio.h>
#include <tchar.h>

void RunPorcess() {
    STARTUPINFO si;
    PROCESS_INFORMATION pi;

    ZeroMemory(&si, sizeof(si));

    si.cb = sizeof(si);
    ZeroMemory(&pi, sizeof(pi));

    //start the child process.
    if (!CreateProcess(NULL,
            "MyChildProcess",
            NULL,
            NULL,
            FALSE,
            0,
            NULL,
            NULL,
            &si,
            &pi)
            )

    {
        printf("CreateProcess failed.\n");
        return;
    }
    // Wait until child process exits.
    WaitForSingleObject(pi.hProcess, INFINITE);

    // Close process and thread handles.
    CloseHandle(pi.hProcess);
    CloseHandle(pi.hThread);
}


int _tmain(int argc, _TCHAR* argv[])
{
    STARTUPINFO structSInfo;
    structSInfo.cb = sizeof(STARTUPINFO);

    GetStartupInfo(&structSInfo);

    _tprintf(_T("Desktop: %s\n"), structSInfo.lpDesktop);
    _tprintf(_T("Console: %s\n"), structSInfo.lpTitle);

    /*
     * printf() is like standard output function in C language,
     * whereas _tprintf() is Microsoft implementation defined in win32 api
     * which is not used now because standard dont support it.
     */

    return 0;
}
~~~

- 실습 :

해당 프로그램을 .exe 파일로 만들어 OllydDbg로 연 다음, 어셈블리 코드를 직접 트레이싱해 보자!

참고로 RunProcess에 필요한 파라미터를 STARTUPINFO, PROCESS_INFORMATION 구조체를 통해 공급하고 있다.




<br/>

---



### Issue #1

해결해야 할 이슈 : 

