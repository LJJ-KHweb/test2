# Allergy Out Backend

**개인의 알레르기 정보와 목표 칼로리를 반영하는 레시피 서비스, Allergy Out의 백엔드입니다.**

사용자가 등록한 알레르기 재료를 기준으로 레시피를 필터링하고, 날짜 및 목표 칼로리에 따른 레시피 추천을 제공합니다. 레시피 작성과 즐겨찾기, 회원 정보 관리, 라즈베리파이에서 수집한 걸음 수 기록을 REST API로 지원합니다.

## 주요 기능

| 영역 | 기능 |
| --- | --- |
| 인증 | 회원가입, 로그인, 로그아웃, JWT Access Token 재발급, 로그인 회원 조회 |
| 회원 | 내 정보 조회, 이름·이메일·연락처·비밀번호 변경, 프로필 이미지 관리, 회원 탈퇴 |
| 알레르기 | 개인별 알레르기 재료 조회 및 수정, 레시피 조회 시 자동 필터링 |
| 레시피 | 목록 및 상세 조회, 제목 검색, 재료 제외, 음식 종류·조리 방법 필터, 최신순·인기순 정렬 |
| 레시피 작성 | 대표 이미지·재료·조리 단계 등록 및 수정, 작성자 권한 확인, 소프트 삭제 |
| 추천 | 날짜 기반 추천, 하루 목표 칼로리를 반영한 추천 |
| 즐겨찾기 | 레시피 즐겨찾기 등록·해제, 내 즐겨찾기 목록 조회 |
| 활동 기록 | 회원별 라즈베리파이 등록, 누적 걸음 수 저장, 오늘 및 최근 7일 기록 조회 |
| 이미지 | AWS S3를 이용한 프로필·레시피 이미지 업로드 및 삭제 |

### 레시피 필터링과 추천

- **알레르기 필터:** 로그인한 회원의 알레르기 재료를 목록 조회와 날짜 기반 추천에 기본 적용합니다. `applyMyAllergy=false`로 해제할 수 있습니다.
- **제외 재료:** `excludeMaterials`로 지정한 재료가 포함된 레시피를 제외합니다. 재료명 부분 일치를 사용합니다.
- **오늘의 추천:** 요청한 날짜와 레시피 번호를 이용한 해시 정렬로 최대 3개를 반환합니다. 후보 데이터와 조회 조건이 같으면 같은 날짜에 같은 결과를 제공합니다.
- **칼로리 기반 추천:** 프런트엔드에서 전달한 하루 목표 칼로리를 3으로 나눈 뒤, 한 끼 목표에 가까운 레시피를 최대 3개 반환합니다. 회원의 알레르기 재료를 제외하며, 칼로리 정보가 있는 레시피만 대상으로 합니다.

## 기술 스택

저장소의 빌드 설정 기준입니다.

| 구분 | 기술 |
| --- | --- |
| 언어 | Java 21 |
| 프레임워크 | Spring Boot 4.0.8, Spring Web MVC |
| 인증·검증 | Spring Security, JJWT 0.12.6, Jakarta Validation |
| 데이터 접근 | Spring JDBC, MyBatis Spring Boot Starter 4.0.1 |
| 데이터베이스 | Oracle Database, Oracle JDBC (`ojdbc17`) |
| 파일 저장소 | AWS S3, AWS SDK for Java v2 |
| API 문서 | springdoc-openapi 3.1.0, Swagger UI |
| 모니터링 | Spring Boot Actuator, Micrometer Prometheus |
| 테스트 | JUnit Jupiter, Mockito, Spring MVC 테스트 |
| 빌드 | Gradle Wrapper 9.7.1 |
| CI/CD | GitHub Actions, EC2 전송 및 Docker Compose 재시작 |

## 프로젝트 구조

도메인별로 Controller, Service, Mapper, DTO를 구분하며, SQL은 MyBatis XML 매퍼에서 관리합니다.

```text
src/
  main/
    java/com/allergyout/
      AllergyOutApplication.java
      auth/        # 회원가입, 로그인, 토큰 관리
      member/      # 회원 정보와 프로필 관리
      allergy/     # 개인별 알레르기 정보
      recipe/      # 레시피 작성, 검색, 필터링, 추천
      bookmark/    # 즐겨찾기
      rasp/        # 디바이스 및 걸음 수 기록
      s3/          # 이미지 업로드와 삭제
      admin/       # 관리자 기능 기반 코드
      global/
        common/    # 공통 응답, 페이지 정보
        config/    # Security, S3 설정
        crypto/    # AES, HMAC 처리
        exception/ # 공통 예외 처리
        security/  # JWT 필터, 인증 정보, 쿠키 처리
    resources/
      mapper/      # MyBatis XML 매퍼
  test/
    java/com/allergyout/
.github/workflows/api-workflow.yml
```

`admin` 패키지는 존재하지만, 현재 `AdminController`에는 개별 API 메서드가 구현되어 있지 않습니다.

## 인증 및 데이터 처리

- Access Token은 로그인·재발급 응답으로 전달하며, 인증이 필요한 API에 `Authorization: Bearer <accessToken>` 헤더로 전송합니다.
- Refresh Token은 `HttpOnly`, `SameSite=Lax`, `Path=/api/auth` 쿠키로 전달합니다. `Secure` 속성은 `jwt.cookie.secure` 설정에 따릅니다.
- 비밀번호는 BCrypt로 해시 처리합니다.
- 이메일과 연락처는 AES-256-GCM으로 암호화하며, 중복 확인에는 별도의 HMAC-SHA256 해시를 사용합니다.
- 레시피 수정·삭제는 작성자 본인만 가능합니다. 레시피 삭제는 `DEL_YN`을 변경하는 소프트 삭제 방식입니다.
- 이미지 업로드는 JPG, JPEG, PNG 확장자를 허용하며 파일당 최대 크기는 5 MiB입니다.

## 주요 API

`선택`은 비로그인 상태에서도 호출할 수 있고, 로그인 시 개인화된 결과를 제공한다는 의미입니다.

| 메서드 | 경로 | 기능 | 인증 |
| --- | --- | --- | --- |
| POST | `/api/auth/signup` | 회원가입 | 불필요 |
| POST | `/api/auth/login` | 로그인 | 불필요 |
| POST | `/api/auth/refresh` | Access Token 재발급 | Refresh Token 쿠키 |
| POST | `/api/auth/logout` | 로그아웃 | 쿠키 기반 처리 |
| GET | `/api/auth/me` | 로그인 회원 조회 | 필요 |
| GET | `/api/members` | 내 정보 조회 | 필요 |
| PATCH | `/api/members/membername` | 이름 변경 | 필요 |
| PATCH | `/api/members/email` | 이메일 변경 | 필요 |
| PATCH | `/api/members/phone` | 연락처 변경 | 필요 |
| PATCH | `/api/members/memberpwd` | 비밀번호 변경 | 필요 |
| PATCH / DELETE | `/api/members/memberimg` | 프로필 이미지 변경·삭제 | 필요 |
| DELETE | `/api/members` | 회원 탈퇴 | 필요 |
| GET / PATCH | `/api/members/allergy` | 알레르기 정보 조회·수정 | 필요 |
| GET | `/api/recipes` | 레시피 목록·검색·필터 | 선택 |
| GET | `/api/recipes/count` | 레시피 총 개수 | 불필요 |
| GET | `/api/recipes/{recipeNo}` | 레시피 상세 | 선택 |
| GET | `/api/recipes/me` | 내가 작성한 레시피 | 필요 |
| GET | `/api/recipes/recommend` | 날짜 기반 추천 | 선택 |
| GET | `/api/recipes/recommend/calorie` | 칼로리 기반 추천 | 필요 |
| POST | `/api/recipes` | 레시피 등록 | 필요 |
| PATCH / DELETE | `/api/recipes/{recipeNo}` | 레시피 수정·삭제 | 작성자 |
| GET / POST | `/api/bookmarks` | 즐겨찾기 목록·등록 | 필요 |
| DELETE | `/api/bookmarks/{recipeNo}` | 즐겨찾기 해제 | 필요 |
| GET / POST | `/api/rasp/devices` | 내 디바이스 조회·등록 | 필요 |
| POST | `/api/rasp/steps` | 디바이스의 누적 걸음 수 저장 | 불필요 |
| GET | `/api/rasp/steps/day` | 오늘 걸음 수 기록 | 필요 |
| GET | `/api/rasp/steps/week` | 최근 7일 걸음 수 기록 | 필요 |

### 조회 파라미터 예시

```http
GET /api/recipes?page=0&size=20&sort=latest&applyMyAllergy=true
GET /api/recipes/recommend?date=2026-09-17
GET /api/recipes/recommend/calorie?totalCalories=2100
```

목록 조회는 `keyword`, `excludeMaterials`, `recipeType`, `cookingMethod`를 추가로 지원합니다. 페이지는 0부터 시작하며, 기본 크기는 20, 최대 크기는 50입니다. `sort=popular`는 조회수 기준 인기순입니다.

레시피 등록·수정과 프로필 이미지 변경은 `multipart/form-data`로 요청합니다. 레시피 폼은 `recipeMainImg`, `materialList[0].materialName`, `stepList[0].stepInfo`, `stepList[0].stepImg`처럼 DTO에 대응하는 필드명을 사용합니다. 레시피 수정 시 재료·조리 단계 목록은 수정 후 최종 상태 전체를 전달해야 합니다.

### 공통 응답

```json
{
  "code": 200,
  "msg": "레시피 목록 조회 성공했습니다.",
  "data": {
    "recipes": [],
    "pageInfo": {}
  }
}
```

위 예시는 응답 구조를 보여주기 위해 `pageInfo`의 세부 필드를 생략했습니다. 성공 시 결과는 `data`에 담기며, 오류 시 `msg`와 필요한 경우 `data`에 검증 정보를 반환합니다.

## 로컬 실행

### 1. 준비 사항

- JDK 21
- Oracle Database 접속 정보와 프로젝트 스키마
- AWS S3 버킷 및 객체 업로드·삭제 권한이 있는 자격 증명
- JWT 서명 키, 개인정보 암호화 키, HMAC 키

**현재 저장소에는 `application.yml`, DB 생성·초기 데이터 SQL, Docker Compose 설정이 포함되어 있지 않습니다.** 실행 전에 프로젝트에 맞는 DB 스키마를 별도로 준비해야 합니다. Gradle은 저장소에 포함된 Wrapper를 사용합니다.

### 2. 저장소 복제

```bash
git clone https://github.com/lno001/allergy-out-BE.git
cd allergy-out-BE
```

### 3. 애플리케이션 설정

`src/main/resources/application.yml`에 아래와 같은 로컬 설정을 구성합니다. 실제 운영 설정이 아닌 예시이며, `${...}`에 해당하는 환경변수를 실행 환경에 지정해야 합니다. 해당 경로의 설정 파일은 `.gitignore`에 등록되어 있습니다.

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

| 환경변수 | 설명 |
| --- | --- |
| `DB_URL` | Oracle JDBC URL. 예: `jdbc:oracle:thin:@//localhost:1521/XEPDB1` |
| `DB_USERNAME`, `DB_PASSWORD` | 프로젝트 스키마의 DB 계정 정보 |
| `JWT_SECRET` | 최소 32바이트의 충분히 무작위적인 서명 키 문자열. 현재 코드는 문자열 바이트를 직접 사용 |
| `APP_CRYPTO_AES_KEY` | 32바이트 키를 Base64로 인코딩한 값 |
| `APP_CRYPTO_HMAC_KEY` | AES 키와 별개인 32바이트 키를 Base64로 인코딩한 값 |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | S3 접근용 자격 증명 |
| `AWS_REGION` | S3 버킷 리전. 예: `ap-northeast-2` |
| `AWS_S3_BUCKET` | 사용할 버킷 이름 |

JWT 만료 시간 단위는 밀리초이며, 예시에서는 Access Token 30분, Refresh Token 7일로 설정했습니다. `max-request-size: 50MB`는 여러 이미지를 포함한 요청을 위한 예시 값입니다. HTTPS 운영 환경에서는 `jwt.cookie.secure=true`로 설정합니다.

현재 CORS 설정에는 로컬 프런트엔드 주소 `http://localhost:5173`이 포함되어 있습니다. 다른 주소를 사용한다면 `SecurityConfig`의 허용 출처 설정을 조정해야 합니다.

### 4. 실행

Windows PowerShell:

```powershell
.\gradlew.bat bootRun
```

macOS / Linux:

```bash
chmod +x gradlew
./gradlew bootRun
```

위 예시 설정으로 실행한 경우 다음 주소를 사용합니다.

- API 기본 주소: `http://localhost:8080`
- Swagger UI: `http://localhost:8080/swagger-ui/index.html`
- OpenAPI JSON: `http://localhost:8080/v3/api-docs`
- 상태 확인: `http://localhost:8080/actuator/health`
- Prometheus 메트릭: `http://localhost:8080/actuator/prometheus`

## 테스트 및 빌드

Windows PowerShell:

```powershell
.\gradlew.bat test
.\gradlew.bat clean build
```

macOS / Linux:

```bash
./gradlew test
./gradlew clean build
```

회원·알레르기·레시피·즐겨찾기·걸음 수 서비스, S3 파일 검증, 개인정보 암호화, 요청 유효성 검증 및 레시피 컨트롤러 바인딩 테스트가 포함되어 있습니다.

테스트 보고서는 `build/reports/tests/test/index.html`, 실행 가능한 JAR는 `build/libs/`에 생성됩니다. 일반 `plain.jar` 생성은 비활성화되어 있습니다.

## CI/CD

`.github/workflows/api-workflow.yml`에 정의된 흐름은 다음과 같습니다.

1. `main`을 대상으로 한 Pull Request에서는 JDK 21 환경에서 `clean build`로 빌드와 테스트를 수행합니다.
2. `main`에 push되면 동일한 검증 후 실행 JAR를 `AllergyOut.jar`로 변경합니다.
3. GitHub Secrets의 `HOST`, `USERNAME`, `SSH_KEY`를 사용해 EC2의 `/home/ubuntu/app`으로 JAR를 전송합니다.
4. EC2에서 `docker compose restart backend`를 실행합니다.

배포 서버에는 `backend` 서비스와 JAR 마운트가 설정된 Docker Compose 환경이 별도로 준비되어 있어야 합니다. 현재 워크플로는 컨테이너 재시작과 상태 출력을 수행하며, HTTP 헬스체크 명령은 주석 상태입니다.
