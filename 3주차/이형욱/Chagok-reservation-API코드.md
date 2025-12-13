### Command - Controller 코드
```java
// 예약 생성
    @PostMapping("/reservation/createReservation")
    public ResponseEntity<ApiResponse<ReservationCommandResponse>> createReservation(
            @RequestBody ReservationCreateRequest request, @AuthenticationPrincipal CustomUserDetails userDetails
    ) {
        Long reservationId = reservationCommandService.createReservation(request, userDetails.getUserNo());
        ReservationCommandResponse reservationCommandResponse = ReservationCommandResponse.builder()
                .reservationId(reservationId)
                .build();
        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(ApiResponse.success(reservationCommandResponse));
    }

    // 결제 호출
    @PostMapping("/payment/{reservationId}")
    public ResponseEntity<ApiResponse<PaymentResponse>> createPayment(
            @PathVariable Long reservationId
    )   {
        PaymentResponse response = reservationCommandService.reservationPayment(reservationId);
        return ResponseEntity
                .status(HttpStatus.OK)
                .body(ApiResponse.success(response));
    }

    // 예약 이용 시작
    @PutMapping("/reservation/start")
    public ResponseEntity<ApiResponse<UsedSpotsUpdateResponse>> startReservation(
            @RequestBody ReservationStartRequest request
    )   {
        UsedSpotsUpdateResponse response = reservationCommandService.startReservation(request);
        return ResponseEntity
                .status(HttpStatus.OK)
                .body(ApiResponse.success(response));
    }

    // 예약 이용 종료
    @PutMapping("/reservation/quit")
    public ResponseEntity<ApiResponse<UsedSpotsUpdateResponse>> quitReservation(
            @RequestBody ReservationEndRequest request, @AuthenticationPrincipal UserDetails user
    )   {
        UsedSpotsUpdateResponse response = reservationCommandService.finishReservation(request);
        return ResponseEntity
                .status(HttpStatus.OK)
                .body(ApiResponse.success(response));
    }
```
### Command-Service 코드
```java
// 예약 생성
    @Transactional
    public Long createReservation(ReservationCreateRequest request, Long userNo) {

        // parkinglot 예외처리
        if(parkingLotClient.getParkinglotBaseInfo(request.getParkingLotId()).getBody() == null){
                throw new BusinessException(ErrorCode.REGIST_ERROR_NO_PARKINGLOT);
            }

        LocalDateTime startTime = localDateTimeConverter.convert(request.getStartTime());
        ResponseEntity<ApiResponse<BaseInfoResponse>> response = parkingLotClient
                .getParkinglotBaseInfo(request.getParkingLotId());

        int baseTimeMinutes;
        if (response.getBody() != null) {
            baseTimeMinutes = response.getBody().getData().getBaseTime();
        } else throw new BusinessException(ErrorCode.REGIST_ERROR_NO_PARKINGLOT);

        LocalDateTime endTime;
        if (startTime != null) {
            endTime = (startTime).plusMinutes(baseTimeMinutes);
        } else throw new BusinessException(ErrorCode.VALIDATION_ERROR_TIME_UNAVAILABLE);

        if (reservationCommandRepository.existsByParkinglotIdAndIsCanceledFalseAndEndTimeGreaterThanAndStartTimeLessThan(
                // 겹치는 시간이 있을 경우
                response.getBody().getData().getParkinglotId(),
                startTime,
                endTime
        )) {
            throw new BusinessException(ErrorCode.REGIST_ERROR_TIME_CONFLICT);
        }

        Reservation newReservation = Reservation.create(
                startTime,
                endTime,
                false,
                LocalDateTime.now(),
                userNo,
                request.getParkingLotId()
        );

        Reservation saved = reservationCommandRepository.save(newReservation);

        if (saved.getReservationId() == null) {
            throw new BusinessException(ErrorCode.REGIST_ERROR);
        }

        return saved.getReservationId();
    }


    // Payment Response 생성
    public PaymentResponse reservationPayment(Long reservationId) {
        Reservation reservation = reservationCommandRepository.findByReservationIdAndIsCanceledFalse(reservationId);

        if (reservation == null) {
            throw new BusinessException(ErrorCode.VALIDATION_ERROR_WRONG_RESERVATIONID);
        }

        ResponseEntity<ApiResponse<BaseInfoResponse>> response = parkingLotClient
                .getParkinglotBaseInfo(reservation.getParkinglotId());
        Integer totalAmount;

        if (response.getBody() != null) {
            totalAmount = response.getBody().getData().getBaseFee();
        } else throw new  BusinessException(ErrorCode.REGIST_ERROR_NO_PARKINGLOT);

        return PaymentResponse.builder()
                .reservationId(reservationId)
                .parkinglotId(response.getBody().getData().getParkinglotId())
                .totalAmount(totalAmount)
                .parkinglotName(response.getBody().getData().getName())
                .build();
    }

    public UsedSpotsUpdateResponse startReservation(ReservationStartRequest request) {
        LocalDateTime currTime = localDateTimeConverter.convert(request.getUpdateTime());
        Reservation reservation = reservationCommandRepository.findByReservationIdAndIsCanceledFalse(request.getReservationId());

        boolean hasReservationStarted;
        if (currTime != null) {
            hasReservationStarted = reservation.isStarted(currTime, reservation.getStartTime(), reservation.getEndTime());
        } else throw new BusinessException(ErrorCode.VALIDATION_ERROR_TIME_UNAVAILABLE);

        if (!hasReservationStarted) {
            throw new BusinessException(ErrorCode.VALIDATION_ERROR_EARLY_START);
        }

        UsedSpotsUpdateResponse response =  UsedSpotsUpdateResponse.builder()
                .parkinglotId(request.getParkinglotId())
                .using(true)
                .build();
        parkingLotClient.updateUsedSpots(response);
        return response;
    }

    public UsedSpotsUpdateResponse finishReservation(ReservationEndRequest request) {
        UsedSpotsUpdateResponse response = UsedSpotsUpdateResponse.builder()
                .parkinglotId(request.getParkinglotId())
                .using(false)
                .build();
        parkingLotClient.updateUsedSpots(response);
        return response;
    }
```
### Command - dto
#### Command - dto - request
```java
@Getter
@AllArgsConstructor
public class ReservationCommandRequest {
    private Long parkingLotId;
    private String time;
}
```
```java
@Getter
@AllArgsConstructor
@Builder
public class ReservationCreateRequest {
    private Long parkingLotId;
    private String startTime;
}
```
```java
@Getter
@Builder
public class ReservationEndRequest {
    private Long parkinglotId;
    private Long reservationId;
}
```
```java
@Getter
@Builder
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor
public class ReservationStartRequest {
    private Long parkinglotId;
    private Long reservationId;
    private String updateTime;
}

```
#### Command - dto - response
```java
@Getter
@Builder
public class BaseInfoResponse {
    private final Long parkinglotId;
    private final String name;
    private final Integer baseFee;
    private final Integer baseTime;
}
```
```java
@Getter
@Builder
public class PaymentResponse {
    private Long reservationId;
    private Long parkinglotId;
    private Integer totalAmount;
    private String parkinglotName;
}
```
```java
@Getter
@Builder
public class ReservationCommandResponse {
    private Long reservationId;
}
```
```java
@Getter
@Builder
@RequiredArgsConstructor
public class UsedSpotsUpdateResponse {
    private final boolean using;
    private final Long parkinglotId;
}
```

### Query - Controller
```java
@GetMapping("/mypage/reservations/{reservationId}")
    public ResponseEntity<ApiResponse<ReservationQueryResponse>> findByReservationId(
            @PathVariable Long reservationId
    ){
        ReservationQueryResponse response = reservationQueryService.getReservationDetailBy(reservationId);
        return ResponseEntity.ok(ApiResponse.success(response));
    }

    @GetMapping("/mypage/reservation/search/")
    public ResponseEntity<ApiResponse<ReservationListResponse>> findByUserId(
            @AuthenticationPrincipal CustomUserDetails userDetail
    ) {
        ReservationListResponse response = reservationQueryService.getReservationsByUserNo(userDetail.getUserNo());
        return ResponseEntity.ok(ApiResponse.success(response));
    }

    // review API: Review 생성용 reservation정보 전달
    @GetMapping("/reservation/{reservationId}")
    public ApiResponse<ReviewReservationInfoResponse> getInfoForReview(
            @PathVariable("reservationId") Long reservationId
    ) {
        Reservation reservation = reservationQueryService.getByReservationId(reservationId);
        ReviewReservationInfoResponse response = ReviewReservationInfoResponse.builder()
                .parkinglotId(reservation.getParkinglotId())
                .userNo(reservation.getUserNo())
                .reservationId(reservation.getReservationId())
                .build();
        return ApiResponse.success(response);
    }

    // payment API: Payment 생성용 reservation정보 전달
    @GetMapping("/payment/{reservationId}")
    public ApiResponse<PaymentResponse> getInfoForPaymentReservation(
            @PathVariable Long reservationId
    )   {
        PaymentResponse response = reservationQueryService.getInfoForPaymentReservation(reservationId);
        return ApiResponse.success(response);
    }
```
### Query - service
```java
// 단일객체 조회(reservationId)
    @Transactional(readOnly = true)
    public ReservationQueryResponse getReservationDetailBy(Long reservationId) {

        Reservation reservation = reservationQueryRepository
                .findByReservationId(reservationId);
//                .orElseThrow(() -> new BusinessException(ErrorCode.NOT_FOUND));
        ReservationDto response = ReservationDto.from(reservation);
        return ReservationQueryResponse.builder()
                .reservationDto(response)
                .build();
    }

    // 리스트 객체 조회(userNo)
    @Transactional(readOnly = true)
    public ReservationListResponse getReservationsByUserNo(Long userNo) {
        List<Reservation> reservations = reservationQueryRepository
                .findAllByUserNoOrderByCreatedAtDesc(userNo);
        List<ReservationDto> reservationDtos = reservations.stream()
                .map(ReservationDto::from)
                .toList();
        return ReservationListResponse.builder()
                .reservationDtoList(reservationDtos)
                .build();

    }

    public Reservation getByReservationId(Long reservationId) {
        return reservationQueryRepository.findByReservationId(reservationId);
//                .orElseThrow(() -> new BusinessException(ErrorCode.NOT_FOUND));
    }

    // Payment API service
    public PaymentResponse getInfoForPaymentReservation(Long reservationId) {
        Reservation reservation = reservationQueryRepository.findByReservationId(reservationId);
//        Reservation entityReservation = reservation.orElseThrow(() -> new BusinessException(ErrorCode.NOT_FOUND));

        ResponseEntity<ApiResponse<BaseInfoResponse>> responseEntity =
                parkingLotClient.getParkinglotBaseInfo(reservation.getParkinglotId());

        ApiResponse<BaseInfoResponse> body = responseEntity.getBody();

        // 1단계: body 자체 검증
        if (body == null) {
            throw new BusinessException(ErrorCode.REGIST_ERROR_NO_PARKINGLOT);
        }

        // 2단계: success 여부 검증
        if (!body.isSuccess()) {
            // 필요하면 errorCode 보고 분기
            throw new BusinessException(ErrorCode.REGIST_ERROR_NO_PARKINGLOT);
        }

        // 3단계: data 존재 여부 검증
        BaseInfoResponse info = body.getData();
        if (info == null) {
            throw new BusinessException(ErrorCode.REGIST_ERROR_NO_PARKINGLOT);
        }

        // 4단계: 이제 안심하고 사용
        Integer amount = info.getBaseFee();
        String name = info.getName();


        return PaymentResponse.builder()
                .parkinglotId(reservation.getParkinglotId())
                .reservationId(reservationId)
                .parkinglotName(name)
                .totalAmount(amount)
                .build();
    }
```
#### Query - dto - response
```java
@Getter
@Builder
public class ReservationListResponse {
    private final List<ReservationDto> reservationDtoList;
}
```
```java
public class ReservationQueryResponse {

    private final ReservationDto reservationDto;
}
```
```java
@Getter
@Setter
@Builder
public class ReviewReservationInfoResponse {
    private Long reservationId;
    private Long userNo;
    private Long parkinglotId;
}
```
### Common dto
```java
public class ReservationDto {

    private long reservationId;
    private LocalDateTime startTime;
    private LocalDateTime endTime;
    private boolean isCanceled;
    private LocalDateTime createdAt;
    private long userNo;
    private long parkinglotId;

    public static ReservationDto from(Reservation reservation) {
        return ReservationDto.builder()
                .reservationId(reservation.getReservationId())
                .startTime(reservation.getStartTime())
                .endTime(reservation.getEndTime())
                .isCanceled(reservation.getIsCanceled())
                .createdAt(reservation.getCreatedAt())
                .userNo(reservation.getUserNo())
                .parkinglotId(reservation.getParkinglotId())
                .build();
    }
}
```
### ErrorCode
```java
@Getter
@AllArgsConstructor(access = AccessLevel.PACKAGE)
public enum ErrorCode {
    // 등록,수정,삭제 관련 오류
    REGIST_ERROR("10001", "등록에 실패하였습니다.", HttpStatus.INTERNAL_SERVER_ERROR),
    REGIST_ERROR_TIME_CONFLICT("10003", "겹치는 시간입니다.",HttpStatus.INTERNAL_SERVER_ERROR),
    REGIST_ERROR_NO_PARKINGLOT("10004", "없는 주차장을 선택하셨습니다.",HttpStatus.INTERNAL_SERVER_ERROR),
    // 조회 관련 오류
    NOT_FOUND("20001", "조회에 실패하였습니다.",  HttpStatus.NOT_FOUND),
    RESERVATION_NOT_FOUND("20002", "예약 정보를 조회할 수 업습니다.", HttpStatus.NOT_FOUND),

    // request 값 입력 오류
    VALIDATION_ERROR("40001", "입력 값 검증 오류입니다.", HttpStatus.BAD_REQUEST),
    VALIDATION_ERROR_EARLY_START("40002", "예약 시간보다 이릅니다.", HttpStatus.BAD_REQUEST),
    VALIDATION_ERROR_TIME_UNAVAILABLE("40003", "입력된 시간 정보가 잘못되었습니다.",HttpStatus.BAD_REQUEST ),
    VALIDATION_ERROR_WRONG_RESERVATIONID("40004", "해당 예약번호는 유효하지 않습니다.", HttpStatus.BAD_REQUEST ),

    // 기타 오류
    INTERNAL_SERVER_ERROR("50001", "내부 서버 오류입니다.", HttpStatus.INTERNAL_SERVER_ERROR),
    ;

    private final String code;
    private final String message;
    private final HttpStatus httpStatus;
}
```