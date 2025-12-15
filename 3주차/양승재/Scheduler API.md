package com.ohgiraffers.promotion.core.api.controller.schedule;

import com.ohgiraffers.common.constants.TimedealKeys;
import com.ohgiraffers.promotion.core.api.controller.v1.response.PromotionListResponse;
import com.ohgiraffers.promotion.core.api.controller.v1.response.PromotionResponse;
import com.ohgiraffers.promotion.core.api.controller.v1.response.RedisPromotionResponse;
import static com.ohgiraffers.promotion.core.enums.PromotionStatus.*;
import com.ohgiraffers.promotion.core.domain.Promotion;
import com.ohgiraffers.promotion.core.domain.PromotionService;
import com.ohgiraffers.promotion.storage.PromotionRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.util.List;


@Component
@RequiredArgsConstructor
public class PromotionScheduler {
    private final StringRedisTemplate stringRedisTemplate;
    private final PromotionService promotionService;
    private final PromotionRepository promotionRepository;

    @Scheduled(fixedRate = 5000)
    public void updatePromotionSchedule() {
        LocalDateTime now = LocalDateTime.now();

        List<Promotion> toActivate = promotionRepository.findByPromotionStatusAndStartTimeBefore(SCHEDULER, now);
        for (Promotion promotion : toActivate) {
            promotionService.updatePromotionStatus(promotion.getId(), ACTIVE);

            String key = TimedealKeys.setPromotion(promotion.getId());
            stringRedisTemplate.opsForValue().set(key, promotion.getTotalQuantity().toString());
        }

        List<Promotion> toEnd = promotionRepository.findByPromotionStatusAndEndTimeBefore(ACTIVE, now);
        for (Promotion promotion : toEnd) {
            promotionService.updatePromotionStatus(promotion.getId(), ENDED);

            String key = TimedealKeys.setPromotion(promotion.getId());
            stringRedisTemplate.delete(key);
        }
    }

    @Scheduled(fixedRate = 500)
    public void checkQuantity(){
        List<Promotion> checkSchedule = promotionRepository.findByPromotionStatus(ACTIVE);
        for(Promotion promotion : checkSchedule){
            String key = TimedealKeys.setPromotion(promotion.getId());
            String redisQuantity = stringRedisTemplate.opsForValue().get(key);
            if(Integer.parseInt(redisQuantity) != promotion.getTotalQuantity()-promotion.getSoldQuantity()){
                redisQuantity = String.valueOf(promotion.getTotalQuantity() - promotion.getSoldQuantity());
                stringRedisTemplate.opsForValue().set(key, redisQuantity);
            }
        }
    }
}
