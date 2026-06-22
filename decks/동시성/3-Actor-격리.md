# [동시성] 3. Actor 격리

> 최근 학습: 2026-06-22 · 누적 내 답안: 7개

## Q1. actor 키워드는 어떤 문제를 언어 차원에서 푸는가?

**정답**

<p>여러 Task가 같은 mutable 상태를 동시에 건드리는 <b>데이터 레이스</b>. GCD에선 serial queue로 <code>queue.sync/async</code> 감싸 직렬화했는데, 까먹으면 크래시.</p><pre><code>actor UserCache {
    private var cache: [String: User] = [:]
    func get(_ id: String) -&gt; User? { cache[id] }
}
// 호출: let u = await cache.get("123")</code></pre><p>class→actor + 호출 시 await. 큐 불필요, 데이터 레이스는 컴파일러가 막는다.</p>

**내 답안 이력**

- `2026-06-22 02:36` Task 비동기 처리가 많아질수록 데이터에 대한 race condition이 발생할 가능성이 커짐. 이때 어떤 데이터 인스턴스에 여러 task가 접근하더라도 격리를 통해 race condition이 일어나지 않게 시스템적으로 강제하도록 나온게 actor임.

---

## Q2. 모든 actor는 시리얼한가? 일반 actor와 @MainActor의 차이는?

**정답**

<p><b>모든 actor는 시리얼</b>이다 — 한 인스턴스 내부에서 메서드 동시 실행 없음. 이게 액터의 가장 본질적 성질.</p><ul><li>일반 actor: 시리얼하되 cooperative pool의 (그때그때 다른) 스레드에서 돈다</li><li>@MainActor: 시리얼하되 <b>반드시 메인 스레드</b>에서 돈다 (UIKit/SwiftUI용)</li></ul><p>메인 액터의 특수성은 "메인 스레드 바인딩"이지 시리얼성이 아니다.</p>

**내 답안 이력**

- `2026-06-22 02:38` 모든 actor는 시리얼함. 일반 actor는 cooperative thread pool의 임의의 쓰레드에서 동작, MainActor는 메인 쓰레드에서 동작.

---

## Q3. Actor Reentrancy란 무엇인가?

**정답**

<p>actor가 시리얼이라도 <b>await을 만나면 그 잠금이 일시적으로 풀린다.</b> 그 사이 다른 Task가 같은 actor의 메서드에 들어올 수 있다.</p><pre><code>func refreshToken() async -&gt; String {
    let old = token
    let new = await fetchFromServer() // 여기서 잠금 풀림
    token = new
    return new
}</code></pre><p>그래서 동일 갱신이 2번 수행되는 등 문제가 생긴다. 핵심: actor는 "한 번에 하나 실행"은 보장하지만 "중간에 인터럽트 없음"은 보장하지 않는다.</p>

**내 답안 이력**

- `2026-06-22 02:40` Actor 내부에 await가 있고 여러 Task가 동시 접근시 actor의 격리가 해제 되어 작업이 완료되기 전에 다른 작업 요청이 도달하게 될수 있음을 의미.

---

## Q4. await에서 actor 잠금을 일부러 풀어주는 이유 3가지는?

**정답**

<ol><li><b>UI 얼어붙음 방지</b>(특히 @MainActor) — await 동안 잠금을 잡고 있으면 스크롤·버튼 등 다른 이벤트 처리 불가</li><li><b>스레드 starvation 방지</b> — 풀에 스레드가 몇 개 없는데 await 중에도 잡고 있으면 풀이 고갈</li><li><b>데드락 방지</b> — A가 B를 await, B가 다시 A를 await해도 잠금이 풀려 있어 서로 기다리는 교착이 안 생김</li></ol>

**내 답안 이력**

- `2026-06-22 02:51` 1.쓰레드 풀의 쓰레드 갯수는 한정적이기 때문에 하나의 쓰레드가 오래 묶여 있는걸 방지하기 위함. 2.? 3.?

---

## Q5. Actor Reentrancy를 해결하는 3가지 패턴은?

**정답**

<ul><li><b>1. In-flight Task 저장</b> — 진행 중인 <code>Task</code>를 프로퍼티에 보관, 이미 있으면 그 결과를 await(중복 요청 방지)</li><li><b>2. State machine 가드</b> — <code>enum State</code>로 idle/inProgress 체크, 이미 진행 중이면 throw/무시</li><li><b>3. Stale result 폐기</b> — 요청마다 <code>UUID</code> 부여, await 후 "내가 여전히 최신인지" 비교해 아니면 결과 폐기(검색 자동완성)</li></ul><p>원칙: await 전후로는 상태가 바뀌어 있다고 가정하고 다시 체크.</p>

**내 답안 이력**

- `2026-06-22 02:52` 1.최신 task만 접근하도록 처리 2.상태 분기처리 3.?

---

## Q6. actor는 결국 하나의 데이터 타입인가? class와의 차이는?

**정답**

<p>그렇다. actor는 reference type이고 class와 거의 같은 위치다. "class + 자동 시리얼 큐"로 외우면 직관적. 추가로:</p><ol><li>자동 스레드 안전(매 메서드 호출 직렬화)</li><li>격리 영역 형성(외부 접근은 await)</li><li>상속 불가(final처럼 동작, 프로토콜 채택은 가능)</li><li>deinit은 nonisolated(격리 밖에서 실행)</li></ol>

**내 답안 이력**

- `2026-06-22 02:52` class와 같은 reference type의 데이터 타입임. 상속 불가능. 언어적으로 여러 task가 시리얼하게 접근하도록 강제.

---

## Q7. 격리 영역(Isolation Domain)이란? "다른 격리 영역" 접근은 어떻게 하나?

**정답**

<p>어떤 mutable state가 하나의 단위로 묶여 외부와 차단된 영역. 그 안의 데이터는 영역 안 코드에서만 동기 접근 가능, <b>다른 영역에서 접근하려면 await 필수</b>.</p><p>"다른 영역"의 예: 다른 actor 타입, 같은 타입의 다른 인스턴스, nonisolated→isolated, 메인 액터→일반 액터, 글로벌 함수→actor. <b>같은 actor 인스턴스 내부 isolated 코드끼리가 아니면 거의 다 다른 영역</b>이다.</p>

**내 답안 이력**

- `2026-06-22 02:53` 격리 영역 내부에선 시리얼하게 접근 되도록 강제되는 영역을 말함. 다른 격리 영역에서 접근은 await를 통해 접근해야함.

---

## Q8. nonisolated 키워드는 언제 쓰고, 무슨 함정이 있나?

**정답**

<p>actor 안에 있어도 "격리 안 받겠다"는 선언. 쓰는 경우: ①불변(let) 데이터만 접근하는 메서드 ②동기 요구 프로토콜 채택(Identifiable의 id) ③Hashable/Equatable.</p><p>함정: nonisolated 멤버에서 isolated mutable 멤버에 <b>동기 접근 불가</b>. 접근하려면 async + await 필요.</p><pre><code>nonisolated func bad() { data["x"] }       // 컴파일 에러
nonisolated func good() async { await data["x"] }</code></pre>

**내 답안 이력**

_(아직 작성한 답안 없음)_

---

## Q9. 커스텀 글로벌 액터(@globalActor)는 언제 유용한가?

**정답**

<p>여러 클래스 인스턴스를 <b>같은 전역 시리얼 영역</b>에 묶고 싶을 때.</p><pre><code>@globalActor
actor DatabaseActor { static let shared = DatabaseActor() }

@DatabaseActor class UserDatabase { ... }
@DatabaseActor class PostDatabase { ... }</code></pre><p>유용한 케이스: SQLite처럼 동시 접근 불가 리소스, 글로벌 캐시 시스템, 렌더링/시뮬레이션 엔진. 대부분은 인스턴스 단위 actor로 충분하다.</p>

**내 답안 이력**

_(아직 작성한 답안 없음)_

---
