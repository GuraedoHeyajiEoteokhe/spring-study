## 독서모임 후기글 파트 API 작성
독서모임 후기글, 댓글 CRUD, 좋아요 기능

### 후기글 생성
```java
    // 독서모임을 지정해 후기글을 작성함
    // 모임의 참가자만 후기글을 작성할 수 있으며 모임에 대한 후기는 한번만 작성이 가능함
    @PostMapping("/clubId/{clubId}")
    public ResponseEntity<ReadingClubReviewResponseDTO> createReview(@PathVariable Long clubId, @RequestBody ReadingClubReviewRequestDTO request,
                                                                     HttpServletRequest req){

        // api-gateway를 통해 로그인한 사용자의 ID를 가져옴
        String id = req.getHeader("X-User-ID");
        Long userId = Long.valueOf(id);

        ReadingClubReviewResponseDTO response = reviewService.createReview(clubId, request, userId);

        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

// Service
@Transactional
    public ReadingClubReviewResponseDTO createReview(Long club, ReadingClubReviewRequestDTO request, Long userId) {

        // 존재하는 클럽인지 체크 
        ReadingClub existClub = readingClubRepository.findById(club)
                .orElseThrow(() -> new IllegalArgumentException("존재하지 않는 클럽입니다."));

        Long clubId = existClub.getId();

        // 클럽 회원인지 확인
        boolean isMember = memberRepository.existsByClubIdAndUserId(clubId, userId);
        if (!isMember) {
            log.warn("유저 {} 는 클럽 {} 멤버가 아님", userId, clubId);
            throw new AccessDeniedException("해당 모임 참가자만 후기를 작성할 수 있습니다.");
        }

        // 이미 작성한 후기 있는지 체크
        boolean alreadyWritten = reviewRepository.existsByClubIdAndWriterId(existClub, userId);
        if (alreadyWritten) {
            throw new IllegalStateException("이 모임에 이미 후기를 작성하셨습니다.");
        }

        ReadingClubReview review = ReadingClubReview.builder()
                .clubId(existClub)
                .writerId(userId)
                .reviewTitle(request.getReviewTitle())
                .reviewContent(request.getReviewContent())
                .build();

        ReadingClubReview saved = reviewRepository.save(review);

        return ReadingClubReviewResponseDTO.from(saved);
    }
```
### 후기글 수정
```java
    //후기글 제목, 내용을 수정한다.
    // 자신이 작성한 후기글만 수정이 가능하다.
    @PutMapping("/reviewId/{reviewId}")
    public ResponseEntity<ReadingClubReviewRdifyReview(@PathVariable Long reviewId,
                                                                     @RequestBody ReadingClubReviewRequestDTO request,
        // api-gateway를 통해 로그인한 사용자의 ID를 가져옴                                                          HttpServletRequest req){
        String id = req.getHeader("X-User-ID");
        Long userId = Long.valueOf(id);

        ReadingClubReviewResponseDTO response = reviewService.modifyReview(reviewId, request, userId);

        return ResponseEntity.ok(response);
    }
// Service
  @Transactional
    public ReadingClubReviewResponseDTO modifyReview(Long reviewId, ReadingClubReviewRequestDTO request, Long userId) {


        // 1. 이 유저가 쓴 해당 리뷰 찾기
        ReadingClubReview review = reviewRepository
                .findByReviewIdAndWriterId(reviewId, userId)
                .orElseThrow(() -> new AccessDeniedException("해당 리뷰를 수정할 수 있는 권한이 없습니다."));

        // 2. 제목/내용 수정
        review.update(request.getReviewTitle(), request.getReviewContent());

        // 3. 저장 후 DTO로 변환
        ReadingClubReview saved = reviewRepository.save(review);
        return ReadingClubReviewResponseDTO.from(saved);
    }
```
### 후기글 삭제
```java
    // 후기글 삭제
    // 자신이 작성한 후기글만 삭제가 가능함 (Hard delete)
    @DeleteMapping("/reviewId/{reviewId}")
    public ResponseEntity<Void> deleteReview(@PathVariable Long reviewId,
                                             HttpServletRequest req)
    {
        // api-gateway를 통해 로그인한 사용자의 ID를 가져옴
        String id = req.getHeader("X-User-ID");
        Long userId = Long.valueOf(id);

        reviewService.deleteReview(reviewId, userId);

        return ResponseEntity.noContent().build();

    }
// Service
      @Transactional
    public void deleteReview(Long reviewId, Long userId) {
        //이 유저가 쓴 해당 리뷰 찾기
        ReadingClubReview review = reviewRepository
                .findByReviewIdAndWriterId(reviewId, userId)
                .orElseThrow(() -> new AccessDeniedException("해당 리뷰를 삭제할 수 있는 권한이 없습니다."));

        reviewRepository.delete(review);
    }
```
### 후기글 목록 조회
```java
    /**
     * 특정 모임의 후기글 목록 조회
     *  - sort=latest (기본값): 최신순
     *  - sort=like        : 좋아요 많은 순
     *  - page=0부터 시작, 한 페이지당 15개
     */
    @GetMapping("/clubId/{clubId}")
    public ResponseEntity<Page<ReadingClubReviewResponseDTO>> getReviews(
            @PathVariable Long clubId,
            @RequestParam(defaultValue = "latest") String sort,
            @RequestParam(defaultValue = "0") int page,
            HttpServletRequest req
    ) {
        // api-gateway를 통해 로그인한 사용자의 ID를 가져옴
        String id = req.getHeader("X-User-ID");
        Long userId = Long.valueOf(id);

        Page<ReadingClubReviewResponseDTO> result;

        if ("like".equals(sort)) { // like
            result = reviewService.getReviewsOrderByLike(clubId, page,userId);
        } else { // latest 
            result = reviewService.getReviewsOrderByLatest(clubId, page,userId);
        }

        return ResponseEntity.ok(result);
    }
// Service
      // 특정 모임 리뷰 – 최신순, 15개씩
    @Transactional(readOnly = true)
    public Page<ReadingClubReviewResponseDTO> getReviewsOrderByLatest(Long clubId, int page, Long userId) {


        Pageable pageable = PageRequest.of(page, 15);   // page: 0부터 시작, size: 15

        Page<ReadingClubReview> result =
                reviewRepository.findByClubId_IdOrderByCreatedAtDesc(clubId, pageable);

        return result.map(ReadingClubReviewResponseDTO::from);
    }

    // 특정 모임 리뷰 – 좋아요 많은 순, 15개씩
    @Transactional(readOnly = true)
    public Page<ReadingClubReviewResponseDTO> getReviewsOrderByLike(Long clubId, int page, Long userId) {


        Pageable pageable = PageRequest.of(page, 15);

        Page<ReadingClubReview> result =
                reviewRepository.findByClubId_IdOrderByLikeTotalDescCreatedAtDesc(clubId, pageable);

        return result.map(ReadingClubReviewResponseDTO::from);
    }
```
### 내 후기글 조회
```java
    // 내가 작성한 후기글 조회
    @GetMapping("/my")
    public ResponseEntity<Page<ReadingClubReviewResponseDTO>> getMyReviews(@RequestParam(defaultValue = "0") int page,HttpServletRequest req)
    {
        // api-gateway를 통해 로그인한 사용자의 ID를 가져옴
        String id = req.getHeader("X-User-ID");
        Long userId = Long.valueOf(id);

        Page<ReadingClubReviewResponseDTO> result;

        result = reviewService.getMyReviews(userId, page);

        return ResponseEntity.ok(result);
    }
// Service
      // 내가 쓴 리뷰 목록 조회
    @Transactional(readOnly = true)
    public Page<ReadingClubReviewResponseDTO> getMyReviews(Long userId, int page) {

        Pageable pageable = PageRequest.of(page, 15);

        Page<ReadingClubReview> reviews =
                reviewRepository.findByWriterId_OrderByCreatedAtDesc(userId, pageable);

        return reviews.map(ReadingClubReviewResponseDTO::from);

    }
```
### 댓글 생성
```java
  // 후기글에 댓글(대댓글) 생성
  // parentConmmetId가 null이라면 댓글, null이 아니라면 대댓글
  @PostMapping("/reviewId/{reviewId}")
    public ResponseEntity<ReviewCommentResponseDTO> createReviewComments(@PathVariable("reviewId") Long reviewId,
                                                                         @RequestBody ReviewCommentRequestDTO request,
                                                                         HttpServletRequest req) {
        // 로그인한 사용자의 username 가져오기
        String id = req.getHeader("X-User-ID");
        Long userId = Long.valueOf(id);

        ReviewCommentResponseDTO response = reviewCommentService.createReviewComment(reviewId, request,userId);

        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
// Service
    @Transactional
    public ReviewCommentResponseDTO createReviewComment(Long reviewId,
                                                        ReviewCommentRequestDTO request,
                                                        Long userId) {
        // 1. 리뷰 찾기
        ReadingClubReview review = reviewRepository.findById(reviewId)
                .orElseThrow(() -> new IllegalArgumentException("존재하지 않는 리뷰입니다."));

        // 2. 부모 댓글이 null인지 확인
        Long parentCommentId = request.getParentCommentId();
        ReviewComment parent = null;

        // 3. null이 아니라면 검증
        if (parentCommentId != null) {
            // 부모 댓글이 존재하는지
            parent = commentRepository.findById(parentCommentId)
                    .orElseThrow(() -> new IllegalArgumentException("부모 댓글이 존재하지 않습니다."));

            // 부모 댓글이 같은 리뷰에 속한 댓글인지
            if (!parent.getReview().getReviewId().equals(reviewId)) {
                throw new IllegalArgumentException("해당 리뷰의 댓글에만 대댓글을 작성할 수 있습니다.");
            }

            // 부모가 이미 대댓글이면 또 대댓글 불가
            if (parent.getParent() != null) {
                throw new IllegalStateException("대댓글에 또 대댓글을 작성할 수 없습니다.");
            }
        }
        // 4. 댓글 생성
        ReviewComment comment = ReviewComment.builder()
                .review(review)
                .user(userId)
                .parent(parent)
                .commentDetail(request.getCommentDetail())
                .build();

        ReviewComment saved = commentRepository.save(comment);

        return ReviewCommentResponseDTO.from(saved);
  }
```
### 댓글 수정
```java
    // 댓글 수정
    // 자신이 작성한 댓글만 수정 가능
    @PatchMapping("/commentId/{commentId}")
    public ResponseEntity<ReviewCommentResponseDTO> modifyComments(@PathVariable("commentId") Long commentId,
                                                                   @RequestBody ReviewCommentRequestDTO request,
                                                                   HttpServletRequest req) {
        // 로그인한 사용자의 username 가져오기
        String id = req.getHeader("X-User-ID");
        Long userId = Long.valueOf(id);

        ReviewCommentResponseDTO response = reviewCommentService.modifyComment(commentId, request, userId);

        return ResponseEntity.ok(response);
    }
// Service
    @Transactional
    public ReviewCommentResponseDTO modifyComment(Long commentId,
                                                  ReviewCommentRequestDTO request,
                                                  Long userId) {

        // 이 유저가 작성한 해당 댓글 찾기 (아니면 권한 없음)
        ReviewComment comment = commentRepository
                .findByReviewCommentIdAndUser(commentId, userId)
                .orElseThrow(() -> new AccessDeniedException("해당 댓글을 수정할 수 있는 권한이 없습니다."));

        // 내용 수정
        comment.updateContent(request.getCommentDetail());

        ReviewComment saved = commentRepository.save(comment);
        return ReviewCommentResponseDTO.from(saved);

    }
```
### 댓글 삭제
```java
    // 댓글 삭제 (soft delete)
    // 자신이 작성한 댓글만 삭제가 가능하고 '삭제된 댓글입니다'로 처리됨
    @DeleteMapping("/commentId/{commentId}")
    public ResponseEntity<Void> deleteComment(@PathVariable("commentId") Long commentId,
                                              HttpServletRequest req) {
        // 로그인한 사용자의 username 가져오기
        String id = req.getHeader("X-User-ID");
        Long userId = Long.valueOf(id);

        reviewCommentService.deleteComment(commentId, userId);

        return ResponseEntity.noContent().build();
    }
// Service
    @Transactional
    public void deleteComment(Long commentId, Long userId) {

        // 이 유저가 쓴 해당 댓글 찾기
        ReviewComment comment = commentRepository
                .findByReviewCommentIdAndUser(commentId, userId)
                .orElseThrow(() -> new AccessDeniedException("해당 댓글을 삭제할 수 있는 권한이 없습니다."));

        comment.softDelete();
    }
    // 소프트 삭제 메서드
    public void softDelete() {
        this.deleteComment = true;
        this.commentDetail = "삭제된 메시지입니다.";
    }
```
### 후기글 댓글 조회
```java
    // 후기글 댓글 조회
    // 최신순으로 정렬되며 대댓글은 부모 댓글 밑으로 정렬됨
    @GetMapping("/reviewId/{reviewId}")
    public ResponseEntity<List<ReviewCommentResponseDTO>> viewComment(@PathVariable("reviewId") Long reviewId,
                                                                      HttpServletRequest req) {
        // 로그인한 사용자의 username 가져오기
        String id = req.getHeader("X-User-ID");
        Long userId = Long.valueOf(id);;

        List<ReviewCommentResponseDTO> comments = reviewCommentService.viewComment(reviewId, userId);

        return ResponseEntity.ok(comments);
    }
// Service
    @Transactional
    public List<ReviewCommentResponseDTO> viewComment(Long reviewId, Long userId) {

        List<ReviewComment> comments =
                commentRepository.findByReview_ReviewIdOrderByCreatedAtDesc(reviewId);

        // 일반 댓글만 필터링
        List<ReviewComment> parents = comments.stream()
                .filter(c -> c.getParent() == null)
                .toList();

        List<ReviewCommentResponseDTO> result = new ArrayList<>();

        for (ReviewComment parent : parents) {
            // 부모 댓글 추가
            result.add(ReviewCommentResponseDTO.from(parent));

            // 대댓글 필터링
            List<ReviewComment> children = comments.stream()
                    .filter(c -> parent.getReviewCommentId().equals(
                            c.getParent() != null ? c.getParent().getReviewCommentId() : null))
                    .toList();

            // 대댓글 추가
            children.forEach(child -> result.add(ReviewCommentResponseDTO.from(child)));
        }

        return result;
    }
```
### 좋아요 기능
```java
    // 좋아요 추가/취소
    // 좋아요를 누르지 않은 상태였을 경우 좋아요 추가, 누른 상태였을 경우 취소됨
    @PostMapping("/reviewId/{reviewId}")
    public ResponseEntity<ReviewLikeToggleResponseDTO> toggleLike(@PathVariable Long reviewId,
                                                                  HttpServletRequest req) {
        String id = req.getHeader("X-User-ID");
        Long userId = Long.valueOf(id);

        ReviewLikeToggleResponseDTO response =
                reviewLikeService.toggleLike(reviewId, userId);
        return ResponseEntity.ok(response);

    }
// Service
    @Transactional
    public ReviewLikeToggleResponseDTO toggleLike(Long reviewId, Long userId) {
        // 존재하는 리뷰인지 검증
        ReadingClubReview review = reviewRepository.findById(reviewId)
                .orElseThrow(()->new IllegalArgumentException("존재하지 않는 리뷰입니다."));
        // 좋아요를 눌렀는지 검증
        boolean alreadyLiked = reviewLikeRepository.existsByReviewAndUserId(review, userId);

        boolean nowLiked;
        if (alreadyLiked) {
            // 좋아요 취소
            reviewLikeRepository.deleteByReviewAndUserId(review, userId);
            review.decreaseLike();
            nowLiked = false;
        } else {
            // 좋아요 추가
            ReviewLike like = ReviewLike.builder()
                    .review(review)
                    .user(userId)
                    .build();
            reviewLikeRepository.save(like);
            review.increaseLike();
            nowLiked = true;
        }
        long likeCount = reviewLikeRepository.countByReview(review);
        return new ReviewLikeToggleResponseDTO(nowLiked, likeCount);
    }
```
### 내 후기글에 좋아요 누른 사람 조회
```java
    // 내 후기글에 좋아요 누른 사람 nickname 조회
    @GetMapping("/{reviewId}/likes")
    public ResponseEntity<List<String>> getLikedUsers(
            @PathVariable Long reviewId,
            HttpServletRequest req) {

        String id = req.getHeader("X-User-ID");
        Long userId = Long.valueOf(id);

        List<String> likedUsernames =
                reviewLikeService.getLikedUsernames(reviewId, userId);

        return ResponseEntity.ok(likedUsernames);
    }
// Service
    @Transactional(readOnly = true)
    public List<String> getLikedUsernames(Long reviewId, Long loginUserId) {


        // 리뷰 조회
        ReadingClubReview review = reviewRepository.findById(reviewId)
                .orElseThrow(() -> new IllegalArgumentException("존재하지 않는 리뷰입니다."));

        // 이 리뷰가 내가 쓴 글인지 검증
        if (!review.getWriterId().equals(loginUserId)) {
            throw new AccessDeniedException("자신이 작성한 후기글의 좋아요만 조회할 수 있습니다.");
        }

        // 해당 리뷰에 달린 좋아요 전체 조회
        List<ReviewLike> likes = reviewLikeRepository.findByReview_ReviewId(reviewId);

        // userId 리스트로 변환 (중복 제거)
        List<Long> userIds = likes.stream()
                .map(ReviewLike::getUserId)
                .distinct()
                .toList();

        if (userIds.isEmpty()) {
            return List.of();   // 빈 리스트 반환
        }

        // 6) userId로 User 조회 → username 리스트 뽑기

        return userIds.stream()
                .map(userFeignClient::getUserProfileById) // user-service 호출
                .map(UserProfileResponse::getNickName)                     // nickname만 추출
                .toList();
    }
// Feign Client DTO
public class UserProfileResponse {
    private Long userId;
    private String nickName;
    private String username;
    private String role;

}
```
### Feign Client
```java
@FeignClient(name = "user-service", configuration = FeignClientConfig.class)
public interface UserFeignClient {

    @GetMapping("/user/userId/{userId}")
    UserProfileResponse getUserProfileById(@PathVariable("userId") Long userId);

    @GetMapping("/user/username/{username}")
    UserProfileResponse getUserProfileByUsername(@PathVariable("username") String username);
}
    // API
    @GetMapping("/username/{username}")
    public ResponseEntity<UserProfileResponse> getUserProfileByUsername(
            HttpServletRequest req,
            @PathVariable("username") String username) {
            UserProfileResponse profile
                    = userService.getProfileByUsername(username);
            return ResponseEntity.ok(profile);
    }

    @GetMapping("/userId/{userId}")
    public ResponseEntity<UserProfileResponse> getUserProfileById(
            HttpServletRequest req,
            @PathVariable("userId") Long userId) {

        UserProfileResponse profile=
                userService.getProfileById(userId);
        return ResponseEntity.ok(profile);
    }

// DTO
@Getter
@NoArgsConstructor
@AllArgsConstructor
public class UserProfileResponse {
    private Long userId;
    private String nickName;
    private String username;
    private String role;

    public static UserProfileResponse from(User user) {
        return new UserProfileResponse(
                user.getId(),
                user.getNickname(),
                user.getUsername(),
                user.getRole().name()
        );
    }
}
```

