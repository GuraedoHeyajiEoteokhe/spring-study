### report-service API

- 일주일마다 (계획한 칼로리 소모량 - 실제 칼로리 소모량)을 계산해 ‘운동을 했다면?’ 이라는 가정을 토대로 리포트를 생성
- 지난 주 리포트 생성 (이미 지난 주 리포트가 있다면 생성 불가능)
- 유저의 전체 리포트 조회
- 날짜를 입력해 해당하는 주의 리포트 조회
- 리포트 삭제

```
package com.team3.reportservice.controller;

import com.team3.reportservice.dto.response.ApiResponse;
import com.team3.reportservice.dto.response.ReportViewDTO;
import com.team3.reportservice.service.ReportService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.security.SecurityRequirement;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDate;
import java.util.List;

@RestController
@RequestMapping("/report")
@RequiredArgsConstructor
public class ReportController {

    private final ReportService reportService;

    @PostMapping("/last-week-report")
    @Operation(summary = "지난 주 리포트 생성 API입니다.")
    @SecurityRequirement(name = "JWT")
    public ResponseEntity<ApiResponse<String>> createLastWeekReport(
            @Parameter(hidden = true) @RequestHeader("X-User-Id") Long userId
    ) {
        try {
            reportService.createLastWeekReport(userId);
            return ResponseEntity.status(HttpStatus.CREATED)
                    .body(ApiResponse.success("지난 주 리포트를 생성했습니다."));
        } catch (IllegalStateException e) {
            return ResponseEntity.ok(
                    ApiResponse.success("이미 지난 주 리포트가 존재합니다.")
            );
        }
    }

    @GetMapping("/all-report")
    @Operation(summary = "사용자별 전체 리포트 조회 API입니다.")
    @SecurityRequirement(name = "JWT")
    public ResponseEntity<ApiResponse<List<ReportViewDTO>>> userViewReport(
            @Parameter(hidden = true) @RequestHeader("X-User-Id") Long userId
    ) {
        List<ReportViewDTO> reports = reportService.getReportsByUserId(userId);

        if (reports.isEmpty()) {
            return ResponseEntity.status(HttpStatus.NO_CONTENT)
                    .body(ApiResponse.success(null));
        }

        return ResponseEntity.status(HttpStatus.OK)
                .body(ApiResponse.success(reportService.getReportsByUserId(userId)));
    }

    @GetMapping("/search-report/{date}")
    @Operation(summary = "날짜 검색을 통한 리포트 조회 API입니다.")
    @SecurityRequirement(name = "JWT")
    public ResponseEntity<ApiResponse<ReportViewDTO>> searchViewReport(
            @Parameter(hidden = true) @RequestHeader("X-User-Id") Long userId,
            @PathVariable("date") LocalDate date
    ) {
        ReportViewDTO report = reportService.getReportByDate(userId, date);

        if (report == null) {
            return ResponseEntity.noContent().build();
        }

        return ResponseEntity.status(HttpStatus.OK)
                .body(ApiResponse.success(reportService.getReportByDate(userId, date)));
    }

    @DeleteMapping("/delete/{reportId}")
    @Operation(summary = "리포트 삭제 API입니다.")
    @SecurityRequirement(name = "JWT")
    public ResponseEntity<ApiResponse<String>> deleteReport(
            @Parameter(hidden = true)  @RequestHeader("X-User-Id") Long userId,
            @PathVariable("reportId") Long reportId
    ) {
        reportService.deleteReportById(userId, reportId);
        return ResponseEntity.status(HttpStatus.OK)
                .body(ApiResponse.success("리포트가 삭제되었습니다."));
    }
}

```

### stats-service API

- 일주일마다 회원별 칼로리 소모량, 운동 총 시간을 합산해 통계 생성
- 유저의 전체 통계 조회 가능
- 전체 유저 칼로리 소모량, 운동 총 시간 랭킹 조회 가능
- 날짜를 입력해 해당하는 주의 전체 유저 칼로리 소모량, 운동 총 시간 랭킹 조회 가능


```
package com.team3.statsservice.controller;

import com.team3.statsservice.dto.response.ApiResponse;
import com.team3.statsservice.dto.response.CalorieRankingDto;
import com.team3.statsservice.dto.response.StatsViewDTO;
import com.team3.statsservice.dto.response.TimeRankingDto;
import com.team3.statsservice.service.StatsService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.security.SecurityRequirement;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDate;
import java.util.List;

@RestController
@RequestMapping("/stats")
@RequiredArgsConstructor
public class StatsController {

    private final StatsService statsService;

    @PostMapping("/last-week-stats")
    @Operation(summary = "지난 주 통계 생성 API입니다.")
    @SecurityRequirement(name = "JWT")
    public ResponseEntity<ApiResponse<String>> createLastWeekStats(
            @Parameter(hidden = true) @RequestHeader("X-User-Id") Long userId
    ) {
        statsService.createLastWeekStats(userId);
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ApiResponse.success("지난 주 랭킹이 생성되었습니다."));
    }

    @GetMapping("/user-stats")
    @Operation(summary = "사용자별 통계 조회 API입니다.")
    @SecurityRequirement(name = "JWT")
    public ResponseEntity<ApiResponse<List<StatsViewDTO>>> viewStatsByUserId(
            @Parameter(hidden = true) @RequestHeader("X-User-Id") Long userId
    ) {
        List<StatsViewDTO> stats = statsService.getStatsByUserId(userId);

        if (stats.isEmpty()) {
            return ResponseEntity.status(HttpStatus.NO_CONTENT)
                    .body(ApiResponse.success(null));
        }

        return ResponseEntity.status(HttpStatus.OK)
                .body(ApiResponse.success(statsService.getStatsByUserId(userId)));
    }

    @GetMapping("/last-week-time-ranking")
    @Operation(summary = "지난 주 운동량 랭킹 조회 API입니다.")
    @SecurityRequirement(name = "JWT")
    public ResponseEntity<ApiResponse<List<TimeRankingDto>>> viewLastWeekTimeRanking() {
        List<TimeRankingDto> rankingList = statsService.getLastWeekTimeRanking();

        if (rankingList.isEmpty()) {
            return ResponseEntity.status(HttpStatus.NO_CONTENT)
                    .body(ApiResponse.success(null));
        }

        return ResponseEntity.status(HttpStatus.OK)
                .body(ApiResponse.success(statsService.getLastWeekTimeRanking()));
    }

    @GetMapping("/last-week-calorie-ranking")
    @Operation(summary = "지난 주 칼로리 소모량 랭킹 조회 API입니다.")
    @SecurityRequirement(name = "JWT")
    public ResponseEntity<ApiResponse<List<CalorieRankingDto>>> viewLastWeekCalorieRanking() {

        List<CalorieRankingDto> rankingList = statsService.getLastWeekCalorieRanking();

        if (rankingList.isEmpty()) {
            return ResponseEntity.status(HttpStatus.NO_CONTENT)
                    .body(ApiResponse.success(null));
        }

        return ResponseEntity.status(HttpStatus.OK)
                .body(ApiResponse.success(statsService.getLastWeekCalorieRanking()));
    }

    @GetMapping("/search-time-ranking/{date}")
    @Operation(summary = "날짜 검색을 통한 운동량 랭킹 조회 API입니다.")
    @SecurityRequirement(name = "JWT")
    public ResponseEntity<ApiResponse<List<TimeRankingDto>>> viewTimeRankingByDate(
            @PathVariable("date") LocalDate date
    ) {
        List<TimeRankingDto> rankingList = statsService.getTimeRankingByDate(date);

        if (rankingList.isEmpty()) {
            return ResponseEntity.status(HttpStatus.NO_CONTENT)
                    .body(ApiResponse.success(null));
        }

        return ResponseEntity.status(HttpStatus.OK)
                .body(ApiResponse.success(statsService.getTimeRankingByDate(date)));
    }

    @GetMapping("/search-calorie-ranking/{date}")
    @Operation(summary = "날짜 검색을 통한 칼로리 소모량 랭킹 조회 API입니다.")
    @SecurityRequirement(name = "JWT")
    public ResponseEntity<ApiResponse<List<CalorieRankingDto>>> viewCalorieRankingByDate(
            @PathVariable("date") LocalDate date
    ) {
        List<CalorieRankingDto> rankingList = statsService.getCalorieRankingByDate(date);

        if (rankingList.isEmpty()) {
            return ResponseEntity.status(HttpStatus.NO_CONTENT)
                    .body(ApiResponse.success(null));
        }

        return ResponseEntity.status(HttpStatus.OK)
                .body(ApiResponse.success(statsService.getCalorieRankingByDate(date)));
    }
}

```
