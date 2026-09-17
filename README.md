# Allergy Out Backend

**나의 알레르기와 목표 칼로리를 고려한 레시피 탐색 서비스**

## 프로젝트 소개

Allergy Out은 사용자가 피해야 할 재료를 제외하고, 자신의 식사 목표에 맞는 레시피를 찾을 수 있도록 돕는 서비스입니다.

이 저장소는 Allergy Out의 백엔드 API를 제공합니다. 개인별 알레르기 정보에 따른 레시피 필터링과 목표 칼로리 기반 추천을 중심으로, 레시피 공유·즐겨찾기·회원 관리 기능을 구현했습니다. 라즈베리파이에서 수집한 걸음 수를 저장하고 조회하는 활동 기록 기능도 지원합니다.

## 핵심 기능

### 알레르기 재료를 제외한 레시피 탐색

회원이 등록한 알레르기 재료를 레시피 목록과 오늘의 추천에 기본 반영합니다. 예를 들어 알레르기 재료로 `땅콩`을 등록하면, 재료명에 `땅콩`이 포함된 `땅콩버터` 등을 사용하는 레시피도 제외합니다.

- 개인별 알레르기 재료 등록 및 수정
- 내 알레르기 필터 적용 여부 선택 및 추가 제외 재료 지정
- 제목 검색, 음식 종류·조리 방법 필터, 최신순·조회수순 정렬

비로그인 상태에서도 레시피를 탐색할 수 있습니다. 로그인하고 인증 토큰을 보내면 내 알레르기 필터와 즐겨찾기 여부가 반영됩니다.

### 목표 칼로리에 맞춘 레시피 추천

프런트엔드에서 전달한 하루 목표 칼로리를 기준으로 한 끼 목표를 계산하고, 칼로리가 가까운 레시피를 최대 3개 추천합니다. 회원의 알레르기 재료가 포함된 레시피는 추천 대상에서 제외합니다.

```text
하루 목표 2,100 kcal
  -> 한 끼 목표 700 kcal
  -> 알레르기 재료 제외
  -> 700 kcal에 가까운 레시피 최대 3개 추천
```

칼로리 정보가 있는 레시피를 대상으로 하며, 조건에 맞는 후보가 없으면 빈 목록을 반환합니다. 별도로 날짜를 기준으로 레시피를 선정하는 오늘의 추천도 제공합니다.

### 레시피 공유와 활동 기록

| 기능 | 내용 |
| --- | --- |
| 레시피 공유 | 대표 이미지, 재료, 조리 단계, 영양 정보를 포함한 레시피 등록·수정·삭제 |
| 즐겨찾기 | 관심 있는 레시피 저장 및 내 즐겨찾기 목록 조회 |
| 회원 관리 | 회원가입·로그인, 회원 정보 수정, 프로필 이미지 관리, 회원 탈퇴 |
| 활동 기록 | 라즈베리파이 등록, 누적 걸음 수 저장, 오늘 및 최근 7일 기록 조회 |

## 기술 스택

| 구분 | 기술 |
| --- | --- |
| 언어·프레임워크 | Java 21, Spring Boot 4.0.8, Spring Web MVC |
| 인증·검증 | Spring Security, JWT (JJWT), Jakarta Validation |
| 데이터베이스·접근 | Oracle Database, Spring JDBC, MyBatis Spring Boot Starter 4.0.1 |
| 이미지 저장 | AWS S3, AWS SDK for Java v2 |
| API 문서 | springdoc-openapi, Swagger UI |
| 모니터링 | Spring Boot Actuator, Micrometer Prometheus |
| 테스트 | JUnit Jupiter, Mockito, Spring MVC 테스트 |
| 빌드·배포 | Gradle Wrapper, GitHub Actions, AWS EC2, Docker Compose |

## 프로젝트 구조

도메인별로 코드를 구성하고, 요청 처리·비즈니스 로직·데이터 접근을 Controller, Service, Mapper로 구분합니다. SQL은 MyBatis XML 매퍼에서 관리합니다.

```text
src/
  main/
    java/com/allergyout/
      auth/        # 회원가입, 로그인, 토큰 관리
      member/      # 회원 정보와 프로필
      allergy/     # 개인별 알레르기 정보
      recipe/      # 레시피 작성, 검색, 필터링, 추천
      bookmark/    # 즐겨찾기
      rasp/        # 라즈베리파이 및 걸음 수 기록
      s3/          # 이미지 업로드와 삭제
      admin/       # 관리자 기능 기반 코드 (API 미구현)
      global/      # 공통 응답, 예외 처리, 보안, 암호화, 설정
      AllergyOutApplication.java
    resources/
      mapper/      # 도메인별 MyBatis XML
  test/
    java/com/allergyout/
.github/
  workflows/
    api-workflow.yml
db/
  schema.sql       # 신규 Oracle DB용 테이블 및 제약조건
  seed.sql         # 개발용 예시 데이터
  verify.sql       # 데이터 확인용 조회
  README.md        # DB 구성 및 예시 데이터 적용 안내
```

### 주요 구현 방식

- **인증:** Access Token은 Bearer 헤더로, Refresh Token은 HttpOnly 쿠키로 처리합니다.
- **개인정보 보호:** 비밀번호는 BCrypt로 해시 처리합니다. 이메일·연락처는 AES-256-GCM으로 암호화하고, 중복 확인에는 HMAC-SHA256을 사용합니다.
- **레시피 관리:** 작성자만 수정·삭제할 수 있으며, 삭제는 데이터의 삭제 여부를 변경하는 소프트 삭제 방식입니다.
- **이미지 관리:** 프로필·레시피 이미지는 S3에 저장합니다. JPG·JPEG·PNG를 지원하며 파일당 최대 크기는 5 MiB입니다.
- **배포:** `main` 대상 PR에서 빌드·테스트를 수행하고, `main` push 시 EC2로 JAR를 전송한 뒤 Docker Compose의 `backend` 서비스를 재시작합니다.

## 실행 방법

### 1. 실행 환경 준비

JDK 21, Oracle Database의 프로젝트 스키마와 접속 정보, AWS S3 버킷 및 접근 자격 증명이 필요합니다. Gradle은 저장소에 포함된 Wrapper를 사용합니다.

신규 Oracle DB용 [스키마](db/schema.sql)와 [예시 데이터](db/seed.sql)를 함께 제공합니다. 기존 운영 DB를 변경하는 마이그레이션이 아닌, 현재 코드 기준의 개발용 구성입니다. 상세 적용 방법은 [DB 구성 안내](db/README.md)를 참고하세요.

> 애플리케이션 설정 파일, Oracle 접속 계정, S3 자격 증명 및 배포용 Docker Compose 환경은 별도로 준비해야 합니다.

```bash
git clone https://github.com/lno001/allergy-out-BE.git
cd allergy-out-BE
```

새 DB를 사용하는 경우 애플리케이션 계정으로 `db/schema.sql`을 먼저 실행합니다. SQL Developer에서는 파일을 열고 스크립트 실행(F5)을 사용합니다. 대상 테이블이 이미 있는 DB에는 적용하지 않습니다.

### 2. 애플리케이션 설정

`src/main/resources/application.yml`을 구성하고, 아래 예시의 `${...}`에 해당하는 환경변수를 실행 환경에 지정합니다. 이 경로의 설정 파일은 `.gitignore`에 등록되어 있습니다.

<details>
<summary>로컬 실행용 application.yml 예시</summary>

실제 운영 설정이 아닌 로컬 구성 예시입니다.

```yaml
server:
  port: 8080

spring:
  datasource:
    driver-class-name: oracle.jdbc.OracleDriver
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  servlet:
    multipart:
      max-file-size: 5MB
      max-request-size: 50MB

mybatis:
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.allergyout
  configuration:
    map-underscore-to-camel-case: true
    jdbc-type-for-null: 'NULL'

jwt:
  secret: ${JWT_SECRET}
  expiration: 1800000
  refresh-expiration: 604800000
  cookie:
    secure: false

app:
  crypto:
    aes-key: ${APP_CRYPTO_AES_KEY}
    hmac-key: ${APP_CRYPTO_HMAC_KEY}

cloud:
  aws:
    credentials:
      access-key: ${AWS_ACCESS_KEY_ID}
      secret-key: ${AWS_SECRET_ACCESS_KEY}
    region:
      static: ${AWS_REGION}
    s3:
      bucket: ${AWS_S3_BUCKET}

management:
  endpoints:
    web:
      exposure:
        include: health,prometheus
```

| 환경변수 | 설정 값 |
| --- | --- |
| `DB_URL` | Oracle JDBC URL. 예: `jdbc:oracle:thin:@//localhost:1521/XEPDB1` |
| `DB_USERNAME`, `DB_PASSWORD` | 프로젝트 스키마의 DB 계정 정보 |
| `JWT_SECRET` | 최소 32바이트의 충분히 무작위적인 서명 키 문자열 |
| `APP_CRYPTO_AES_KEY` | 32바이트 키를 Base64로 인코딩한 값 |
| `APP_CRYPTO_HMAC_KEY` | AES 키와 별개인 32바이트 키를 Base64로 인코딩한 값 |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | S3 객체 업로드·삭제 권한이 있는 자격 증명 |
| `AWS_REGION` | 버킷 리전. 예: `ap-northeast-2` |
| `AWS_S3_BUCKET` | 사용할 S3 버킷 이름 |

JWT 만료 시간은 밀리초 단위입니다. 예시에서는 Access Token 30분, Refresh Token 7일을 사용합니다. JWT 키는 문자열 바이트를 직접 사용하며, AES·HMAC 키는 Base64 디코딩 후 사용합니다. HTTPS 운영 환경에서는 `jwt.cookie.secure=true`로 설정합니다.

요청 전체 업로드 한도인 `max-request-size: 50MB`는 여러 이미지를 전송하기 위한 예시 값입니다.

</details>

로컬 프런트엔드의 CORS 허용 주소는 `http://localhost:5173`입니다. 다른 주소를 사용하는 경우 `SecurityConfig`의 허용 출처를 조정합니다.

### 3. 서버 실행

Windows PowerShell:

```powershell
.\gradlew.bat bootRun
```

macOS / Linux:

```bash
chmod +x gradlew
./gradlew bootRun
```

위 설정 기준 API 기본 주소는 `http://localhost:8080`입니다.

### 4. API 문서 확인

서버 실행 후 **[Swagger UI](http://localhost:8080/swagger-ui/index.html)**에서 엔드포인트와 요청·응답 스키마를 확인할 수 있습니다. OpenAPI 명세는 [API Docs](http://localhost:8080/v3/api-docs)에서 제공합니다.

인증이 필요한 요청에는 `Authorization: Bearer <accessToken>` 헤더를 사용합니다. 로그인·재발급 요청에서 Refresh Token 쿠키를 주고받으려면 클라이언트의 쿠키 전송 설정도 필요합니다.

### 5. 예시 데이터 적용

서버 실행 후 회원가입 API로 테스트 회원 `demouser`를 생성하고, 같은 DB에 `db/seed.sql`을 실행합니다. 회원가입 요청 예시는 [DB 구성 안내](db/README.md)에 있습니다. 회원을 API로 생성하므로 비밀번호와 개인정보에 실제 애플리케이션의 해시·암호화 처리가 적용됩니다.

예시 데이터에는 레시피 6개, 알레르기 재료 2개, 즐겨찾기 2개, 디바이스 1개와 최근 7일 걸음 수 기록이 포함됩니다. `db/verify.sql`로 데이터와 추천 결과를 확인할 수 있습니다. 이미 샘플 데이터가 있는 회원에게 다시 적용하면 중복 삽입 전에 중단합니다.

적용 순서: **스키마 생성 → 서버 설정·실행 → 테스트 회원 가입 → 예시 데이터 삽입 → 결과 확인**

### 6. 테스트 및 빌드

| 작업 | Windows PowerShell | macOS / Linux |
| --- | --- | --- |
| 테스트 | `.\gradlew.bat test` | `./gradlew test` |
| 테스트 포함 빌드 | `.\gradlew.bat clean build` | `./gradlew clean build` |

주요 도메인 서비스, S3 파일 검증, 개인정보 암호화, 요청 유효성 검증, 레시피 컨트롤러 바인딩 테스트를 포함합니다. 테스트 보고서는 `build/reports/tests/test/index.html`, 실행 JAR는 `build/libs/`에 생성됩니다.
