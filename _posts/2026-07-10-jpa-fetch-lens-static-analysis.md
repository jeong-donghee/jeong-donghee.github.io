---
title: "IntelliJ에서 JPA N+1 방지에 도움을 주는 플러그인 추천"
description: IntelliJ에서 Spring Data JPA Repository 메서드의 JPA N+1 위험을 보여주는 플러그인.<br>연관 관계 매핑과 쿼리를 읽어 엔티티들의 실제 로딩 상태를 정적으로 계산하는 방법.
categories: [도구, 설계]
tags: [jpa, hibernate, spring-data-jpa, psi, intellij, jpa-fetch-lens]
---

<a href="https://plugins.jetbrains.com/plugin/32767-jpa-fetch-lens" target="_blank" rel="noopener noreferrer" style="display:flex; align-items:center; gap:16px; border:1px solid rgba(128,128,128,.28); border-radius:12px; padding:14px 16px; text-decoration:none; color:inherit; margin:1.2rem 0; box-shadow:0 1px 3px rgba(0,0,0,.07);"><span style="flex:0 0 auto; width:56px; height:56px; border-radius:14px; background:url('/assets/img/jpa-fetch-lens-icon.png') center/cover no-repeat;"></span><span style="display:flex; flex-direction:column; min-width:0;"><strong style="font-size:1.05rem; line-height:1.3;">JPA Fetch Lens</strong><span style="opacity:.75; font-size:.9rem; margin:3px 0 6px;">Hover over a Spring Data JPA repository method to see which associated entities that query actually loads.</span><span style="font-size:.78rem; opacity:.55; letter-spacing:.02em;">plugins.jetbrains.com · JetBrains Marketplace ↗</span></span></a>

## 플러그인 소개

Entity간 연관 관계 매핑과 쿼리를 읽어 엔티티들의 실제 로딩 상태 결과를 색으로 보여줍니다.<br>
N+1 발생을 미리 알아채는 데 도움이 됩니다.<br>
Spring Data JPA Repository 메서드에 hover하면 기존 메서드 설명에 더해 JPA Fetch Lens 트리가 뜹니다.

![repository 메서드에 hover하면 뜨는 fetch 트리](https://cdn.jsdelivr.net/gh/jeong-donghee/jpa-fetch-lens@main/docs/shot-full.png)
_repository 메서드에 hover하면 실제로 이렇게 뜬다._

- <span style="background:#43A047;color:#fff;padding:1px 8px;border-radius:4px;font-weight:600;">초록</span> = 이 쿼리가 당김(`join fetch` / `@EntityGraph`)
- <span style="background:#FDD835;color:#000;padding:1px 8px;border-radius:4px;font-weight:600;">노랑</span> = 매핑이 EAGER라 쿼리와 무관하게 항상 로딩
- <span style="background:#E53935;color:#fff;padding:1px 8px;border-radius:4px;font-weight:600;">빨강</span> = LAZY — 프록시, 건드리면 추가 쿼리(N+1 위험)

## 누구에게 추천하나요?

많은 수의 엔티티와 다양한 엔티티 연관 관계를 다루는 분들에게 추천합니다.

실무에서 여러 사람들이 만들어놓은 Spring Data JPA Repository 메서드를 쓰다보니 종종 엔티티 연관 관계를 놓치고 사용하여 N+1 문제가 발생하곤 했습니다.<br>
보다 손쉽게 메서드의 결과에 따른 엔티티 로딩 상태를 볼 수 있으면 좋겠다는 생각이 들어 이 플러그인을 만들었습니다.

## 그동안의 문제

```java
@Entity
class Order {
    @ManyToOne
    private Customer customer;         // 기본 EAGER

    @OneToMany(mappedBy = "order")
    private List<OrderItem> items;     // 기본 LAZY
}

interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findAll();             // customer (🟡EAGER), items (🔴LAZY)

    @Query("select o from Order o join fetch o.items")
    List<Order> findAllWithItems();    // customer (🟡EAGER), items (🟢FETCH)
}
```

연관 관계 매핑은 Entity에 있고, 실제 조회 쿼리는 JpaRepository에 있습니다.<br>
**"이 메서드를 부르면 무엇이 로딩되는가"는 이 둘을 머릿속에서 합성**해야 알 수 있습니다.<br>
이것을 잘못하면 N+1이 발생하거나 필요없는 로딩까지 발생할 수가 있습니다.

## 무엇을 만들었나

이 문제를 해결하기 위해 코드에 있는 연관 관계 매핑과 쿼리를 활용하기로 하였습니다.<br>
둘 다 코드에 적혀 있으니 실행 없이 IntelliJ IDE에서 코드만 읽어 정적으로 계산할 수 있었습니다.<br>
이때 IntelliJ가 제공하는, 코드를 타입·상속 관계까지 붙은 구조로 읽게 해주는 API인 PSI(Program Structure Interface)를 활용하여 매핑·쿼리를 읽었습니다.

### 핵심 아이디어: 연관 관계 매핑 + 쿼리 @Query(join fetch), @EntityGraph

"이 관계들이 로딩되나?"는 한 곳에 안 적혀 있으므로 아래 세 입력을 합쳤습니다.

1. **매핑 기본값** — JPA 규칙상 to-one(`@ManyToOne`/`@OneToOne`)은 기본 EAGER, to-many(`@OneToMany`/`@ManyToMany`)는 기본 LAZY. `fetch=`로 덮어쓸 수 있다.
2. **쿼리 오버라이드** — `@Query(join fetch)`나 `@EntityGraph`로 명시적으로 당기면 LAZY 매핑이어도 이 쿼리에선 로딩된다.
3. **프레임워크 특이케이스** — 선언과 실제가 어긋나는 함정(`@OneToOne`).

이제 중요한 것은 **"이 계산에 필요한 값을 소스에서 어떻게 읽어오나"** 였습니다.

### 1단계 — Repository에서 뿌리 엔티티 알아내기

`interface OrderRepository extends JpaRepository<Order, Long>`에서 뿌리 엔티티가 `Order`임을 알아야 합니다.<br>
그런데 `Order`라는 이름은 `JpaRepository`의 **제네릭 인자**로만 적혀 있고, 그 사이엔 `JpaRepository → PagingAndSortingRepository → CrudRepository → Repository` 상속 체인이 있었습니다.

PSI가 제공해주는 타입시스템의 **치환기(substitutor)** 를 호출하여 이를 해결했습니다.

```java
// Repository<T, ID> 마커를 기준으로, 이 repository에서 T가 무엇으로 치환되는지 계산
PsiSubstitutor s = TypeConversionUtil.getClassSubstitutor(marker, repositoryInterface, EMPTY);
PsiType domain = s.substitute(marker.getTypeParameters()[0]);   // T → Order
```

### 2단계 — 엔티티의 연관 관계와 로딩 결정 속성 읽기

PSI로 뿌리 엔티티의 필드를 훑어 연관 애노테이션과 **로딩을 결정하는 일부 속성** 을 확인했습니다.

```java
// "이 필드에 @OneToMany 붙어 있나?" (jakarta / javax)
PsiAnnotation ann = field.getAnnotation("...OneToMany");
// "그 애노테이션의 mappedBy 값은?" — '노드'로 반환
PsiAnnotationMemberValue mappedByNode = ann.findAttributeValue("mappedBy");
// 값 노드에서 실제 문자열 리터럴만 추출
String mappedBy = literalString(mappedByNode); // "order"
```

- 어떤 애노테이션인지, 기본 fetch → `@ManyToOne(EAGER)`, `@OneToMany(LAZY)`, `@OneToOne(EAGER)`, `@ManyToMany(LAZY)`
- `fetch` → LAZY/EAGER 명시 오버라이드
- `mappedBy` → 이 연관 관계의 소유권 확인 (뒤의 `@OneToOne` 함정과 역참조(A->B & B->A 관계) 판정에 쓰인다)

대상 엔티티 타입은 to-many면 컬렉션 제네릭(`List<OrderItem>` → `OrderItem`), to-one이면 필드 타입에서 얻었습니다.

### 3단계 — 쿼리가 무엇을 당기는지 읽기 (+별칭 해석)

`@EntityGraph(attributePaths)`는 경로 문자열을 읽어 마지막 경로까지 펼쳤습니다.<br>
(`@EntityGraph(attributePaths = "items.product")` → `items`, `items.product`로 펼침)

`@Query`의 JPQL은 정규식 두 개로 훑었습니다. (`nativeQuery=true`는 처리하지 않았습니다.)

> **JPQL이란?** 테이블이 아니라 **엔티티**를 대상으로 쓰는 JPA의 쿼리 언어.<br>
> `select o from Order o join fetch o.items`처럼 클래스·필드명으로 쓴다.
{: .prompt-tip }

**별칭 해석**은 다음과 같은 방법으로 이루어졌습니다.

```java
@Query("select o from Order o join fetch o.items i join fetch i.product")
```

`i.product`가 뿌리 기준으로 `items.product`임을 알려면, `i`가 `items`를 가리킨다는 걸 기억해야 합니다.<br>
그래서 `Map<별칭, 뿌리기준경로>` 심볼 테이블을 만들었습니다.<br>
`from Order o`에서 `o → ""`로 시작해, join fetch를 **왼→오로 한 번 훑으며** 새 별칭이 생길 때마다 등록했습니다.

> **왜 파서가 아니고 정규식으로 했나?** 정식 JPQL 파서는 정확하지만, IntelliJ의 파서(`com.intellij.jpa`)는 **Ultimate 전용**이라 붙이면 Community에서 못 돌아간다.
> 이 도구가 필요한 정보는 "어떤 연관 경로가 join fetch됐나"뿐이라, 그 부분만 정규식으로 집었다.
> 대신 `where`·`order` 같은 **키워드를 별칭으로 오인하지 않게** 거르는 방어가 필요했다.
> (트레이드오프: 서브쿼리·복잡 구문은 놓친다.)
{: .prompt-info }

### 4단계 — 연관 관계 매핑과 쿼리를 합쳐 색을 칠하고, 함정을 잡기

이제 각 연관마다 실제 로딩 상태를 계산해 색을 칠했습니다.<br>
이때 `@OneToOne` LAZY 함정을 잡았습니다.

```java
// 비소유(mappedBy 있음) @OneToOne + 선언 LAZY = 실제로는 EAGER 로딩
boolean lazyButEager = kind == ONE_TO_ONE && mappedBy != null && explicitLazy;
```

> `@OneToOne(mappedBy=..., fetch=LAZY)`로 **선언**해도, 비소유 쪽은
> Hibernate가 프록시로 만들 수 없어 **EAGER로 로딩**한다. "LAZY라 적었으니 안 불러오겠지"가 틀린다.
{: .prompt-info }

이를 잡아 노랑(EAGER)으로 찍고 `LAZY ignored → loads EAGER` 경고를 붙였습니다.

### 5단계 — 로딩되는 것만 펼쳐 그래프 순회

뿌리에서 DFS로 내려갔습니다. 
> **핵심 규칙: 로딩되는 엣지(FETCH/EAGER)만 대상 엔티티의 연관으로 재귀 확장하고, LAZY는 잎으로 둔다.** 
{: .prompt-info }

DFS로 내려가면서 현재 경로에서 이미 펼친 엔티티의 이름(QName)을 집합에 담아두었고, 어떤 연관의 대상 엔티티가 이미 그 집합에 있으면 재귀 확장을 멈췄습니다.<br>
이를 통해 `Order(EAGER)→Customer(EAGER)→Order…` 처럼 양방향 EAGER가 서로 물고 있으면 무한히 도는 현상을 막을 수 있습니다.<br>
또한 최대 재귀 깊이를 두어 과도하게 깊은 탐색을 방지했습니다.

## 한계

**보이는 것(정적)**
- 매핑 fetch.
- `@Query(join fetch)`/`@EntityGraph`가 당기는 것.
- `@OneToOne` LAZY→EAGER 함정.

**못 보는 것**
- **영속성 컨텍스트** — 트랜잭션 내에서 이미 로딩된 엔티티는 연관 관계 매핑상 LAZY라서 플러그인 표현상 LAZY로 보인다.
- **`@BatchSize` / `default_batch_fetch_size`** — LAZY 로딩을 IN 쿼리로 묶는다.
- **복잡한 JPQL·네이티브 쿼리(nativeQuery=true)** — 정규식의 한계, SQL은 지원하지 않는다.

## 마무리

이 플러그인을 만들며 결국 배운 건 JPA 문법이 아니라, **매핑을 안다고 fetch를 아는 게 아니라는 것**이었습니다.<br>
실제 로딩 상태는 매핑 + 쿼리 + 런타임의 합성이고, 그중 앞의 둘만 정적으로 앞당길 수 있었습니다.

안 보이던 걸 보이게 만들고 나서야, 내가 무엇을 못 보고 있었는지 알게 되었습니다.