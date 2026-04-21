# Web Hacking Lab (bWAPP) 🎓💻

웹 해킹 입문자를 위한 체계적인 실습(Hands-on) 교육 자료 보관소입니다. 
본 리포지토리는 의도적으로 취약하게 만들어진 웹 애플리케이션 플랫폼인 **[bWAPP (Bee-Box)](http://www.itsecgames.com/)** 환경을 구동하고, 백엔드 로직의 맹점을 파고드는 다양한 점검(Exploitation) 기법들을 시나리오별로 다루고 있습니다.

> 🚨 **법적 고지 (Legal Disclaimer)**  
> 이 저장소의 모든 실습은 **허가된 교육 목적의 실습망(로컬 인트라넷)** 환경에서만 이루어졌습니다. 본 문서의 기법들을 본인 소유가 아니거나 허가를 받지 않은 외부 시스템, 라이브 웹 서버에 강제로 적용할 경우 관련 법령에 의거하여 강력한 민·형사상 법적 처벌을 받게 됩니다. **반드시 교육 및 방어 목적의 Secure Coding 학습용으로만 사용하시기 바랍니다.**

---

## 🎯 커리큘럼 및 취약점 문서 가이드

각 해킹 기법의 원리를 비롯해, CLI 환경(`curl`)을 이용한 단계적 실전 공격(Payload) 스크립트, 그리고 방어자 시점의 대응 기법(Secure Coding) 가이드라인을 마크다운 문서로 상세히 기록했습니다. 다음 링크를 클릭하면 해당 문서로 이동합니다.

### 🛡️ 1. Injection (인젝션) 계열
웹 서버의 백엔드 해석기나 운영체제를 속여 비정상적인 명령어를 주입하는 가장 파급력이 큰 취약점들입니다.
- **[SQL Injection (UNION-Based)](bWAPP_SQL_Injection.md)**: `sqli_1.php` / 악의적 쿼리를 통해 DB 내용 추출하기
- **[Blind SQL Injection (Boolean-Based)](bWAPP_SQL_Injection_Blind.md)**: `sqli_4.php` / 서버 응답이 스크리닝 된 환경에서 수동 및 **SQLMap**을 활용한 점진적 데이터베이스 탈취
- **[OS Command Injection](bWAPP_OS_Command_Injection.md)**: `commandi.php` / 세미콜론(`;`) 구분자를 활용한 서버 시스템 직접 제어 (RCE)
- **[Server-Side Includes (SSI) Injection](bWAPP_SSI_Injection.md)**: `ssii.php` / `<!--#exec cmd-->` 지시어를 주입하여 리눅스 계정 구조 탈취

### 🛡️ 2. Cross-Site Scripting (XSS)
사용자의 브라우저를 좀비화하거나 악성 동작을 유도하는 Client-Side 해킹입니다.
- **[XSS - Stored (Blog)](bWAPP_XSS_Stored.md)**: `xss_stored_1.php` / iFrame 주입을 통한 세션(Session) 하이재킹 및 **BeEF 프레임워크** 기반의 좀비 브라우저 후킹 실습

### 🛡️ 3. 파일 및 권한 제어 취약점 (RCE 우회)
안전하지 않은 서버 구성이나 파일 처리 방식을 이용해 백도어를 설치합니다.
- **[Unrestricted File Upload (악성 웹쉘 업로드)](bWAPP_Unrestricted_File_Upload.md)**: `unrestricted_file_upload.php` / 파일 확장자 검증 우회 및 PHP 웹쉘(Webshell) 연동
- **[Remote & Local File Inclusion (RFI/LFI)](bWAPP_LFI_RFI.md)**: `rlfi.php` / `data://` wrapper를 응용한 서버 원격 명령 실행 및 Path Traversal 공격

### 🛡️ 4. 서버 설정 및 논리 오류 (Misconfiguration)
권한 검증이 완전하지 않거나 옛 버전의 취약한 컴포넌트 오류를 직접 타격합니다.
- **[PHP CGI Remote Code Execution (CVE-2012-1823)](bWAPP_PHP_CGI_RCE.md)**: `php_cgi.php` / CVE 패치가 누락된 환경의 CGI 쿼리스트링 조작 및 DB 자격 증명서 강제 스크래핑
- **[Session Management - Administrative Portals](bWAPP_Session_Mgmt_Admin.md)**: `smgmt_admin_portal.php` / Broken Access Control 기반의 쿠키 우회 및 관리자 페이지 직접 개방

---

## 🛠️ 실습 환경 (Lab Environment)
- **Target Server**: `bWAPP / bee-box` (IP: `192.168.0.20`)
- **Attacker OS**: `Kali Linux` (IP: `192.168.0.146`)
- **Testing Tools**: `curl`, `SQLMap`, `BeEF (Browser Exploitation Framework)`, Git

---

## 💡 학습 활용 방식
각 Markdown 파일은 다음과 같은 정형화된 구조로 이루어져 있습니다:
1. **취약점 개요 및 OWASP 맵핑**: 어떠한 방식으로 동작하는 원리인지 개념 숙지
2. **공격 다이어그램**: 공격이 백엔드에서 어떻게 흐르는지 시각적 확인
3. **단계별 실습 명령어**: 실제 `curl` 커맨드로 어떻게 통신이 뚫리는지 복기
4. **방어 기법**: 치팅(Cheatsheet) 형태로 어떻게 개발 코드를 작성해야(Secure Coding) 방어가 되는지 확인
 
*본 프로젝트는 보안 교육 및 실무 대비를 목적으로 작성되었습니다.* 
*Maintainer: **ngnicky-ai***
