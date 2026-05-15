# 🔨 실시간 경매 플랫폼 MACTA
> SK쉴더스 루키즈 지능형 애플리케이션 개발 5기 Mini Project 3 - 4조
>
> 배포 주소: https://macta.store

**MACTA**: 사용자가 **상품을 등록**하고 **실시간 입찰**에 참여할 수 있는 **경매 플랫폼**으로, 경매 마감 직전에 **트래픽이 집중**되는 상황과 다양한 **보안 위협** 환경에서도 안정적으로 동작하도록 설계된 서비스
- **낙관적 락** 기반 **동시성 제어**를 통해 마감 직전 입찰 경쟁에서 발생할 수 있는 **Race Condition**, **중복 갱신**, **데이터 무결성** 문제를 방지
- **WAF**를 **ALB 앞단**에 구성하여 매크로성 반복 요청, **DDoS**, **SQL Injection**, **XSS**, **패킷 위변조** 등 **비정상 트래픽**을 사전에 차단하도록 구성
- **Kubernetes** 기반 배포 환경을 구축하여 **애플리케이션 컨테이너**를 안정적으로 운영하고, 트래픽 증가 상황에서도 **서비스 확장**과 **복구** 가능
- **Rolling Update** 전략을 적용하여 새로운 버전 배포 시에도 기존 **Pod**와 신규 **Pod**를 순차적으로 교체함으로써 **서비스 중단을 최소화**

&nbsp;
## 📌 Application
<img width="1100" alt="image" src="https://github.com/user-attachments/assets/91362b0e-22cf-441d-b4a9-e064fdd1a9d9" />

&nbsp;
### 경매 비즈니스 로직
<img width="1100" alt="image" src="https://github.com/user-attachments/assets/fbd1a5a3-c641-40d4-85b8-a71ed92f35c7" />
<img width="1100" alt="image" src="https://github.com/user-attachments/assets/b37780e3-c142-46f9-a3ff-7f849571b7d1" />

- 사용자가 **경매 상품**에 입찰하면 서버에서 **경매 진행 상태**, **종료 시간**, **현재 최고 입찰가**를 먼저 확인한 뒤 **입찰 가능 여부**를 판단
- **입찰 금액**은 현재 최고 입찰가보다 높은 경우에만 허용하여 **낮은 금액 입찰**이나 **동일 금액 중복 입찰**이 저장되지 않도록 검증
- **음수 금액**, **0원 입찰**, **필수 값 누락**, 종료된 경매에 대한 입찰 요청 등 **비정상 요청**을 사전에 차단하도록 **입력값 검증**을 수행
- 유효한 입찰이 등록되면 경매의 **현재 최고가**와 **최고 입찰자 정보**를 함께 갱신하여 이후 사용자에게 **최신 입찰 상태**가 반영되도록 처리
- **경매 종료 시점**에는 경매 상태를 **CLOSED**로 변경하고 마지막 최고 입찰자를 **최종 낙찰자**로 확정하여 **거래 단계**로 이어질 수 있도록 구성
- **댓글 및 답글 기능**을 제공하여 상품 상세 페이지에서 **구매 문의**, **판매자 답변**, 사용자 간 **질의응답**

&nbsp;
### 경매 종료 스케줄링
<img width="1100" alt="image" src="https://github.com/user-attachments/assets/f8cfa673-c3f3-4b15-8da9-b0b49e79e5f7" />

- **백엔드 스케줄러**가 일정 주기로 실행되며 진행 중인 경매 목록에서 **종료 시간이 지난 경매**를 조회
- **만료 대상 경매**를 선별할 때 이미 종료 처리된 경매는 제외하여 **중복 종료 처리**와 불필요한 **상태 변경**이 발생하지 않도록 구현
- 종료 시간이 지난 경매는 **추가 입찰**을 받을 수 없도록 상태를 변경하고, **현재 최고 입찰 내역**을 기준으로 **낙찰자**를 확정
- **입찰자가 존재하지 않는 경매**는 낙찰자 없이 종료 상태로 처리하여 **결제 및 배송 단계**가 생성되지 않도록 분기
- **스케줄러 기반 자동 처리**를 통해 관리자가 직접 경매를 마감하지 않아도 정해진 시간에 경매가 종료

&nbsp;
### 결제 및 배송
<img width="1100" alt="낙찰자 결제" src="https://github.com/user-attachments/assets/635e9a4e-024a-4df7-998e-c15ce54af2df" />

> **낙찰자 결제**

<img width="1100" alt="판매자 배송" src="https://github.com/user-attachments/assets/6d4436d7-0a5b-4fa9-a5a8-f02d1940cbf0" />

> **판매자 배송**

- 경매가 종료되고 **낙찰자**가 확정되면 해당 낙찰자를 기준으로 **거래 정보**가 생성되며 **결제 대기 상태**로 전환
- 낙찰자는 **마이페이지** 또는 **거래 화면**에서 낙찰 상품과 **최종 결제 금액**을 확인한 뒤 **결제 절차**를 진행 가능
- **결제 완료** 시 거래 상태를 결제 완료로 변경하고 판매자가 **배송 정보**를 입력하거나 배송을 시작할 수 있는 단계로 이어지도록 처리
- **판매자**는 결제 완료된 거래를 기준으로 배송을 진행하며, **배송 상태 변경 내역**이 구매자 화면에도 반영
- 거래 진행 단계에 따라 **결제 대기**, **결제 완료**, **배송 진행**, **거래 완료** 등의 상태를 관리하여 사용자별로 필요한 액션만 노출
- **낙찰자**와 **판매자**의 역할을 구분하여 결제는 낙찰자만, 배송 처리는 판매자만 수행할 수 있도록 **권한 흐름**을 분리

&nbsp;
### 동시성 제어
```java
@Entity
@Table(name = "auctions")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor
@Builder
public class Auction {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "current_price", nullable = false)
    private Long currentPrice;

    @Version
    @Column(nullable = false)
    private Long version;

    public void updateCurrentPrice(Long bidPrice) {
        this.currentPrice = bidPrice;
    }
}
```

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class BidService {

    private final AuctionRepository auctionRepository;

    @Transactional
    public void placeBid(Long auctionId, Long bidPrice) {

        Auction auction = auctionRepository.findById(auctionId)
                .orElseThrow();

        // 현재 최고가보다 높은 경우만 입찰 허용
        if (bidPrice <= auction.getCurrentPrice()) {
            throw new IllegalArgumentException("INVALID_BID_PRICE");
        }

        // Optimistic Lock 기반 최고가 갱신
        auction.updateCurrentPrice(bidPrice);
    }
}
```
- 경매 마감 직전에 여러 사용자의 **입찰 요청**이 동시에 들어오는 상황에서도 **데이터 무결성**을 보장하기 위해 **낙관적 락(Optimistic Lock)**을 적용
- **Auction 엔티티**에 `@Version` 필드를 두어 같은 경매 데이터를 동시에 수정하려는 요청이 발생하면 **버전 충돌**을 감지
- 입찰 처리 시 **현재 최고가 검증**과 **최고가 갱신**을 하나의 **트랜잭션** 안에서 수행하여 검증 시점과 저장 시점의 **데이터 불일치**를 줄임
- 동시에 들어온 입찰 요청 중 먼저 **커밋**된 요청만 경매 정보를 갱신하고, 이후 충돌이 발생한 요청은 **실패 처리**되어 잘못된 최고가 덮어쓰기를 방지
- 동일 경매에 대한 **중복 갱신**과 **Race Condition** 문제를 방지하여 **최종 입찰가**와 **낙찰자 정보**가 일관되게 저장

&nbsp;
### 실시간 알림

<img width="1100" alt="실시간 알림" src="https://github.com/user-attachments/assets/0a23279a-1130-4479-89cc-64f81801954d" />

- 사용자가 참여한 경매에서 **새로운 입찰**, **경매 종료**, **낙찰 결과**와 같은 이벤트가 발생하면 **알림 데이터**가 생성
- **입찰자**, **낙찰자**, **판매자**처럼 이벤트와 관련된 사용자에게 필요한 알림만 전달되도록 **수신 대상**을 구분
- 사용자가 서비스 이용 중 중요한 **경매 상태 변화**를 즉시 확인할 수 있도록 **실시간 알림** 형태로 이벤트를 전달
- **읽지 않은 알림 개수**를 사용자별로 관리하여 마이페이지 또는 알림 영역에서 확인하지 않은 알림 수를 표시할 수 있도록 구성
- 사용자가 알림을 확인하면 **읽음 상태**로 변경하여 이미 확인한 알림과 새로 도착한 알림을 구분할 수 있도록 구현

&nbsp;
### 인증 및 인가

<img width="1100" alt="JWT 인증 및 인가" src="https://github.com/user-attachments/assets/65bfbc22-096c-4c69-850f-9415a2ada8c1" />

- **JWT 기반 인증** 방식을 적용하여 로그인 성공 시 발급된 토큰으로 사용자의 요청을 식별 가능
- **보호된 API 요청**에서는 토큰의 **유효성**을 검증하고 인증된 사용자 정보가 필요한 **비즈니스 로직**에 전달되도록 구성
- 로그인한 사용자만 **입찰 등록**, **결제 진행**, **마이페이지 조회**, **관심 상품 관리**와 같은 주요 기능을 사용할 수 있도록 제한
- **상품 수정 및 삭제 요청**에서는 요청 사용자와 상품 등록자를 비교하여 **판매자 본인**만 상품 정보를 변경할 수 있도록 검증
- **거래 정보**, **입찰 내역**, **배송 처리 기능**에서는 낙찰자와 판매자의 역할을 구분하여 타인의 거래 정보를 임의로 수정할 수 없도록 제어
- **인증되지 않은 요청**이나 **권한이 없는 요청**은 비즈니스 로직 실행 전에 차단하여 **사용자 데이터**가 노출되거나 변경되지 않도록 처리

&nbsp;
### 마이페이지

<img width="1100" alt="image" src="https://github.com/user-attachments/assets/729dad27-80d7-4c74-bbcf-04c504a1f112" />

- 사용자가 등록한 **경매 상품 목록**을 제공하여 **판매 중인 상품**, **종료된 상품**, **낙찰 여부**를 한 화면에서 확인 가능
- 사용자가 참여한 **입찰 내역**을 제공하여 입찰한 상품, **입찰 금액**, **경매 진행 상태**, **낙찰 여부**를 추적
- **낙찰된 거래**와 **판매 중인 거래**의 진행 상태를 사용자 기준으로 분리하여 **결제 필요 여부**와 **배송 처리 필요 여부**를 확인할 수 있도록 구성

&nbsp;
## 🚀 Infrastructure & Deployment
<img width="1100" alt="스크린샷 2026-05-14 161959" src="https://github.com/user-attachments/assets/40a682ca-d015-4d7e-a90c-9b8faa4727ec" />

&nbsp;
### AWS 네트워크 분리
<img width="1100" alt="mermaid-diagram-guest-low" src="https://github.com/user-attachments/assets/d1d51fbe-c162-45d5-8463-c563578fe9d7" />

- **Public Subnet**에는 외부 요청을 수신하는 **ALB(Application Load Balancer)**와 Private Subnet의 아웃바운드 통신을 위한 **NAT Gateway**를 배치
- **Private Subnet**에는 실제 애플리케이션이 실행되는 **EKS**, 세션 및 실시간 처리에 활용되는 **Redis**, 영속 데이터를 저장하는 **RDS MariaDB**를 구성하여 내부망 기반으로 운영
- 외부 사용자는 **ALB까지만 접근** 가능하며, 애플리케이션 Pod와 데이터베이스는 **Private 네트워크 내부**에서만 통신하도록 분리
- 데이터베이스와 캐시 계층을 외부에 직접 노출하지 않아 **공격 표면을 최소화**하고, 장애 발생 시에도 네트워크 계층별로 문제 범위를 분리해 대응할 수 있도록 설계

&nbsp;
### GitOps 기반 CI/CD
<img width="1100" alt="image" src="https://github.com/user-attachments/assets/b7000f99-1e45-4f52-8cb4-f944c7e3a05d" />

- 개발자가 애플리케이션 코드를 변경하면 **GitHub Actions**가 자동으로 테스트 및 빌드 과정을 수행하고, **Docker Image**를 생성한 뒤 **Amazon ECR**에 Push
- 배포 대상 이미지 태그와 Kubernetes 설정은 **Infra Repository의 Manifest**로 관리하여, 애플리케이션 코드와 배포 상태를 명확히 분리
- **Argo CD**가 Infra Repository의 변경 사항을 감지하고, 선언된 Manifest와 실제 **EKS 클러스터 상태**를 비교하여 자동으로 동기화
- 배포 이력과 설정 변경이 모두 Git에 남기 때문에 **추적 가능성**, **롤백 용이성**, **운영 일관성**을 확보

&nbsp;
### 무중단 배포
<img width="1100" height="586" alt="스크린샷 2026-05-15 111754" src="https://github.com/user-attachments/assets/38baafa6-51db-4648-aa44-374708db4a80" />

- **Kubernetes Rolling Update** 전략을 적용하여 기존 Pod를 한 번에 종료하지 않고, 새로운 버전의 Pod를 순차적으로 생성한 뒤 트래픽을 전환
- 신규 Pod가 **Readiness Probe**를 통과한 경우에만 서비스 트래픽을 받을 수 있도록 구성하여, 준비되지 않은 애플리케이션으로 요청이 전달되는 상황을 방지
- 배포 중 문제가 발생하면 이전 ReplicaSet으로 되돌릴 수 있어 **서비스 중단 시간 최소화**와 **빠른 장애 복구**가 가능
- 실시간 입찰 서비스 특성상 배포 중에도 사용자의 입찰 요청과 WebSocket 연결 흐름이 최대한 유지되도록 **가용성 중심의 배포 방식**을 적용

&nbsp;
## 🔐 Security & Network

### AWS WAF 적용

- **AWS WAF**를 **ALB 앞단**에 배치하여 애플리케이션 서버로 요청이 전달되기 전에 1차 보안 필터링을 수행
- **SQL Injection**, **XSS**, 비정상 User-Agent, 과도한 반복 요청 등 웹 취약점을 노리는 트래픽을 사전에 차단
- **Rate Limit Rule**을 적용하여 특정 IP에서 짧은 시간 동안 과도하게 요청하는 패턴을 제한하고, 입찰 API나 로그인 API에 대한 악성 반복 호출을 완화
- 보안 규칙을 인프라 계층에서 적용함으로써 애플리케이션 코드 변경 없이도 **공통 보안 정책**을 일관되게 유지

&nbsp;
### HTTPS / WSS 암호화 통신

- **ACM(AWS Certificate Manager)** 인증서를 활용하여 사용자와 서비스 간 통신에 **HTTPS**를 적용
- 실시간 알림과 입찰 상태 전달에 사용되는 WebSocket 역시 **WSS(WebSocket Secure)** 기반으로 구성하여 양방향 통신 구간을 암호화
- 전송 구간에서 발생할 수 있는 **패킷 위변조**, **스니핑**, **중간자 공격(MITM)** 위험을 줄이고 사용자 인증 정보와 거래 데이터를 보호
- 브라우저 보안 정책에 맞는 안전한 연결을 제공하여 로그인, 결제, 실시간 알림 같은 주요 기능이 신뢰된 채널에서 동작하도록 구성

&nbsp;
### Secret 관리

- **DB 비밀번호**, **JWT Secret**, **Redis 접속 정보**, 외부 API Key와 같은 민감 정보는 Git Repository와 Kubernetes Manifest에 직접 저장하지 않도록 분리
- 민감 값은 **AWS SSM Parameter Store**에 저장하고, 클러스터 내부에서는 **External Secrets Operator**를 통해 **Kubernetes Secret**으로 동기화하여 사용
- Secret 변경 시 애플리케이션 설정 파일이나 Manifest를 직접 수정하지 않아도 되어 **비밀 값 노출 위험**과 **운영 실수**를 줄임
- 코드와 설정 저장소에는 Secret의 실제 값이 아닌 참조 구조만 남기므로, 협업 과정에서도 **민감 정보 유출 가능성**을 최소화

&nbsp;
### IRSA 기반 AWS 권한 관리

- Pod 내부에 장기 **Access Key**를 직접 저장하지 않고, **IRSA(IAM Roles for Service Accounts)**를 적용하여 AWS 권한을 부여
- **Kubernetes ServiceAccount**와 **IAM Role**을 연결해 특정 Pod가 필요한 AWS 리소스에만 접근할 수 있도록 **최소 권한 원칙**을 적용
- 예를 들어 S3 업로드가 필요한 Pod에는 S3 관련 권한만 부여하고, 다른 Pod에는 해당 권한이 전달되지 않도록 역할을 분리
- Access Key 유출 위험을 제거하고, 워크로드 단위로 권한을 추적할 수 있어 **보안성**과 **감사 가능성**을 높임

&nbsp;
### Private S3 통신

- **S3 Gateway VPC Endpoint**를 적용하여 Private Subnet의 애플리케이션이 인터넷을 거치지 않고 **AWS 내부 네트워크**를 통해 S3에 접근하도록 구성
- 이미지 업로드 및 조회 과정에서 S3 트래픽이 외부 인터넷 경로로 나가지 않도록 하여 **네트워크 보안성**과 **전송 안정성**을 강화
- **S3 Bucket Public Access**를 비활성화하여 외부 공개 접근을 차단하고, 필요한 요청만 애플리케이션과 IAM 정책을 통해 제어
- VPC Endpoint와 Bucket 정책을 함께 사용해 허용된 네트워크와 권한 주체만 S3를 사용할 수 있도록 **접근 제어 범위**를 명확히 제한

&nbsp;
## 🔧 Tech Stack

### Backend

#### Language & Framework
![Java](https://img.shields.io/badge/Java_17-007396?style=flat-square&logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring_Boot_3.5.14-6DB33F?style=flat-square&logo=springboot&logoColor=white)
![Spring Security](https://img.shields.io/badge/Spring_Security-6DB33F?style=flat-square&logo=springsecurity&logoColor=white)
![Spring Data JPA](https://img.shields.io/badge/Spring_Data_JPA-6DB33F?style=flat-square&logo=spring&logoColor=white)
![Spring Validation](https://img.shields.io/badge/Spring_Validation-6DB33F?style=flat-square)

#### Authentication & Realtime
![JWT](https://img.shields.io/badge/JWT-000000?style=flat-square&logo=jsonwebtokens&logoColor=white)
![WebSocket](https://img.shields.io/badge/WebSocket-010101?style=flat-square)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=flat-square&logo=redis&logoColor=white)

#### Database & Storage
![MariaDB](https://img.shields.io/badge/MariaDB-003545?style=flat-square&logo=mariadb&logoColor=white)
![H2](https://img.shields.io/badge/H2-1F6FEB?style=flat-square)

#### AWS & Build
![AWS SDK](https://img.shields.io/badge/AWS_SDK_S3_/_STS-FF9900?style=flat-square&logo=amazonaws&logoColor=white)
![Maven](https://img.shields.io/badge/Maven-C71A36?style=flat-square&logo=apachemaven&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)

&nbsp;
### Frontend

#### Core
![React](https://img.shields.io/badge/React_18-61DAFB?style=flat-square&logo=react&logoColor=black)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white)
![Vite](https://img.shields.io/badge/Vite-646CFF?style=flat-square&logo=vite&logoColor=white)
![React Router](https://img.shields.io/badge/React_Router-CA4245?style=flat-square&logo=reactrouter&logoColor=white)

#### UI & Styling
![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-06B6D4?style=flat-square&logo=tailwindcss&logoColor=white)
![Shadcn UI](https://img.shields.io/badge/Shadcn_UI-000000?style=flat-square)
![Lucide React](https://img.shields.io/badge/Lucide_React-F56565?style=flat-square)

#### State & Data
![Axios](https://img.shields.io/badge/Axios-5A29E4?style=flat-square&logo=axios&logoColor=white)
![TanStack Query](https://img.shields.io/badge/TanStack_Query-FF4154?style=flat-square&logo=reactquery&logoColor=white)
![Zustand](https://img.shields.io/badge/Zustand-4B5563?style=flat-square)

#### Form & Validation
![React Hook Form](https://img.shields.io/badge/React_Hook_Form-EC5990?style=flat-square&logo=reacthookform&logoColor=white)
![Zod](https://img.shields.io/badge/Zod-3068B7?style=flat-square)
![date-fns](https://img.shields.io/badge/date--fns-770C56?style=flat-square)

#### Build & Container
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)


&nbsp;
### Infrastructure

#### Infrastructure as Code
![Terraform](https://img.shields.io/badge/Terraform-844FBA?style=flat-square&logo=terraform&logoColor=white)

#### Network
![VPC](https://img.shields.io/badge/VPC-FF9900?style=flat-square&logo=amazonaws&logoColor=white)
![Public Subnet](https://img.shields.io/badge/Public_Subnet-FF9900?style=flat-square)
![Private Subnet](https://img.shields.io/badge/Private_Subnet-FF9900?style=flat-square)
![Internet Gateway](https://img.shields.io/badge/IGW-FF9900?style=flat-square)
![NAT Gateway](https://img.shields.io/badge/NAT_Gateway-FF9900?style=flat-square)

#### Kubernetes & Compute
![EKS](https://img.shields.io/badge/EKS-326CE5?style=flat-square&logo=amazoneks&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)
![AWS Load Balancer Controller](https://img.shields.io/badge/AWS_Load_Balancer_Controller-326CE5?style=flat-square)

#### Database & Storage
![RDS MariaDB](https://img.shields.io/badge/RDS_MariaDB-527FFF?style=flat-square&logo=amazonrds&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=flat-square&logo=redis&logoColor=white)
![S3](https://img.shields.io/badge/S3-569A31?style=flat-square&logo=amazons3&logoColor=white)
![ECR](https://img.shields.io/badge/ECR-FF9900?style=flat-square&logo=amazonecr&logoColor=white)

#### Security
![WAFv2](https://img.shields.io/badge/WAFv2-DD344C?style=flat-square)
![ACM](https://img.shields.io/badge/ACM-FF9900?style=flat-square)
![IRSA](https://img.shields.io/badge/IRSA-DD344C?style=flat-square)
![SSM Parameter Store](https://img.shields.io/badge/SSM_Parameter_Store-FF9900?style=flat-square)
![External Secrets Operator](https://img.shields.io/badge/External_Secrets_Operator-326CE5?style=flat-square)

#### CI/CD & Monitoring
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=flat-square&logo=githubactions&logoColor=white)
![Argo CD](https://img.shields.io/badge/Argo_CD-EF7B4D?style=flat-square&logo=argo&logoColor=white)
![CloudWatch](https://img.shields.io/badge/CloudWatch-FF4F8B?style=flat-square&logo=amazoncloudwatch&logoColor=white)


&nbsp;
## 🔗 Repository
- [**Backend Repository**](https://github.com/SK-Rookies5-Auction/backend)
- [**Frontend Repository**](https://github.com/SK-Rookies5-Auction/frontend)
- [**Infra Repository**](https://github.com/SK-Rookies5-Auction/infra)

&nbsp;
## 💻 Developers

| <a href="https://github.com/owhat02" target="_blank"><img width="120" height="120" src="https://github.com/owhat02.png" /></a> | <a href="https://github.com/Eojinn" target="_blank"><img width="120" height="120" src="https://github.com/Eojinn.png" /></a> | <a href="https://github.com/Hyeonseok93" target="_blank"><img width="120" height="120" src="https://github.com/Hyeonseok93.png" /></a> | <a href="https://github.com/mmije0ng" target="_blank"><img width="120" height="120" src="https://github.com/mmije0ng.png" /></a> | <a href="https://github.com/seoyeon020" target="_blank"><img width="120" height="120" src="https://github.com/seoyeon020.png" /></a> | <a href="https://github.com/JangSeonguk1011" target="_blank"><img width="120" height="120" src="https://github.com/JangSeonguk1011.png" /></a> |
|:-------------:|:------:|:------:|:------:|:------:|:------:|
| [이새연(팀장)](https://github.com/owhat02) | [김어진](https://github.com/Eojinn) | [김현석](https://github.com/Hyeonseok93) | [박미정](https://github.com/mmije0ng) | [임서연](https://github.com/seoyeon020) | [장성욱](https://github.com/JangSeonguk1011) |
