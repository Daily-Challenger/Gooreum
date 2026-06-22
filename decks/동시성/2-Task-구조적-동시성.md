# [동시성] 2. Task & 구조적 동시성

> 최근 학습: 2026-06-22 · 누적 내 답안: 10개

## Q1. 동기 함수(viewDidLoad, IBAction)에서 async 함수를 호출하려면?

**정답**

<p><code>Task { }</code>로 새 비동기 컨텍스트를 만든다. Task 클로저 안은 자동으로 async 컨텍스트라 await을 쓸 수 있다.</p><pre><code>override func viewDidLoad() {
    super.viewDidLoad()
    Task {
        let user = await fetchUser()
        titleLabel.text = user.name
    }
}</code></pre><p>Task는 동기 세계에서 비동기 세계로 들어가는 "문"이다.</p>

**내 답안 이력**

- `2026-06-22 00:58` 동기함수 내에서 async 함수를 호출하기 위해선 Task 함수를 호출해야함.

---

## Q2. let task = Task { ... } 처럼 변수에 담으면 아직 실행 안 된 상태인가?

**정답**

<p>아니다. <b>Task는 생성 즉시 실행을 시작</b>한다. 변수에 담든 안 담든 상관없다.</p><pre><code>let task = Task { print("Hello") }
// 이 줄을 지나는 순간 이미 실행 중</code></pre><p>일반 클로저(<code>let f = { } ; f()</code>)와 다르다. 변수는 실행 트리거가 아니라 <b>핸들</b>이다 — <code>task.cancel()</code>, <code>await task.value</code>, <code>task.isCancelled</code>에 쓴다.</p>

**내 답안 이력**

- `2026-06-22 00:59` 아니다. Task { } 클로저가 호출되면 즉시 실행된다.

---

## Q3. Task { }와 GCD DispatchQueue.global().async의 본질적 차이는?

**정답**

<p>GCD async는 "백그라운드에서 이 클로저 돌려줘"가 전부 — 던지면 끝, 추적 불가, 취소는 직접 플래그.</p><p>Task는 작업이 <b>일등 시민</b>으로 대접받는다: 취소 가능, 우선순위 부여, 결과 받기, 부모-자식 관계 형성. GCD에서 매번 직접 만들던 것이 빌트인.</p>

**내 답안 이력**

_(아직 작성한 답안 없음)_

---

## Q4. 메인 액터에서 Task { }를 만들면 A/B/C/D 출력 순서가 보장되는 이유는? (A=동기, Task 안 B→await→C, D=동기)

**정답**

<p>결과는 <b>A → D → B → (대기) → C</b>. 메인 액터는 시리얼 큐라 한 번에 하나만 실행한다.</p><ul><li>viewDidLoad가 메인 액터 점유 중 → Task 클로저는 대기</li><li>D까지 찍고 viewDidLoad가 메인 액터 해제 → 그제서야 Task(B) 실행</li></ul><p><code>Task {}</code> in main ≈ <code>DispatchQueue.main.async</code>, <code>Task.detached {}</code> ≈ <code>DispatchQueue.global().async</code>(순서 보장 X).</p>

**내 답안 이력**

- `2026-06-22 01:01` 메인 액터 내부 큐는 기본적으로 시리얼 큐 이기 때문에 순서를 보장함.

---

## Q5. Task와 Task.detached의 차이는? 언제 detached를 쓰나?

**정답**

<p><code>Task { }</code>는 부모 컨텍스트의 actor isolation과 priority를 <b>상속</b>한다. <code>Task.detached { }</code>는 <b>상속하지 않고</b> 독립 실행한다.</p><p>실무에선 99% <code>Task { }</code>로 충분. detached는 부모 액터/우선순위를 의도적으로 끊어야 하는 특수한 경우만(예: CPU 작업 격리).</p>

**내 답안 이력**

- `2026-06-22 01:02` 기본적으로 Task는 부모 액터를 상속받음. 부모 액터를 상속 받고 싶지 않을 경우 Task.detached를 사용. 부모 액터를 상속 받는 다는 것은 부모 액터의 쓰레드와 priority를 상속받는 것을 의미함.

---

## Q6. async let의 역할과 전체 소요 시간 계산은?

**정답**

<p>독립적인 async 작업들을 <b>병렬</b>로 돌린다. <code>async let</code>은 await 없이 즉시 백그라운드 자식 Task로 시작하고, 결과는 나중에 한 번에 받는다.</p><pre><code>async let user = fetchUser()   // 2초
async let posts = fetchPosts() // 3초
let (u, p) = await (user, posts)</code></pre><p><b>전체 시간 = max(작업 시간들) = 3초</b> (순차였다면 5초). 작업 개수는 컴파일 타임에 고정(정적).</p>

**내 답안 이력**

- `2026-06-22 01:03` 정적 병렬 실행을 위해 사용. async let에 걸려 있는 함수 중 가장 실행 시간이 긴 시간 만큼 소요됨.

---

## Q7. async let으로 안 되고 TaskGroup이 필요한 경우는?

**정답**

<p>작업 개수가 <b>런타임에 결정</b>되는 동적 병렬(예: N개 이미지 업로드). async let은 컴파일 타임 고정이라 N번 못 적는다.</p><pre><code>await withTaskGroup(of: URL.self) { group in
    for image in images {
        group.addTask { await upload(image) }
    }
    var urls: [URL] = []
    for await url in group { urls.append(url) }
    return urls
}</code></pre><p>결과는 <b>완료 순서대로</b> 나온다(시작 순서 아님).</p>

**내 답안 이력**

- `2026-06-22 01:04` 정적 병렬이 아니라 런타임에 동적으로 비동기 작업을 처리할때 필요하다.

---

## Q8. async let과 TaskGroup의 차이를 요약하면?

**정답**

<ul><li><b>async let</b>: 정적(컴파일 타임 고정 N개), 결과는 <code>await (a,b,c)</code> 튜플, 코드 순서 고정</li><li><b>TaskGroup</b>: 동적(런타임 N개), <code>addTask</code> + <code>for await</code>, 완료 순서</li></ul><p>둘 다 structured concurrency — 부모가 자식 끝까지 대기 + 취소 자동 전파.</p>

**내 답안 이력**

- `2026-06-22 01:06` 비동기 병렬 처리를 런타임에 동적으로 처리가능하냐 여부가 가장 큰 차이.

---

## Q9. task.cancel()을 호출하면 진행 중인 작업이 즉시 멈추는가?

**정답**

<p>아니다. <code>cancel()</code>은 <b>취소 신호만 세팅</b>하고 강제 종료하지 않는다(협력적 취소). 작업이 직접 확인해야 멈춘다.</p><pre><code>for i in 0..&lt;n {
    if i % 1000 == 0 {
        try Task.checkCancellation()
    }
    sum += i
}</code></pre><p>체크가 한 줄도 없으면 cancel 신호가 와도 끝까지 돈다. 표준 라이브러리(URLSession, Task.sleep)는 자동 협력, 직접 짠 CPU 루프는 명시 체크 필요.</p>

**내 답안 이력**

- `2026-06-22 01:07` 아니다. cancellation 처리가 되어 있어야 멈출수 있음. 그렇지 않다면 자식 task 처리가 다 될때까지 기다려야 됨.

---

## Q10. Task.isCancelled와 try Task.checkCancellation()의 차이는?

**정답**

<ul><li><code>Task.isCancelled</code> — Bool. 어떻게 종료할지 내가 결정(return으로 부분 결과 반환, break, cleanup 등)</li><li><code>try Task.checkCancellation()</code> — 취소됐으면 <code>CancellationError</code> throw. 호출 체인 전체를 한 번에 풀어버림</li></ul>

**내 답안 이력**

- `2026-06-22 01:07` cancel 처리에 대한 예외처리가 가능하냐 안하냐가 차이임.

---

## Q11. Structured Concurrency의 "가장 중요한 룰"과 그 결과(에러 전파 시점)는?

**정답**

<p><b>부모는 모든 자식이 끝날 때까지 끝나지 않는다.</b></p><p>자식 하나가 throw하면: ①다른 자식들에 cancel 신호 ②모든 자식이 정리될 때까지 대기 ③그제서야 호출자에게 에러 throw.</p><p>자식들이 협력 안 하면(직접 짠 CPU 루프) cancel을 무시하고 끝까지 돌고, 부모도 그때까지 기다린다. 그래서 자식이 둥둥 떠다니는 fire-and-forget 상태를 구조적으로 막는다.</p>

**내 답안 이력**

- `2026-06-22 01:09` 부모 task의 취소 전파가 자식 task에게도 일괄 전파 되는 것이 가장 중요. 자식에게 즉각 전파 되지만 자식 타스크의 작업을 즉시 중단하는 것은 아님. 즉각 중단 시키기 위해선 cancellation 로직 추가가 필요함.

---
