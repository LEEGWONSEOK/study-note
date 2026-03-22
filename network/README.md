# 웹개발자를 위한 네트워크 완전 정복

**대상 독자**: 웹 개발자 (Frontend/Backend/DevOps)
**목표**: 웹개발부터 클라우드 배포까지 필요한 모든 네트워크 지식 마스터
**분량**: 책 한 권 분량

---

## 목차

### Part 1: 네트워크 기초
- [Chapter 1. 네트워크란 무엇인가](part01-기초/01-네트워크-소개.md)
- [Chapter 2. OSI 7계층과 TCP/IP](part01-기초/02-OSI와-TCP-IP.md)
- [Chapter 3. IP 주소와 서브넷](part01-기초/03-IP주소와-서브넷.md)
- [Chapter 4. TCP vs UDP](part01-기초/04-TCP-UDP.md)
- [Chapter 5. 포트와 소켓](part01-기초/05-포트와-소켓.md)

### Part 2: HTTP/HTTPS
- [Chapter 6. HTTP 기초](part02-HTTP-HTTPS/06-HTTP-기초.md)
- [Chapter 7. HTTP 메서드와 상태코드](part02-HTTP-HTTPS/07-메서드와-상태코드.md)
- [Chapter 8. HTTP 헤더 완전정복](part02-HTTP-HTTPS/08-HTTP-헤더.md)
- [Chapter 9. HTTPS와 SSL/TLS](part02-HTTP-HTTPS/09-HTTPS-SSL-TLS.md)
- [Chapter 10. HTTP/2와 HTTP/3](part02-HTTP-HTTPS/10-HTTP2-HTTP3.md)
- [Chapter 11. RESTful API 설계](part02-HTTP-HTTPS/11-RESTful-API.md)

### Part 3: DNS와 도메인
- [Chapter 12. DNS 동작 원리](part03-DNS-도메인/12-DNS-동작원리.md)
- [Chapter 13. DNS 레코드 타입](part03-DNS-도메인/13-DNS-레코드.md)
- [Chapter 14. 도메인 구매와 설정](part03-DNS-도메인/14-도메인-설정.md)
- [Chapter 15. CDN과 DNS](part03-DNS-도메인/15-CDN.md)

### Part 4: 클라우드 네트워킹 (중요!)
- [Chapter 16. VPC와 서브넷](part04-클라우드네트워킹/16-VPC와-서브넷.md)
- [Chapter 17. 보안 그룹과 ACL](part04-클라우드네트워킹/17-보안그룹-ACL.md)
- [Chapter 18. 라우팅 테이블](part04-클라우드네트워킹/18-라우팅테이블.md)
- [Chapter 19. NAT와 인터넷 게이트웨이](part04-클라우드네트워킹/19-NAT-IGW.md)
- [Chapter 20. VPN과 Private Link](part04-클라우드네트워킹/20-VPN-PrivateLink.md)

### Part 5: 보안
- [Chapter 21. 방화벽 기초](part05-보안/21-방화벽.md)
- [Chapter 22. CORS 완전정복](part05-보안/22-CORS.md)
- [Chapter 23. 인증과 인가](part05-보안/23-인증과-인가.md)
- [Chapter 24. 쿠키와 세션](part05-보안/24-쿠키와-세션.md)
- [Chapter 25. JWT와 OAuth](part05-보안/25-JWT-OAuth.md)
- [Chapter 26. 웹 보안 공격과 방어](part05-보안/26-웹보안.md)
- [Chapter 27. DDoS와 Rate Limiting](part05-보안/27-DDoS-RateLimiting.md)

### Part 6: 웹 성능 최적화
- [Chapter 28. 브라우저 렌더링 과정](part06-웹성능/28-브라우저-렌더링.md)
- [Chapter 29. 캐싱 전략](part06-웹성능/29-캐싱.md)
- [Chapter 30. 압축과 최적화](part06-웹성능/30-압축과-최적화.md)
- [Chapter 31. 로드밸런싱](part06-웹성능/31-로드밸런싱.md)
- [Chapter 32. WebSocket과 실시간 통신](part06-웹성능/32-WebSocket.md)

### Part 7: 실전 운영
- [Chapter 33. 웹앱 배포 아키텍처](part07-실전/33-배포-아키텍처.md)
- [Chapter 34. NGINX 설정](part07-실전/34-NGINX.md)
- [Chapter 35. 프록시와 리버스 프록시](part07-실전/35-프록시.md)
- [Chapter 36. API Gateway](part07-실전/36-API-Gateway.md)
- [Chapter 37. 네트워크 모니터링](part07-실전/37-모니터링.md)
- [Chapter 38. 네트워크 디버깅](part07-실전/38-디버깅.md)

### Part 8: 부록
- [부록 A. 네트워크 명령어 치트시트](part08-부록/부록A-명령어-치트시트.md)
- [부록 B. 트러블슈팅 가이드](part08-부록/부록B-트러블슈팅.md)
- [부록 C. 클라우드별 네트워크 비교](part08-부록/부록C-클라우드-비교.md)
- [부록 D. 유용한 도구와 리소스](part08-부록/부록D-도구와-리소스.md)

---

## 학습 로드맵

### 입문 (1주) - 네트워크 기본기
- Part 1: 네트워크 기초
- OSI 7계층, IP, TCP/UDP, 포트 개념 확립

### 초급 (2주) - 웹 프로토콜
- Part 2: HTTP/HTTPS
- HTTP 프로토콜, REST API, HTTPS 보안
- 웹개발 필수 지식

### 중급 (2주) - 인프라와 클라우드
- Part 3-4: DNS, VPC, 보안 그룹, ACL
- **가장 중요!** 실제 배포 시 반드시 필요한 지식
- AWS/GCP/Azure 공통 개념

### 고급 (2주) - 보안과 성능
- Part 5-6: 방화벽, CORS, 인증/인가, 캐싱
- 실무 보안 이슈 해결
- 성능 최적화 기법

### 실전 (1주) - 운영과 트러블슈팅
- Part 7: NGINX, API Gateway, 디버깅
- 실제 운영 환경 구축
- 장애 대응 능력

---

## 왜 이 내용들을 알아야 할까?

### 실무 시나리오로 보는 필요성

#### 시나리오 1: "서버 배포 후 API 호출이 안 돼요"
```
문제: 프론트엔드에서 백엔드 API 호출 실패
원인: 보안 그룹에서 80/443 포트 미개방
해결: Chapter 17. 보안 그룹과 ACL 참고
```

#### 시나리오 2: "DB에 외부에서 접근이 돼요. 보안 위험!"
```
문제: RDS가 public subnet에 배치됨
원인: VPC와 서브넷 개념 부족
해결: Chapter 16. VPC와 서브넷 참고
```

#### 시나리오 3: "프론트엔드 배포 후 CORS 에러"
```
문제: Access-Control-Allow-Origin 에러
원인: CORS 정책 미설정
해결: Chapter 22. CORS 완전정복 참고
```

#### 시나리오 4: "특정 IP만 관리자 페이지 접근 허용하려면?"
```
문제: IP 기반 접근 제어 필요
원인: ACL 설정 방법 모름
해결: Chapter 17. 보안 그룹과 ACL 참고
```

#### 시나리오 5: "Lambda에서 RDS 연결 안 됨"
```
문제: Lambda와 RDS가 다른 VPC/서브넷
원인: 네트워크 격리 이해 부족
해결: Chapter 16, 18 참고
```

#### 시나리오 6: "사내망에서만 API 접근 가능하게 하려면?"
```
문제: VPN이나 Private Link 필요
원인: 프라이빗 네트워킹 개념 부족
해결: Chapter 20. VPN과 Private Link 참고
```

---

## 웹개발자가 반드시 알아야 할 네트워크 개념

### 레벨 1: 필수 (모든 개발자)
- HTTP/HTTPS 프로토콜
- CORS
- DNS
- 쿠키/세션
- 기본 보안 (XSS, CSRF)

### 레벨 2: 실무 (배포 경험 개발자)
- **VPC와 서브넷** ⭐
- **보안 그룹과 ACL** ⭐
- NGINX 설정
- 로드밸런싱
- CDN

### 레벨 3: 고급 (시니어/DevOps)
- 라우팅 테이블
- NAT Gateway
- VPN/Private Link
- API Gateway
- 네트워크 모니터링

---

## 클라우드 환경별 적용

이 자료의 내용은 다음 클라우드 환경에 모두 적용됩니다:

| 개념 | AWS | GCP | Azure |
|------|-----|-----|-------|
| VPC | VPC | VPC | Virtual Network |
| 서브넷 | Subnet | Subnet | Subnet |
| 보안 그룹 | Security Group | Firewall Rules | Network Security Group |
| ACL | Network ACL | Firewall Rules | Network Security Group |
| 로드밸런서 | ELB/ALB | Cloud Load Balancing | Load Balancer |
| NAT | NAT Gateway | Cloud NAT | NAT Gateway |

---

## 학습 방법

### 1. 이론 + 실습 병행
```bash
# 예: DNS 학습 시
이론: DNS A 레코드란?
실습: 직접 도메인 구매해서 A 레코드 설정하기
```

### 2. 클라우드 프리티어 활용
- AWS Free Tier로 VPC 직접 구성
- GCP $300 크레딧으로 실습
- Azure 무료 계정 활용

### 3. 도구 활용
- **Chrome DevTools**: Network 탭 (HTTP 분석)
- **curl/wget**: API 테스트
- **Postman**: REST API 테스트
- **ping/traceroute**: 네트워크 진단
- **nslookup/dig**: DNS 조회
- **telnet/nc**: 포트 확인

### 4. 실전 프로젝트
```
프로젝트 예시: 간단한 웹앱 배포

1. VPC 생성 (Public/Private 서브넷)
2. 보안 그룹 설정 (웹서버, DB 분리)
3. EC2에 NGINX + 앱 배포
4. RDS (Private 서브넷)
5. 도메인 연결 (Route 53)
6. HTTPS 인증서 (Let's Encrypt)
7. CloudFront CDN 연결
```

---

## 이 자료의 특징

### 1. 웹개발자 관점
- 네트워크 엔지니어가 아닌 **개발자 시각**
- 실무에서 **실제로 마주치는 문제** 중심
- "왜 이게 필요한가?"에 대한 명확한 답

### 2. 클라우드 시대 반영
- 온프레미스가 아닌 **클라우드 환경** 기준
- AWS/GCP/Azure **공통 개념** 설명
- Docker, Kubernetes 등 **현대 인프라** 고려

### 3. 실습 중심
- 모든 챕터에 **실습 과제** 포함
- **명령어 예제**와 **설정 파일** 제공
- **트러블슈팅** 가이드

### 4. 시각적 설명
- 다이어그램으로 **개념 시각화**
- Before/After **비교 예시**
- **네트워크 플로우** 도식화

---

## 준비물

### 필수
- 웹 브라우저 (Chrome 권장)
- 터미널 접근 권한
- 텍스트 에디터 / IDE

### 권장
- **클라우드 계정** (AWS/GCP/Azure 중 하나) ⭐
- Docker 설치
- Postman 또는 Insomnia
- 본인 소유 도메인 (DNS 실습용)

### 비용
- 클라우드 프리티어 활용 시 **거의 무료**
- 도메인: 연간 $10-15 (선택)
- 대부분 실습은 **로컬**에서 가능

---

## 각 Part별 학습 목표

### Part 1: 네트워크 기초
"IP 주소가 뭐죠?", "TCP와 UDP 차이가 뭐죠?" 같은 질문에 자신있게 답하기

### Part 2: HTTP/HTTPS
REST API를 설계하고, HTTP 상태코드를 정확히 사용하며, HTTPS 인증서를 직접 설정할 수 있기

### Part 3: DNS와 도메인
도메인을 구매해서 서버와 연결하고, CDN을 직접 설정할 수 있기

### Part 4: 클라우드 네트워킹 ⭐ 가장 중요!
VPC를 설계하고, 서브넷을 나누고, 보안 그룹과 ACL로 접근 제어를 할 수 있기

### Part 5: 보안
CORS 에러를 해결하고, JWT를 구현하며, 기본적인 보안 공격을 방어할 수 있기

### Part 6: 웹 성능 최적화
캐싱으로 응답 속도를 10배 개선하고, 로드밸런서를 설정할 수 있기

### Part 7: 실전 운영
NGINX를 설정하고, 네트워크 장애를 진단하고 해결할 수 있기

---

## 학습 후 얻게 되는 것

### 기술적 역량
- 웹 애플리케이션 **전체 아키텍처** 이해
- 클라우드 환경에서 **안전하고 확장 가능한** 네트워크 설계
- 네트워크 관련 **장애 대응** 능력

### 실무 역량
- DevOps 팀과 **원활한 소통**
- 성능 문제 **근본 원인 파악**
- 보안 이슈 **선제적 대응**

### 커리어
- **백엔드 개발자** → 네트워크까지 이해하는 개발자
- **프론트엔드 개발자** → 배포와 인프라까지 다루는 개발자
- **풀스택 개발자** → DevOps 역량을 갖춘 개발자

---

## 자주 묻는 질문

**Q: 네트워크 엔지니어가 되려는 건 아닌데, 이 정도까지 알아야 하나요?**
A: 현대 웹개발은 클라우드 배포가 기본입니다. VPC, 보안 그룹 정도는 반드시 알아야 합니다.

**Q: 클라우드를 안 쓰는데도 필요한가요?**
A: HTTP, CORS, DNS, 보안 등은 어떤 환경에서든 필수입니다.

**Q: AWS만 다루나요?**
A: 개념은 AWS/GCP/Azure 공통입니다. 부록에서 클라우드별 차이를 설명합니다.

**Q: 프론트엔드 개발자도 봐야 하나요?**
A: 네! CORS, HTTP, 캐싱, CDN 등은 프론트엔드에 직접적으로 영향을 줍니다.

---

## 기여 및 피드백

이 자료는 계속 업데이트됩니다.
- 오타/오류 발견 시 Issue 등록
- 추가 주제 제안 환영
- 실무 경험 공유 환영

---

**작성일**: 2024년
**대상**: 웹개발자 (Frontend/Backend/DevOps)
**학습 기간**: 6-8주 권장
**난이도**: 입문~고급

---

## 시작하기

준비되셨나요? 기초부터 탄탄히 시작하세요!

👉 [Chapter 1. 네트워크란 무엇인가](part01-기초/01-네트워크-소개.md)

또는 급하다면 바로 실전으로:

🚀 [Chapter 16. VPC와 서브넷 (클라우드 필수)](part04-클라우드네트워킹/16-VPC와-서브넷.md)