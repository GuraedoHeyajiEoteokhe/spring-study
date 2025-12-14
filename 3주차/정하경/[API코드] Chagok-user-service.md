### Controller

```java
@RestController
@RequestMapping("/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthService authService;

    @PostMapping("/login")
    public ResponseEntity<ApiResponse<TokenResponse>> login(@RequestBody LoginRequest request) {
        TokenResponse token = authService.login(request);
        return ResponseEntity.ok(ApiResponse.success(token));
    }

    @PostMapping("/logout")
    public ResponseEntity<ApiResponse<Void>> logout(@RequestBody RefreshTokenRequest request){
        authService.logout(request.getRefreshToken());
        return ResponseEntity.ok(ApiResponse.success(null));
    }
}

```

---

### Service

```java
@Service
@RequiredArgsConstructor
public class AuthService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtTokenProvider jwtTokenProvider;
    private final RefreshTokenRepository refreshTokenRepository;

    public TokenResponse login(LoginRequest request) {

        if (request.getEmail() == null || request.getEmail().isEmpty() ||
            request.getPassword() == null || request.getPassword().isEmpty()) {
            throw new BusinessException(ErrorCode.REQUIRED_FIELD_MISSING);
        }

        User user = userRepository.findByEmail(request.getEmail())
                .orElseThrow(() -> new BusinessException(ErrorCode.INVALID_CREDENTIALS));

        if(!passwordEncoder.matches(request.getPassword(), user.getPassword())) {
            throw new BusinessException(ErrorCode.INVALID_CREDENTIALS);
        }

        String accessToken = jwtTokenProvider.createAccessToken(user.getEmail(), user.getRole().name(), user.getUserNo());
        System.out.println("accessToken = " + accessToken);
        String refreshToken = jwtTokenProvider.createRefreshToken(user.getEmail());
        System.out.println("refreshToken = " + refreshToken);

        RefreshToken refreshTokenEntity = RefreshToken.builder()
                .email(user.getEmail())
                .token(refreshToken)
                .expiryDate(jwtTokenProvider.getRefreshTokenExpiryDate(refreshToken))
                .build();

        refreshTokenRepository.save(refreshTokenEntity);

        return TokenResponse.builder()
                .accessToken(accessToken)
                .refreshToken(refreshToken)
                .build();
    }

    @Transactional
    public void logout(String refreshToken) {
        jwtTokenProvider.validateToken(refreshToken);

        String email = jwtTokenProvider.getEmailFromJWT(refreshToken);

        RefreshToken storedToken = refreshTokenRepository.findById(email)
                .orElseThrow(() -> new BusinessException(ErrorCode.REFRESH_TOKEN_NOT_FOUND));
        if(!storedToken.getToken().equals(refreshToken)) {
            throw new BusinessException(ErrorCode.REFRESH_TOKEN_MISMATCH);
        }

        if (storedToken.getExpiryDate().before(new Date())) {
            throw new BusinessException(ErrorCode.REFRESH_TOKEN_EXPIRED);
        }

        refreshTokenRepository.deleteByEmail(email);
    }
}
```

---

### Login관련

#### DTO

```java
@Getter
@RequiredArgsConstructor
public class LoginRequest {

    private String email;
    private String password;

}
```

```java
@Getter
@Builder
public class TokenResponse {

    private String accessToken;
    private String refreshToken;
}
```

---

#### Entity

```java
@Entity
@Table(name = "tbl_user")
@NoArgsConstructor
@Getter
@ToString
public class User {

    @Id
    @Column(name = "user_no", unique = true, nullable = false)
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long userNo;

    @Column(name = "email", unique = true, nullable = false)
    private String email;

    @Column(name = "password", nullable = false)
    private String password;

    @Column(name = "user_name", nullable = false)
    private String userName;

    @Column(name = "nickname", unique = true ,nullable = false)
    private String nickname;

    @Column(name = "car_number", unique = true, nullable = false)
    private String carNumber;

    @Enumerated(EnumType.STRING)
    @Column(name = "role", nullable = false)
    private UserRole role = UserRole.USER;

    public void setEncodedPassword(String encodedPassword) {
        this.password = encodedPassword;
    }

    public void setCarNumber(String carNumber) {
        this.carNumber = carNumber;
    }

}
```

#### 로그인 후 나오는 refreshToken DB에 저장

```java
@Entity
@Table(name = "refresh_token")
@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class RefreshToken {

    @Id
    @Column(nullable = false)
    private String email;

    @Column(nullable = false)
    private String token;

    @Column(nullable = false)
    private Date expiryDate;

}
```

---

### Logout관련

#### DTO

```java
@Getter
@RequiredArgsConstructor
public class RefreshTokenRequest {
    private final String refreshToken;
}
```

```java
@Entity
@Table(name = "refresh_token")
@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class RefreshToken {

    @Id
    @Column(nullable = false)
    private String email;

    @Column(nullable = false)
    private String token;

    @Column(nullable = false)
    private Date expiryDate;

}
```
