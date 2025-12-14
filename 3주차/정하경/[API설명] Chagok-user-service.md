# JWT 인증 API 정리

## 1. 전체 구조 개요

```
클라이언트
  ↓ HTTP 요청
AuthController (API 진입점)
  ↓ 비즈니스 위임
AuthService (인증 로직)
  ↓ DB 접근
UserRepository / RefreshTokenRepository
```

* **Controller**: HTTP 요청/응답 담당
* **Service**: 인증, 토큰 발급, 검증 로직
* **Repository**: DB 접근
* **JWT**: 서버 상태를 저장하지 않는 인증 방식

---

## 2. AuthController 설명

```java
@RestController
@RequestMapping("/auth")
@RequiredArgsConstructor
public class AuthController {
    private final AuthService authService;
}
```

### 역할

* `/auth` 경로의 API 엔드포인트 제공
* 비즈니스 로직 없이 Service 호출만 담당

---

### 2-1. 로그인 API (`POST /auth/login`)

**요청**

```json
{
  "email": "test@test.com",
  "password": "1234"
}
```

**동작 흐름**

1. 요청 데이터를 `LoginRequest`로 변환
2. `authService.login()` 호출
3. AccessToken, RefreshToken 반환

---

### 2-2. 로그아웃 API (`POST /auth/logout`)

**핵심 기능**

* DB에 저장된 RefreshToken 삭제
* 이후 토큰 재발급 불가

---

## 3. AuthService 설명

```java
@Service
@RequiredArgsConstructor
public class AuthService {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtTokenProvider jwtTokenProvider;
    private final RefreshTokenRepository refreshTokenRepository;
}
```

### 역할

* **인증 비즈니스 로직의 중심**
* Controller에서 받은 요청을 실제로 처리
* 사용자 검증, 토큰 생성/검증, DB 제어 담당

### 의존성 설명

| 필드                     | 역할                    |
| ---------------------- | --------------------- |
| UserRepository         | 사용자 조회 (이메일 기준)       |
| PasswordEncoder        | 비밀번호 암호화 비교           |
| JwtTokenProvider       | JWT 생성/검증             |
| RefreshTokenRepository | RefreshToken 저장/조회/삭제 |

---

## 4. 로그인 로직 (`login()`)

### 4-1. 필수값 검증

```java
if (request.getEmail() == null || request.getEmail().isEmpty())
```

* 필수 입력값 누락 방지

---

### 4-2. 사용자 조회

```java
userRepository.findByEmail(email)
```

* 이메일 기준 사용자 조회
* 없으면 `INVALID_CREDENTIALS`

---

### 4-3. 비밀번호 검증

```java
passwordEncoder.matches(rawPassword, encodedPassword)
```

* 입력 비밀번호와 DB 암호 비교

---

### 4-4. JWT 토큰 생성

```java
accessToken = jwtTokenProvider.createAccessToken(...)
refreshToken = jwtTokenProvider.createRefreshToken(...)
```

* **AccessToken**: 짧은 만료 시간, API 요청용
* **RefreshToken**: 긴 만료 시간, 재발급용

---

### 4-5. RefreshToken 저장

```java
refreshTokenRepository.save(refreshTokenEntity)
```

* RefreshToken은 DB에서 관리
* 로그아웃, 탈취 대응 가능

---

## 5. 로그아웃 로직 (`logout()`)

```java
refreshTokenRepository.deleteByEmail(email)
```

* RefreshToken 삭제
* 이후 재발급 불가 → 로그아웃 완료

---

## 6. ❓ 왜 AccessToken은 삭제하지 않을까?

### 6-1. JWT의 핵심 특징: Stateless

* AccessToken은 **서버에 저장하지 않음**
* 서버는 토큰의 서명과 만료만 검증

---

### 6-2. AccessToken을 서버에서 관리하지 않는 이유

| 이유  | 설명                   |
| --- | -------------------- |
| 성능  | 매 요청마다 DB 조회 불필요     |
| 확장성 | 서버가 늘어나도 상태 공유 불필요   |
| 구조  | JWT 설계 자체가 Stateless |

---

### 6-3. 로그아웃 시 AccessToken은?

* 클라이언트가 그냥 버림
* 서버는 **만료될 때까지 유효하다고 간주**

---
