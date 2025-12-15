package com.ohgiraffers.promotion.core.api.controller.v1;

import com.ohgiraffers.common.support.response.ApiResult;
import com.ohgiraffers.promotion.core.api.controller.v1.request.OrderRequest;
import com.ohgiraffers.promotion.core.api.controller.v1.request.PromotionRequest;
import com.ohgiraffers.promotion.core.api.controller.v1.response.OrderResponse;
import com.ohgiraffers.promotion.core.api.controller.v1.response.PromotionListResponse;
import com.ohgiraffers.promotion.core.api.controller.v1.response.PromotionResponse;
import com.ohgiraffers.promotion.core.domain.Promotion;
import com.ohgiraffers.promotion.core.domain.PromotionService;
import com.ohgiraffers.promotion.core.enums.PromotionStatus;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import org.hibernate.query.Order;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.*;

import java.util.ArrayList;
import java.util.List;

@RestController
public class PromotionController {
    private PromotionService promotionService;


    @Autowired
    public PromotionController(PromotionService promotionService) {
        this.promotionService = promotionService;
    }

    // 새로운 프로모션 추가
    @PostMapping
    @Operation(summary = "프로모션 생성", description = "새로운 프로모션을 생성합니다..")
    public ApiResult<?> save(@AuthenticationPrincipal String userId,
            @RequestBody PromotionRequest promotionRequest) {
        promotionService.promotionSave(Long.parseLong(userId),promotionRequest);
        return ApiResult.success();
    }

    @PutMapping("/{id}")
    @Operation(summary = "프로모션 수정", description = " 대기상태인 프로모션을 수정합니다.. ")
    public ApiResult<?> update(
            @AuthenticationPrincipal String userId,
            @PathVariable long id,
            @RequestBody PromotionRequest promotionRequest) {
        promotionService.promotionUpdateById(Long.parseLong(userId),id, promotionRequest);
        return ApiResult.success();
    }

    @DeleteMapping("/{id}")
    @Operation(summary = "프로모션 삭제", description = "프로모션을 삭제합니다...")
    public ApiResult<?> deleteById(@PathVariable Long id) {
        promotionService.deletePromotion(id);
        return ApiResult.success();
    }

    @GetMapping
    @Operation(summary = "프로모션 조회", description = "전체 프로모션을 조회합니다..")
    public ApiResult<List<PromotionResponse>> getAllPromotion() {
        return ApiResult.success(promotionService.findAll());
    }

    @Operation(summary = "상태별 프로모션 조회", description = "상태별 프로모션 조회..")
    @GetMapping("/status/{promotionStatus}")
    public ApiResult<PromotionListResponse> getPromotionsStatusAll(@PathVariable String promotionStatus) {
        return ApiResult.success(promotionService.getPromotionsByStatus(promotionStatus));
    }

    @Operation(summary = "프로모션 검색", description = "입력하신 ID로 프로모션을 검색합니다..")
    @GetMapping("/{id}")
    public ApiResult<PromotionResponse> findPromotionById(@PathVariable Long id) {
        return ApiResult.success(promotionService.findPromotionById(id));
    }

    @Operation(summary = "feign 필요 파일 조회")
    @PostMapping("/order")
    public ApiResult<Integer> checkTotalQuantity(@RequestBody OrderRequest orderRequest) {
        return ApiResult.success(promotionService.updateSoldQuantity(orderRequest));
    }
}
