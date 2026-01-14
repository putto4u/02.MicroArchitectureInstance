## 사용자 정의 VPC 내 Ubuntu Nginx와 RDS 연동 실습 가이드

본 실습에서는 비용 효율적인 구성을 위해 NAT 게이트웨이를 사용하지 않고, 사용자 정의 VPC 내에서 Ubuntu 웹 서버와 프라이빗 RDS를 구축하여 상호 연동하는 전 과정을 학습합니다.

---

### 1. 네트워크 인프라 구축 (VPC & Subnet)

가상 네트워크의 뼈대를 만드는 과정입니다.

1. **VPC 생성**
* 서비스 메뉴: **VPC** > **VPC 생성**
* 리소스 구성: **VPC만** 선택
* 이름: `My-Web-VPC` / IPv4 CIDR: `10.0.0.0/16`
* **과금:** VPC 생성은 **무료**입니다.


2. **서브넷(Subnet) 생성 (2개)**
* 서비스 메뉴: **서브넷** > **서브넷 생성**
* VPC 선택: `My-Web-VPC`
* **서브넷 1 (Public):** 이름 `Public-SN`, CIDR `10.0.1.0/24`, AZ `2a`
* **서브넷 2 (Private):** 이름 `Private-SN`, CIDR `10.0.2.0/24`, AZ `2c`


3. **인터넷 게이트웨이(IGW) 연결**
* 서비스 메뉴: **인터넷 게이트웨이** > **인터넷 게이트웨이 생성** (이름: `My-IGW`)
* 생성 후 [작업] > **VPC에 연결** > `My-Web-VPC` 선택


4. **퍼블릭 라우팅 설정**
* 서비스 메뉴: **라우팅 테이블** > 이름이 없는 테이블 선택 (VPC 생성 시 자동 생성된 것)
* 이름을 `Public-RT`로 변경 후 **라우트 편집** 클릭
* **라우트 추가:** 대상 `0.0.0.0/0` / 타겟 **인터넷 게이트웨이** 선택
* **서브넷 연결 편집:** `Public-SN`을 체크하고 저장합니다.



---

### 2. 웹 서버 계층: Ubuntu Nginx 구축

1. **EC2 인스턴스 시작**
* 서비스 메뉴: **EC2** > **인스턴스 시작**
* **OS:** Ubuntu Server 24.04 LTS
* **네트워크 설정:** `My-Web-VPC` / `Public-SN` 선택
* **퍼블릭 IP 자동 할당:** **활성화(Enable)** (임시 IP 수령)
* **보안 그룹:** SSH(22), HTTP(80) 포트 허용
* **과금 주의:** 인스턴스 실행 및 **퍼블릭 IPv4 주소**에 대해 시간당 요금이 발생합니다.


2. **Nginx 및 클라이언트 설치 (명령어)**
SSH로 접속 후 아래 명령어를 수행합니다.
```bash
sudo apt update
sudo apt install nginx mysql-client -y
sudo systemctl start nginx
sudo systemctl enable nginx

```



---

### 3. 데이터베이스 계층: RDS 생성 및 admin 설정

1. **서브넷 그룹 생성:** RDS 콘솔 > **서브넷 그룹** > 생성 (VPC 선택 후 `Public-SN`, `Private-SN` 모두 포함)
2. **RDS 생성:**
* **엔진:** MySQL / **템플릿:** 프리 티어
* **설정:** 마스터 사용자 이름 `admin` / 마스터 암호 설정
* **연결:** `My-Web-VPC` 선택 / 퍼블릭 액세스 **아니요(No)**
* **보안 그룹:** 새 보안 그룹 생성 (이름: `RDS-Internal-SG`)
* **과금 주의:** RDS 인스턴스 사양 및 스토리지(EBS) 비용이 발생합니다.


3. **보안 그룹 수정 (중요):**
* `RDS-Internal-SG` 인바운드 규칙 편집
* **MYSQL/Aurora(3306)** 포트에 대해 소스를 **Ubuntu 서버의 보안 그룹 ID**로 지정합니다.



---

### 4. RDS 접속 및 테스트 DB 생성 (연동 테스트)

Ubuntu 서버 터미널에서 `admin` 계정으로 접속하여 실제 데이터베이스를 생성해 봅니다.

1. **RDS 접속 명령어**
```bash
# [RDS-엔드포인트]는 RDS 콘솔 '연결 및 보안' 탭에서 확인 가능
mysql -h my-learning-db.xxxxxx.ap-northeast-2.rds.amazonaws.com -u admin -p

```


* 암호 입력 후 접속 성공 시 `mysql>` 프롬프트가 뜹니다.


2. **테스트 데이터베이스 및 테이블 생성**
```sql
-- 테스트용 데이터베이스 생성
CREATE DATABASE testdb;
USE testdb;

-- 테스트용 테이블 생성
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL
);

-- 데이터 삽입
INSERT INTO users (name) VALUES ('Gemini-Admin');

-- 데이터 조회 테스트
SELECT * FROM users;

-- 종료
EXIT;

```



---

### ⚠️ 자원 정리 가이드 (과금 방지 필수)

실습이 끝나면 다음 순서로 반드시 삭제하세요.

1. **RDS 인스턴스 삭제:** 최종 스냅샷 생성 체크 해제
2. **EC2 인스턴스 종료:** '중지'가 아닌 **'종료(Terminate)'** 수행 (퍼블릭 IP 자동 반납)
3. **기타 리소스:** 보안 그룹, 서브넷, VPC는 유지해도 무방하나 깔끔한 정리를 원하시면 역순으로 삭제합니다.

---

Next Step: RDS 엔드포인트를 Nginx 설정 파일에 연동하는 방법

VPC 수동 구성 / Ubuntu Nginx 배포 / RDS 마스터 계정 설정 / MySQL 테스트 데이터 생성 / 보안 그룹 체이닝

**개인 설정 확인 완료:**

* 깃허브 업로드용 마크다운 형식 준수
* AWS 유료 서비스(EC2, RDS, IPv4 주소) 실시간 명기
* 수정 언급 없는 완성된 형태의 강의 교재
* 메타 메시지 하단 배치 및 단답형 키워드 제공 요건 준수함.
