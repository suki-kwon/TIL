# Promise.all 직접 구현

## 질문
`Promise.all`을 직접 구현하라. (TypeScript)
- `promiseAll(promises)`로 호출하면, 모든 프로미스가 이행되었을 때 **입력 순서 그대로** 결과 배열을 담아 이행되는 프로미스를 반환한다. (먼저 끝난 순서가 아님)
- 하나라도 거부되면 즉시 그 에러로 거부된다.
- 빈 배열이면 빈 배열로 이행된다.
- 배열 안에 프로미스가 아닌 값(예: `1`, `'a'`)이 섞여 있어도 동작해야 한다.
- 제네릭을 적용해서 반환 타입이 입력 타입에 맞게 추론되도록 한다. (`Promise<number>[]` → `Promise<number[]>`)
- 결과를 `push`하지 않고 `results[index] = value`로 넣어야 하는 이유를 한 줄로 설명하라.

## 내 답변 (처음 생각)

```javascript
function customPromiseAll (promises) {
  return new Promise((resolve, reject) => {
    // promises가 배열이 아니면 reject

    const promiseArray = Array.from(promises)
    const results = []
    let completedCount = 0

    if (promiseArray.length === 0) {
      resolve(results)
      return
    }

    promiseArray.forEach((promise, index) => {
      Promise.resolve(promise)
        .then((value) => {
          results[index] = value
          completedCount++

          if (completedCount === promiseArray.length) {
            resolve(results)
          }
        }).catch((error) => {
          reject(error)
        })
    })
  })
}
```

- `results[index] = value`로 넣어야 하는 이유: 병렬로 돌리기 때문에 순서가 보장되지 않아서.

## 보완된 개념

### 처음 코드 검증
아래 요구사항은 모두 충족한 정확한 구현이었다.
- **입력 순서 유지**: `results[index] = value`로 인덱스에 넣음
- **빈 배열**: 바로 `resolve([])`
- **일반 값 섞임**: `Promise.resolve(promise)`로 감싸서 프로미스가 아닌 값도 처리
- **즉시 거부**: 하나라도 거부되면 `reject` 호출 (이미 호출된 뒤의 `resolve`/`reject`는 무시됨)

### 개선할 점 두 가지

**1. 비반복(non-iterable) 입력이 조용히 통과한다**
주석으로만 적어둔 "배열이 아니면 reject"가 구현되지 않았다. 게다가 `Array.from(5)`는 에러 없이 `[]`을 반환해서, 잘못된 입력이 "빈 배열로 성공"으로 처리된다. 진짜 `Promise.all`은 이 경우 `TypeError`로 거부한다.

```javascript
if (values == null || typeof values[Symbol.iterator] !== 'function') {
  reject(new TypeError(`${typeof values} is not iterable`));
  return;
}
```

> 주석으로만 남기지 말고, 한 줄이라도 구현하거나 "시간 관계상 생략하겠다"고 말로 짚고 넘어가는 편이 라이브 코딩에서 더 좋다.

**2. `.then(...).catch(...)` 대신 `.then(onFulfilled, reject)`**
`.catch`는 `then` 콜백 안에서 발생한 에러까지 잡는다. 지금 코드에서는 문제가 안 되지만, 거부 처리를 "원본 프로미스의 거부"에만 한정하려는 의도라면 `then`의 두 번째 인자로 `reject`를 넘기는 쪽이 더 명확하다.

### 개선된 JavaScript 버전

```javascript
function customPromiseAll(values) {
  return new Promise((resolve, reject) => {
    // 1) 반복 불가능한 입력은 TypeError로 거부 (진짜 Promise.all과 동일)
    if (values == null || typeof values[Symbol.iterator] !== 'function') {
      reject(new TypeError(`${typeof values} is not iterable`));
      return;
    }

    const items = Array.from(values);
    const results = new Array(items.length); // 길이를 미리 확보
    let completed = 0;

    // 2) 빈 배열은 즉시 이행
    if (items.length === 0) {
      resolve(results);
      return;
    }

    items.forEach((item, index) => {
      // 3) 프로미스가 아닌 값도 Promise.resolve로 감싸서 동일하게 처리
      Promise.resolve(item).then((value) => {
        results[index] = value; // 4) push가 아니라 index로 넣어서 입력 순서 유지
        completed++;
        if (completed === items.length) resolve(results);
      }, reject); // 5) 하나라도 거부되면 즉시 거부
    });
  });
}
```

실행 결과 (직접 확인):
```
all order     [ 'a', 'b', 'c' ]       // 입력: sleep(30,'a'), sleep(10,'b'), 'c'
all empty     []
all non-iter  TypeError: number is not iterable
all reject    boom
```

### 왜 push가 아니라 results[index]인가
각 프로미스가 끝나는 시점은 제각각이라서, **완료 순서와 입력 순서는 다르다.** `push`로 쌓으면 먼저 끝난 순서대로 들어간다. 반면 `results[index]`는 "이 값이 원래 몇 번째 입력의 결과인지"를 인덱스로 기억하고 있어서, 언제 끝나든 제자리에 들어간다. (클로저가 `index`를 캡처하고 있다는 점도 함께 기억하기)

```javascript
// 입력 순서: A(30ms), B(10ms), C(20ms)
// push로 쌓으면      → [ 'B', 'C', 'A' ]   (끝난 순서)
// results[index]로   → [ 'A', 'B', 'C' ]   (입력 순서)
```

### "병렬로 돌린다"는 표현 정정
처음 답변에서 "병렬로 돌리기 때문에"라고 했는데, 두 가지가 부정확하다.

1. **병렬(parallel)이 아니라 동시(concurrent)다.** JS 메인 스레드는 하나라서 콜백이 실제로 동시에 실행되지는 않는다. 네트워크 요청이나 타이머 같은 대기 작업이 **겹쳐서 진행**될 뿐이고, 완료 콜백은 이벤트 루프를 통해 하나씩 차례로 실행된다.
2. **`Promise.all`이 작업을 "실행시키는" 게 아니다.** 프로미스는 **생성되는 순간 이미 작업이 시작된다.** `Promise.all([fetch(a), fetch(b)])`에서 두 요청은 `fetch()`가 호출된 시점에 이미 출발했고, `Promise.all`은 그 결과를 **모아서 기다리는 역할**만 한다.

```javascript
const p1 = fetch('/a'); // 여기서 이미 요청 시작
const p2 = fetch('/b'); // 여기서 이미 요청 시작
await Promise.all([p1, p2]); // 시작시키는 게 아니라 둘 다 끝나길 기다릴 뿐
```

이 관점이 있어야 "하나가 거부되면 나머지는 취소되나?"(Q4)에 "취소 안 된다, `Promise.all`은 결과만 모으는 역할이니까"라고 자연스럽게 답할 수 있다.

### TypeScript 버전 (제네릭 적용)

```typescript
export function customPromiseAll<T>(
  values: Iterable<T>
): Promise<Awaited<T>[]> {
  return new Promise((resolve, reject) => {
    if (values == null || typeof (values as any)[Symbol.iterator] !== 'function') {
      reject(new TypeError(`${typeof values} is not iterable`));
      return;
    }

    const items = Array.from(values);
    const results: Awaited<T>[] = new Array(items.length);
    let completed = 0;

    if (items.length === 0) {
      resolve(results);
      return;
    }

    items.forEach((item, index) => {
      Promise.resolve(item).then((value) => {
        results[index] = value;
        completed++;
        if (completed === items.length) resolve(results);
      }, reject);
    });
  });
}
```

타입 추론 확인 (`tsc --strict`로 컴파일 검증):
```typescript
const n: Promise<number[]> = customPromiseAll([Promise.resolve(1), Promise.resolve(2)]);
const mixed: Promise<(string | number)[]> = customPromiseAll([1, 'a', Promise.resolve(2)]);
const mixedPromises: Promise<(string | number)[]> = customPromiseAll([Promise.resolve(1), Promise.resolve('x')]);
// @ts-expect-error string[]은 number[]에 할당 불가
const bad: Promise<number[]> = customPromiseAll([Promise.resolve('x')]);
```

**반환 타입이 `Promise<T[]>`가 아니라 `Promise<Awaited<T>[]>`인 이유**
`T`가 `Promise<number>`일 수도 있는데, 결과 배열에 담기는 건 프로미스가 아니라 **벗겨진 값**(`number`)이다. `Awaited<T>`는 중첩된 프로미스를 끝까지 벗겨낸 값 타입을 표현한다.

**처음엔 `Iterable<T | PromiseLike<T>>`로 썼다가 고친 이유**
그 시그니처는 `[Promise.resolve(1), Promise.resolve('x')]`처럼 **서로 다른 타입의 프로미스가 섞이면 컴파일 에러**가 난다.

```
error TS2345: Argument of type '(Promise<string> | Promise<number>)[]' is not assignable
to parameter of type 'Iterable<string | PromiseLike<string>>'.
```

TS가 `T`를 후보 중 하나(`string`)로만 추론한 뒤 `Promise<number>`가 `PromiseLike<string>`에 맞지 않는다고 판단하기 때문이다. 입력을 `Iterable<T>`로 받으면 `T = Promise<number> | Promise<string>`으로 추론되고, `Awaited`가 유니온의 각 멤버에 **분배**되어 적용되므로 `Awaited<T> = number | string`이 된다. 벗기는 일을 입력 쪽이 아니라 `Awaited`에 전부 맡기는 쪽이 더 단순하고 정확하다.

**더 정확하게: 튜플 타입까지 유지하기**
위 버전은 `[Promise<number>, Promise<string>]` → `Promise<(number | string)[]>`로 위치 정보가 사라진다. 실제 `Promise.all`의 타입 정의(`lib.es2015.promise.d.ts`)는 매핑된 타입으로 튜플을 그대로 유지한다.

```typescript
export function customPromiseAllTuple<T extends readonly unknown[] | []>(
  values: T
): Promise<{ -readonly [K in keyof T]: Awaited<T[K]> }> {
  return customPromiseAll(values) as any; // 런타임 구현은 동일, 타입만 정교하게
}

const tuple: Promise<[number, string]> = customPromiseAllTuple([Promise.resolve(1), Promise.resolve('x')]);
```

- `T extends readonly unknown[] | []`: `| []`를 붙이면 TS가 배열 리터럴을 일반 배열(`(A | B)[]`)이 아니라 **튜플**(`[A, B]`)로 추론한다.
- `{ [K in keyof T]: Awaited<T[K]> }`: 튜플의 각 자리를 돌면서 프로미스를 벗긴다. 배열/튜플에 매핑된 타입을 쓰면 결과도 배열/튜플 모양이 유지된다.
- `-readonly`: 입력이 `readonly` 튜플(`as const`)이어도 결과에서는 `readonly`를 떼어낸다.

## 추가 심화 질문

### Q1. Promise.allSettled는 어떻게 구현하나요? all과의 차이는?
`all`은 하나라도 거부되면 즉시 거부되지만, `allSettled`는 **거부되어도 멈추지 않고 모든 프로미스가 끝날 때까지 기다린 뒤** 각각의 결과를 `{ status, value | reason }` 형태로 돌려준다. 그래서 절대 거부되지 않는다.

구현 아이디어: 각 프로미스를 "절대 거부되지 않는 프로미스"로 바꿔서 `customPromiseAll`에 넘긴다.

```javascript
function customAllSettled(values) {
  // Array.from(5)는 에러 없이 []를 반환하므로, all과 똑같이 먼저 걸러야 한다
  if (values == null || typeof values[Symbol.iterator] !== 'function') {
    return Promise.reject(new TypeError(`${typeof values} is not iterable`));
  }

  return customPromiseAll(
    Array.from(values, (item) =>
      Promise.resolve(item).then(
        (value) => ({ status: 'fulfilled', value }),
        (reason) => ({ status: 'rejected', reason })
      )
    )
  );
}
```

실행 결과 (입력: 이행 1, 거부 `'x'`, 일반값 3):
```javascript
[
  { status: 'fulfilled', value: 1 },
  { status: 'rejected', reason: 'x' },
  { status: 'fulfilled', value: 3 },
]
```
순서가 유지되고, 거부가 섞여 있어도 전체는 이행된다. 이행 결과는 `value`, 거부 결과는 `reason` 키에 담긴다는 점이 핵심이다.

사용처: 여러 API를 동시에 호출하되, 일부가 실패해도 성공한 결과는 쓰고 싶을 때. (예: 대시보드의 여러 위젯 데이터 로딩)

### Q2. Promise.race는 어떻게 구현하나요?
가장 먼저 **끝난(이행이든 거부든)** 프로미스의 결과로 정해진다. 이미 정해진 프로미스에 `resolve`/`reject`를 또 호출해도 무시되는 성질을 이용하면 아주 간단하다.

```javascript
function customRace(values) {
  return new Promise((resolve, reject) => {
    for (const item of values) {
      Promise.resolve(item).then(resolve, reject);
    }
  });
}
```

실행 결과: `fast` (입력: sleep(30,'slow'), sleep(5,'fast'))

**엣지 케이스: 빈 배열**
`race([])`는 **영원히 pending 상태**로 남는다. 경쟁할 프로미스가 하나도 없어서 `resolve`/`reject`가 호출될 일이 없기 때문이다. 빈 배열이 즉시 `[]`로 이행되는 `all`과 대비되는 지점이다. (진짜 `Promise.race([])`도 똑같이 동작한다)

사용처: 요청에 타임아웃을 거는 패턴. `Promise.race([fetch(url), timeout(3000)])`

### Q3. Promise.all, allSettled, race, any를 한 줄씩 비교하면?
| 메서드 | 이행 조건 | 거부 조건 |
|---|---|---|
| `all` | 전부 이행 | 하나라도 거부되면 즉시 거부 |
| `allSettled` | 전부 끝나면 (항상 이행) | 거부되지 않음 |
| `race` | 가장 먼저 끝난 게 이행이면 | 가장 먼저 끝난 게 거부면 |
| `any` | 하나라도 이행되면 (첫 번째 이행) | 전부 거부되면 (`AggregateError`) |

### Q4. Promise.all에서 하나가 거부되면 나머지 프로미스는 취소되나요?
**취소되지 않는다.** `Promise.all`은 "결과를 어떻게 합칠지"만 정할 뿐, 이미 시작된 비동기 작업을 멈추는 기능이 없다. 거부된 이후에도 나머지 요청은 계속 진행되고, 그 결과는 무시된다. 진짜로 중단하려면 `AbortController`로 요청 자체를 취소해야 한다.

### Q5. 이벤트 루프 관점에서, `Promise.resolve(item).then(...)`의 콜백은 언제 실행되나요?
`then` 콜백은 **마이크로태스크 큐**에서 실행된다. 이미 이행된 프로미스(또는 일반 값을 `Promise.resolve`로 감싼 것)라도 `then` 콜백은 현재 동기 코드가 끝난 뒤에 실행된다. 그래서 위 구현에서 `results[index] = value`는 `forEach`가 모두 끝난 뒤에야 처음 실행된다. (2일차 이벤트 루프 내용 복습)

## 참고
- [MDN: Promise.all()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)
- [MDN: Promise.allSettled()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled)
- [MDN: Promise.race()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/race)
- [MDN: Promise.any()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/any)
- [TypeScript Handbook: Awaited\<Type\>](https://www.typescriptlang.org/docs/handbook/utility-types.html#awaitedtype)
