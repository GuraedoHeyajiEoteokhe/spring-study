# 단위 프로젝트 API 리뷰

# Reservation Service API

본 문서는 **ValetParker Reservation Service**에서 구현된 API를 정리한 문서이다.

API는 역할에 따라 **Command API(상태 변경)** 와 **Query API(조회 전용)** 로 분리되어 있다.

---

## 1. Command API (예약 상태 변경)

> 예약 생성, 결제 요청, 이용 시작/종료 등
>
>
> **데이터를 변경하는 책임을 가지는 API**
>

패키지 경로

```
command
 ├─ controller
 ├─ service
 ├─ repository
 └─ dto (request / response)

```

---

### 1-1. 예약 생성 API

#### API 개요

- 사용자가 주차장 예약을 생성한다.
- 예약 시간, 주차장 ID를 기준으로 예약 가능 여부를 검증한 뒤 예약을 저장한다.

#### Endpoint

```
POST /reservation/createReservation

```

#### Controller

```java
@PostMapping("/reservation/createReservation")
public ResponseEntity<ApiResponse<ReservationCommandResponse>> createReservation(
        @RequestBody ReservationCreateRequest request,
        @AuthenticationPrincipal CustomUserDetails userDetails
) {
    Long reservationId =
            reservationCommandService.createReservation(
                    request,
                    userDetails.getUserNo()
            );

    ReservationCommandResponse response =
            ReservationCommandResponse.builder()
                    .reservationId(reservationId)
                    .build();

    return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(ApiResponse.success(response));
}

```

#### Service

```java
public Long createReservation(ReservationCreateRequest request, Long userNo)

```

#### Request DTO

```java
public class ReservationCreateRequest {
    private Long parkingLotId;
    private String startTime;
}

```

#### Response DTO

```java
public class ReservationCommandResponse {
    private Long reservationId;
}

```

#### 동작 설명

1. 인증된 사용자로부터 `userNo`를 추출한다.
2. 요청된 `parkingLotId`가 존재하는지 확인한다.
3. `startTime`을 기준으로 주차장의 기본 이용 시간(`baseTime`)을 적용해 `endTime`을 계산한다.
4. 동일 시간대에 이미 예약이 존재하는지 검증한다.
5. 모든 검증 통과 시 예약을 저장하고 `reservationId`를 반환한다.

---

### 1-2. 결제 호출 API

#### API 개요

- 예약된 내역을 기반으로 결제에 필요한 정보를 생성한다.
- Payment Service에서 사용될 정보를 반환한다.

#### Endpoint

```
POST /payment/{reservationId}

```

#### Controller

```java
@PostMapping("/payment/{reservationId}")
public ResponseEntity<ApiResponse<PaymentResponse>> createPayment(
        @PathVariable Long reservationId
) {
    PaymentResponse response =
            reservationCommandService.reservationPayment(reservationId);

    return ResponseEntity
            .status(HttpStatus.OK)
            .body(ApiResponse.success(response));
}

```

#### Service

```java
public PaymentResponse reservationPayment(Long reservationId)

```

#### Response DTO

```java
public class PaymentResponse {
    private Long reservationId;
    private Long parkinglotId;
    private Integer totalAmount;
    private String parkinglotName;
}

```

#### 동작 설명

1. 예약 ID를 기준으로 예약 정보를 조회한다.
2. 주차장 기본 요금(`baseFee`)을 조회한다.
3. 결제에 필요한 정보들을 조합하여 Payment Service로 전달할 응답을 생성한다.

---

### 1-3. 예약 이용 시작 API

#### API 개요

- 예약 시간이 도래한 사용자가 실제 주차 이용을 시작할 때 호출한다.
- 주차장 사용 중 상태로 변경한다.

#### Endpoint

```
PUT /reservation/start

```

#### Controller

```java
@PutMapping("/reservation/start")
public ResponseEntity<ApiResponse<UsedSpotsUpdateResponse>> startReservation(
        @RequestBody ReservationStartRequest request
) {
    UsedSpotsUpdateResponse response =
            reservationCommandService.startReservation(request);

    return ResponseEntity
            .status(HttpStatus.OK)
            .body(ApiResponse.success(response));
}

```

#### Request DTO

```java
public class ReservationStartRequest {
    private Long reservationId;
    private Long parkinglotId;
    private String updateTime;
}

```

#### Response DTO

```java
public class UsedSpotsUpdateResponse {
    private Long parkinglotId;
    private boolean using;
}

```

#### 동작 설명

1. 현재 시간을 기준으로 예약 시작 가능 여부를 검증한다.
2. 예약 시간이 이전인 경우 예외를 발생시킨다.
3. 주차장 사용 중 상태로 변경하고 사용 중임을 알린다.

---

### 1-4. 예약 이용 종료 API

#### API 개요

- 사용자가 주차 이용을 종료할 때 호출한다.
- 주차장 사용 상태를 해제한다.

#### Endpoint

```
PUT /reservation/quit

```

#### Controller

```java
@PutMapping("/reservation/quit")
public ResponseEntity<ApiResponse<UsedSpotsUpdateResponse>> quitReservation(
        @RequestBody ReservationEndRequest request,
        @AuthenticationPrincipal UserDetails user
) {
    UsedSpotsUpdateResponse response =
            reservationCommandService.finishReservation(request);

    return ResponseEntity
            .status(HttpStatus.OK)
            .body(ApiResponse.success(response));
}

```

#### Request DTO

```java
public class ReservationEndRequest {
    private Long reservationId;
    private Long parkinglotId;
    private String updateTime;
}

```

#### 동작 설명

1. 예약 정보를 조회한다.
2. 종료 시간이 예약 시간 이전/이후인지 판단한다.
3. 주차장 사용 상태를 종료 처리한다.

---

## 2. Query API (조회 전용)

> 데이터 변경 없이
>
>
> **예약 정보 조회만 담당하는 API**
>

패키지 경로

```
query
 ├─ controller
 ├─ service
 └─ dto (response)

```

---

### 2-1. 예약 단건 조회 API

#### Endpoint

```
GET /mypage/reservations/{reservationId}

```

#### Controller

```java
@GetMapping("/mypage/reservations/{reservationId}")
public ResponseEntity<ApiResponse<ReservationQueryResponse>> findByReservationId(
        @PathVariable Long reservationId
) {
    ReservationQueryResponse response =
            reservationQueryService.getReservationDetailBy(reservationId);

    return ResponseEntity.ok(ApiResponse.success(response));
}

```

#### Response DTO

```java
public class ReservationQueryResponse {
    private Long reservationId;
    private Long parkinglotId;
    private LocalDateTime startTime;
    private LocalDateTime endTime;
    private boolean isCanceled;
}

```

#### 설명

- 특정 예약 ID에 대한 상세 정보를 조회한다.

---

### 2-2. 사용자 예약 목록 조회 API

#### Endpoint

```
GET /mypage/reservation/search/{userNo}

```

#### Controller

```java
@GetMapping("/mypage/reservation/search/{userNo}")
public ResponseEntity<ApiResponse<ReservationListResponse>> findByUserId(
        @PathVariable Long userNo
) {
    ReservationListResponse response =
            reservationQueryService.getReservationsByUserNo(userNo);

    return ResponseEntity.ok(ApiResponse.success(response));
}

```

#### Response DTO

```java
public class ReservationListResponse {
    private List<ReservationQueryResponse> reservations;
}

```

#### 설명

- 특정 사용자의 모든 예약 내역을 조회한다.

---

### 2-3. Review 생성용 예약 정보 조회 API

#### Endpoint

```
GET /reservation/{reservationId}

```

#### Response DTO

```java
public class ReviewReservationInfoResponse {
    private Long reservationId;
    private Long parkinglotId;
    private Long userNo;
}

```

#### 설명

- Review Service에서 리뷰 생성 시 필요한 예약 정보만 제공한다.

---

### 2-4. Payment 생성용 예약 정보 조회 API

#### Endpoint

```
GET /payment/{reservationId}

```

#### 설명

- Payment Service에서 결제 생성 시 예약 기반 정보를 조회한다.

---

## 정리

- **Command API**

  → 예약 생성, 결제 요청, 이용 시작/종료

  → 데이터 변경 책임

- **Query API**

  → 예약 조회 전용

  → 다른 서비스(Review, Payment)와 연동 목적

- 모든 응답은 `ApiResponse<T>` 공통 포맷을 사용한다.

---