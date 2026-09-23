# 이벤트 루프 (Event Loop)

## 질문
JavaScript의 이벤트 루프가 무엇인지 설명하라. 콜스택(Call Stack), 태스크 큐(Task Queue), 마이크로태스크 큐(Microtask Queue)가 각각 어떤 역할을 하며, 이 셋의 실행 우선순위는 어떻게 되는가?

## 내 답변 (처음 생각)
- 콜스택은 함수들이 쌓이는 곳.
- 테스크 큐는 대기공간.
- 마이크로 테스크 큐는 테스크 큐보다는 우선순위가 높은 대기공간.
- 정도로 애매하게 알고 있고 추가 설명을 요청함

## 보완된 개념

### 왜 이벤트 루프가 필요한가
JavaScript는 싱글 스레드로 동작한다. 한 번에 한 가지 일만 처리할 수 있다. 그런데 `setTimeout`, 네트워크 요청, 사용자 클릭처럼 "기다려야 하는 일"이 있는데, 기다리는 동안 JS가 멈춰있으면 페이지가 먹통이 된다. 이 기다림을 브라우저가 대신 처리하고, 끝나면 결과를 JS에 다시 넘겨주는 구조를 관리하는 게 이벤트 루프다.

### 세 가지 구성 요소

**1. 콜스택 (Call Stack)**
현재 실행 중인 함수들이 쌓이는 곳. 함수 호출 시 push, 종료 시 pop. LIFO 구조.

**2. 태스크 큐 (Task Queue, = 매크로태스크 큐)**
`setTimeout`, `setInterval`, 클릭 이벤트, `fetch` 완료 콜백 등이 대기하는 큐. 콜스택이 완전히 비면 이벤트 루프가 여기서 하나씩 꺼내 콜스택에 올린다.

**3. 마이크로태스크 큐 (Microtask Queue)**
`Promise.then/catch/finally`, `queueMicrotask`, `async/await` 이후 코드가 대기하는 큐. 태스크 큐보다 우선순위가 높다.

### 실행 순서
1. 콜스택이 비었는지 확인
2. 비었으면 → 마이크로태스크 큐를 완전히 다 비울 때까지 실행 (처리 도중 새로 추가돼도 이번 사이클에 다 처리)
3. 마이크로태스크 큐가 다 비면 → 태스크 큐에서 딱 하나만 꺼내서 실행
4. 다시 1번으로

즉 **매크로태스크 하나 실행 → 마이크로태스크 큐 전부 비우기 → 다음 매크로태스크 하나** 패턴 반복.

### 예시 코드
```js
console.log('1');

setTimeout(() => console.log('2'), 0);

Promise.resolve().then(() => console.log('3'));

console.log('4');
```

**실행 순서: `1 → 4 → 3 → 2`**
- `1`, `4`는 동기 코드라 콜스택에서 바로 실행
- `setTimeout`은 태스크 큐, `Promise.then`은 마이크로태스크 큐로 각각 등록만 되고 대기
- 동기 코드가 끝나면 마이크로태스크 큐 먼저 비움(`3`) → 그 다음 태스크 큐(`2`)

## 추가 심화 질문

### Q1. async/await는 이벤트 루프에서 어떻게 동작하나요?
`async` 함수 안에서 `await`를 만나면, 그 시점에서 함수 실행이 일시 중단되고 제어권이 호출자에게 넘어간다. `await` 뒤의 나머지 코드는 **마이크로태스크로 스케줄링**된다 — 즉 `.then()`과 동일한 우선순위를 가진다.

```js
async function foo() {
  console.log('A');
  await null; // 여기서 중단, 이후 코드는 마이크로태스크로 예약
  console.log('B');
}

console.log('start');
foo();
console.log('end');

// 실행 순서: start → A → end → B
```
`foo()` 호출 시 `console.log('A')`까지는 동기 실행되고, `await` 만나는 순간 함수를 빠져나가 `console.log('end')`가 먼저 실행된 뒤, 마이크로태스크 큐에 등록된 `console.log('B')`가 실행된다.

### Q2. requestAnimationFrame은 태스크/마이크로태스크 중 어디에 속하나요?
둘 다 아니다. `requestAnimationFrame`(rAF)은 **브라우저의 렌더링 단계 직전**에 실행되는 별도의 타이밍이다.

이벤트 루프의 한 사이클을 더 정확히 보면:
1. 태스크 큐에서 태스크 하나 실행
2. 마이크로태스크 큐 전부 비우기
3. (필요 시) 렌더링 수행 — 이 렌더링 직전에 rAF 콜백들이 실행됨
4. 다시 1번으로

즉 rAF는 "다음 화면을 그리기 직전"에 실행되도록 브라우저가 최적화해놓은 타이밍이라, 애니메이션처럼 화면 갱신과 동기화되어야 하는 작업에 적합하다. `setTimeout`으로 애니메이션을 구현하면 프레임 타이밍이 안 맞을 수 있는 반면, rAF는 브라우저 리프레시 주기(보통 60fps)에 맞춰 실행된다.

## 참고
- [MDN: Concurrency model and the event loop (영문)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop)