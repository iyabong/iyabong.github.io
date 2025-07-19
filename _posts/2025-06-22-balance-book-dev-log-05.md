---
title: "Balance-Book 개발기 (5) - 루틴 점검 백엔드 API 개발"
date: 2025-06-22
categories: [Balance-Book]
tags: [루틴 점검, 백엔드, Routine, C#, Oracle, EF Core, API]
---

## 작업 개요

루틴 점검 기능을 위한 백엔드 API를 개발했습니다.  

주요 기능

- 기준 달력 테이블(`TBL_CALENDAR_DATES`)과 루틴 점검 테이블(`TBL_ROUTINE_CHECK`) 조인을 통해 날짜별 점검 상태 조회
- 사용자별 루틴 템플릿 대상으로 조회

---

## 주요 구현 내용

### 1. Controller 구현

- `/api/routine/calendar` 엔드포인트 생성
- `startDate`, `endDate`, `userId`를 쿼리스트링으로 받아서 점검 현황 응답
- 서비스 클래스에서 날짜 목록 + 체크 내역을 조합하여 반환

```csharp
[HttpGet("calendar")]
public async Task<IActionResult> GetRoutineCalendar([FromQuery] DateTime startDate, [FromQuery] DateTime endDate, [FromQuery] string userId)
{
    var result = await _routineService.GetRoutineCheckCalendarAsync(startDate, endDate, userId);
    return Ok(result);
}
```

### 2. Service 로직

- 사용자별 활성 템플릿 ID 목록 조회
- 해당 템플릿의 루틴 점검 내역 필터링
- `TBL_CALENDAR_DATE` 기준 날짜 목록과 JOIN하여 상태 병합

```csharp
public async Task<IEnumerable<RoutineCheckCalendarDto>> GetRoutineCheckCalendarAsync(DateTime startDate, DateTime endDate, String userId)
{
    var result = calendarDates
        .GroupJoin(
            routineChecks,
            cd => cd.CalDate.Date,
            rc => rc.CheckTime.Date,
            (cd, rcGroup) => new RoutineCheckCalendarDto(
                cd.CalDate,
                cd.DayOfWeek,
                rcGroup.Select(rc => new RoutineCheckDto(
                    rc.TemplateId,
                    rc.Status,
                    rc.CheckTime.Date
                )).ToList()
            )
        ).ToList();

    return result;
}
```


### 3. DTO 설계

```csharp
using System;
using System.Collections.Generic;
using BalanceBook.Models;

namespace BalanceBook.Dtos
{
    public record RoutineCheckDto(
        string TemplateId,
        string Status,
        DateTime CheckTime
    );

    public record RoutineCheckCalendarDto(
        DateTime Date,
        string DayOfWeek,
        IReadOnlyList<RoutineCheckDto> RoutineChecks
    );
} 
```

---

### API 요청&응답 예시
요청 URL: `http://localhost:5000/api/routine/calendar?startDate=20250608&endDate=20250610`

```JSON
[
  {
    "date": "2025-06-08T00:00:00",
    "dayOfWeek": "SUN",
    "routineChecks": [
      {
        "templateId": "T100",
        "status": "PROG",
        "checkTime": "2025-06-08T00:00:00"
      }
    ]
  },
  {
    "date": "2025-06-09T00:00:00",
    "dayOfWeek": "MON",
    "routineChecks": []
  },
  {
    "date": "2025-06-10T00:00:00",
    "dayOfWeek": "TUE",
    "routineChecks": []
  }
]
```
- 2025-06-08 체크결과 있는 경우


---

## 향후 계획

- 백엔드 배포 환경에 OCI DB 연동 설정
- 프론트 UI 연결

---
