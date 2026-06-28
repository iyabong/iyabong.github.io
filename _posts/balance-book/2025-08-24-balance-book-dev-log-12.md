---
categories:
- Balance-Book
date: 2025-08-24
tags:
- Blazor
- Resume
- InMemory
- MVP
- Vercel
title: Balance-Book 개발기 (12) - [Blazor WASM] InMemory로 프로젝트 이력 MVP 구현 🚀
---

Blazor WASM으로 이력 관리 기능 MVP(최소기능제품) 구현하여
Vercel Dev Preview에 배포.

---

## 1️⃣ InMemory Service 구현

`IProjectService` 인터페이스를 만들고, `InMemoryProjectService`로 임시 데이터 제공.  
DB 연동 전까지는 이 방식으로 기본 CRUD 시뮬레이션.

```csharp
public class InMemoryProjectService : IProjectService
{
    private readonly List<Project> _items = new()
    {
        new Project {
            Role = "개발자",
            Responsibilities = "업무 모듈 개발 및 유지보수",
            Client = "A은행",
            Company = "BalanceBook Inc."
        }
    };

    public Task<List<Project>> GetAllAsync() => Task.FromResult(_items);

    public Task<Project> CreateAsync(Project item)
    {
        _items.Add(item);
        return Task.FromResult(item);
    }
}
```

---

## 2️⃣ MVP 동작 화면 🖼️

- 초기 프로젝트 1건이 리스트에 표시됨  
- 신규 프로젝트 추가 버튼 → InMemory에 저장됨  
- 즉시 화면에 반영되어 리스트 갱신 확인 ✅

> UI는 Blazor 기본 폼과 테이블을 활용

---

## 3️⃣ DEV Vercel 배포 확인 🔗

- Dev 환경 배포 링크: [https://dev-balance-book.vercel.app](https://dev-balance-book.vercel.app) 
  - 메뉴: 이력 관리 > Projects