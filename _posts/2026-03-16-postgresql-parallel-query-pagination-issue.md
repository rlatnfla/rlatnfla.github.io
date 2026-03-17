---
layout: post
title: "정렬 기준이 Uniqueness 하지 않을 때, 병렬 쿼리가 일으키는 결과의 불일치"
date: 2026-03-16
categories: [JPA, DB]
---

### 문제 상황

리스트 페이지를 작업 하던 중 이상한 현상을 발견했다.   
동일한 페이지인데도 불구하고 새로고침을 할때마다 다른 결과가 나오는 것이었다. 

페이지네이션은 JPA의 `Pageable`을 사용하여 구현하고 있었고,  
당시 `Pageable`의 기본 정렬 값은 엔티티 가 생성된 시점인 `created_time`을 사용하고 있었다.  
```java
@PageableDefault(size = 50, sort = "createdTime", direction = Direction.ASC)
```

`creatd_time`을 기본 정렬 값으로 사용하고 있던 엔티티는 `Album`과 `Track`이었는데 `Album`에 대해서만 이러한 현상이 발생하는 것이 이상했다.

`Album`과 `Track`의 쿼리 플랜에서 원인을 찾을 수 있었다.

```text
# Album의 Query Plan (Before Optimization)
Gather Merge  (actual time=230.750..350.082 rows=528826 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  Buffers: shared hit=6970 read=12240, temp read=24660 written=24698
  ->  Sort  (actual time=151.302..182.424 rows=176275 loops=3)
        Sort Key: album.created_time DESC
        Sort Method: external merge  Disk: 58696kB
        Worker 0:  Sort Method: external merge  Disk: 38088kB
        Worker 1:  Sort Method: external merge  Disk: 41856kB
        ->  Parallel Seq Scan on public.album  (actual time=0.185..20.732 rows=176275 loops=3)
              Buffers: shared hit=6896 read=12240

Planning Time: 0.072 ms
Execution Time: 372.496 ms
```
```text
# Track의 Query Plan (Before Optimization)
Sort (actual time=11.537..13.137 rows=13898 loops=1)
  Sort Key: track.created_time DESC
  Sort Method: external merge  Disk: 4400kB
  Buffers: shared hit=640, temp read=550 written=551
  ->  Seq Scan on public.track (actual time=0.011..1.170 rows=13898 loops=1)
        Buffers: shared hit=640

Planning Time: 0.068 ms
Execution Time: 19.534 ms
```

주목해야할 곳은 `Album`의 `Gather Merge`와 `Workers Launched: 2` 이다.  
옵티마이저는 `Album`의 레코드 수가 많기 때문에 2개의 추가 워커를 사용하는 병렬 쿼리(Parallel Query)를 계획했다.  
이때 `created_time`이 동일한 여러 레코드가 서로 다른 워커에 의해 처리되는데, **병렬 처리 특성상 각 워커로부터 데이터가 도달하는 순서가 매번 달라질 수 있다.**  
결과적으로 정렬 기준 내에서 순위가 같은 데이터들의 최종 출력 순서가 비결정적(Non-deterministic)으로 결정되는 것이다.

반면 `Track`은 옵티마이저가 단일 워커(Single Process)로 처리하는 것이 효율적이라고 판단했다.  
혼자서 **순차적으로 데이터를 정렬하므로 물리적인 읽기 순서가 유지되어** 결과가 일관되게 보였던 것뿐이며, **데이터 증가로 인해 병렬 스캔이 도입된다면 동일한 문제가 발생할 잠재적 위험을 안고 있다.**

### 결론

`id`와 같은 유니크한 컬럼을 두 번째 정렬 조건으로 명시하여 위 현상을 막을 수 있다.

비일관적인 결과 말고도 성능상 병목이 발견 되었다. 두 쿼리 플랜에서 `Sort Method: external merge  Disk: ~`가 나타나는데,  
이는 정렬을 수행할 때, 메모리 내에서 해결 하지 못해 디스크의 임시 공간을 사용했음을 의미한다.

이 상황에서 `(created_time, id)` 복합 인덱스를 생성하면 아래와 같은 이점을 챙길 수 있다.
1. **일관성 보장**: 인덱스 리프 노드에 `id`까지 정렬되어 저장되므로, 병렬 워커가 작업해도 일관된 순서를 유지 할 수 있다.
2. **성능 최적화**: DB는 이미 정렬된 인덱스 경로를 따라 읽기만 하면 되므로 정렬 작업을 위해 디스크에 접근 하지 않아도 되므로 조회 성능을 단축 할 수 있다.

```text
# Album의 Query Plan (After Optimization)
Index Scan Backward using idx_album_created_time_id on public.album (actual time=0.055..134.977 rows=528826 loops=1)
  Buffers: shared hit=40464 read=17766
Execution Time: 147.944 ms                                                                                                                      |
```

```text
# Track의 Query Plan (After Optimization)
Index Scan Backward using idx_track_created_time_id on public.track (actual time=0.287..72.937 rows=13898 loops=1)
  Buffers: shared hit=2183 read=655
Execution Time: 73.423 ms
```

디스크 소트가 사라졌고 복합 인덱스로 정렬된 덕분에 결과의 비일관성이라는 문제가 해결되었다.




















