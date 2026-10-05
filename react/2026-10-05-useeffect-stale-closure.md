# useEffect와 stale closure

## 질문
아래 컴포넌트는 1초마다 숫자가 1씩 올라가길 기대하고 만든 것이다.

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []);

  return <div>{count}</div>;
}
```

1. 화면에는 실제로 무엇이 보이는가? 왜 그런지 "클로저"라는 단어를 넣어서 설명하라.
2. 고치는 방법을 두 가지 이상 쓰고, 각각의 트레이드오프를 설명하라. (의존성 배열에 `count`를 넣으면 interval에서는 무슨 일이 일어나는가?)
3. `return () => clearInterval(id)`는 언제 실행되는가? 이 줄이 없으면 어떤 문제가 생기는가?
4. (보너스) React 18 개발 모드의 StrictMode에서 이 effect는 몇 번 실행되는가? React는 왜 일부러 그렇게 하는가?

## 내 답변 (처음 생각)
1. 0에서 1로 올라간 뒤 멈춰 있다. 처음 시점인 0만 포착해서 1로 올리고, 그 이후에는 올라가지 않는다. 자바스크립트 클로저 속성 때문에 처음만 포착한다. 클로저는 처음 스냅샷만 바라보고 있기 때문이다.
2. - `prev => prev + 1`처럼 이전 상태값을 받도록 고치면 1초에 한 번씩 값이 올라간다.
   - `useEffect` 의존성에 `count`를 추가하면, 의존성이 변경될 때마다 `useEffect`가 실행되기 때문에 값이 올라간다. 하지만 1초, 2초, 3초 이렇게 되는 게 아니고 함수가 통째로 다시 실행되는 것이라, `useEffect` 내부에 무거운 내용이 있으면 비효율적이다.
3. 이 컴포넌트가 언마운트될 때 실행된다.
4. 2번 실행된다. 왜 일부러 그러는지는 모르겠다.

## 보완된 개념

### 1. "클로저는 스냅샷을 본다"는 정확한 표현이 아니다
결과(0 → 1에서 멈춤)와 원인(클로저)은 맞다. 하지만 클로저는 값을 복사해 두는 게 아니라 **변수 자체를 참조**한다. (9/28 클로저 노트: `let count`를 바꾸면 클로저 안에서도 바뀐 값이 보인다)

스냅샷처럼 보이는 진짜 이유는 **React의 렌더링 모델**이다.

- 렌더링될 때마다 `Counter` 함수가 새로 호출되고, 그때마다 **새로운 `const count`**가 만들어진다. 렌더 1의 `count`(0)와 렌더 2의 `count`(1)는 서로 다른 변수다.
- 의존성이 `[]`라서 effect는 첫 렌더에서 한 번만 실행된다. 그래서 interval 콜백은 **렌더 1의 `count`**를 계속 참조한다. 이 변수는 `const`라서 영원히 0이다.
- 매초 `setCount(0 + 1)`이 호출된다. 이미 값이 1이므로 React는 `Object.is`로 같은 값임을 확인하고 리렌더링을 건너뛴다.
- 화면은 멈춘 것처럼 보이지만 **interval은 계속 돌고 있다.**

```
렌더 1: count = 0  ── effect 실행 → interval 콜백이 "렌더 1의 count"를 붙잡음
  1초 후: setCount(0 + 1) → 렌더 2: count = 1  (effect 재실행 안 됨, deps = [])
  2초 후: setCount(0 + 1) → 값이 같으므로 리렌더 생략
  3초 후: setCount(0 + 1) → 리렌더 생략 ...
```

> 클로저 자체가 문제인 게 아니라, 클로저가 **오래된(stale) 렌더의 변수**를 붙잡고 있는 게 문제다. 그래서 이름이 stale closure다.

### 2. 해결 방법별 트레이드오프

**방법 A: 함수형 업데이트 `setCount(prev => prev + 1)` (가장 좋은 답)**
콜백이 더 이상 클로저에서 `count`를 읽지 않는다. React가 업데이트를 처리하는 시점의 최신 값을 `prev`로 넘겨준다. interval은 한 번만 만들어지고 계속 유지된다.

```tsx
useEffect(() => {
  const id = setInterval(() => {
    setCount((prev) => prev + 1);
  }, 1000);
  return () => clearInterval(id);
}, []);
```

- 한계: 다음 상태가 **이전 상태만으로** 계산될 때만 쓸 수 있다. `prev + step`처럼 다른 state나 props가 필요하면 `step`이 다시 stale해진다. 이럴 때는 방법 C나 `useReducer`를 쓴다.

**방법 B: 의존성 배열에 `count` 추가 (동작은 하지만 비추천)**
"1초, 2초, 3초처럼 안 된다"는 처음 답변은 정정이 필요하다. 화면에서는 **거의 1초마다 잘 올라간다.** 실제로 일어나는 일은 이렇다.

```
count 변경 → 리렌더 → cleanup(clearInterval) → 새 setInterval 등록 → 1초 후 count 변경 → ...
```

- `setInterval`이 사실상 "한 번 쓰고 버리는 `setTimeout`"이 된다.
- 렌더링과 effect 재실행 시간만큼 조금씩 밀린다(drift).
- 다른 곳(예: 버튼 클릭)에서 `count`가 바뀌면 **타이머가 처음부터 다시 시작된다.**
- effect 안의 로직이 매번 통째로 다시 실행된다는 처음 답변의 지적은 맞다.

**방법 C: `useRef`로 최신 콜백 보관 (`useInterval` 패턴)**
interval은 한 번만 만들고, 실행할 콜백만 ref로 최신 값으로 갈아끼운다. 콜백이 다른 state나 props를 읽어야 할 때 쓴다.

```tsx
function useInterval(callback: () => void, delay: number) {
  const savedCallback = useRef(callback);

  // 렌더마다 최신 콜백으로 갱신 (최신 count, step 등을 클로저로 가진 함수)
  useEffect(() => {
    savedCallback.current = callback;
  });

  useEffect(() => {
    const id = setInterval(() => savedCallback.current(), delay);
    return () => clearInterval(id);
  }, [delay]);
}

// 사용
useInterval(() => setCount(count + step), 1000); // count, step 모두 최신 값
```

ref는 렌더가 바뀌어도 **같은 객체**로 유지되므로, interval 콜백은 `savedCallback.current`를 통해 항상 가장 최근 렌더의 함수를 호출한다.

| 방법 | interval 재생성 | 다른 state/props 참조 | 복잡도 |
|---|---|---|---|
| A. 함수형 업데이트 | 없음 | 불가 (stale) | 낮음 |
| B. deps에 `count` | 매 tick마다 | 가능 | 낮음 (하지만 drift) |
| C. `useRef` + 커스텀 훅 | 없음 | 가능 | 중간 |

### 3. cleanup은 언마운트 때 "말고도" 실행된다
처음 답변은 절반만 맞다. cleanup이 실행되는 시점은 두 가지다.

1. **컴포넌트가 언마운트될 때**
2. **의존성이 바뀌어 effect가 다시 실행되기 직전.** 이때 **이전 렌더의 cleanup**이 먼저 실행된다.

방법 B에서 interval이 매초 제거되고 다시 만들어진 게 바로 2번 때문이다.

```
렌더 1 → effect 1 실행
렌더 2 (deps 변경) → cleanup 1 → effect 2 실행
렌더 3 (deps 변경) → cleanup 2 → effect 3 실행
언마운트          → cleanup 3
```

**cleanup이 없으면 생기는 문제**
- **언마운트 후에도 interval이 계속 돈다.** 사라진 컴포넌트의 `setCount`를 계속 호출하고, 클로저가 붙잡은 값들이 GC되지 않아 메모리 누수가 생긴다. (9/28 클로저 노트의 Q1과 연결)
- **interval이 쌓인다.** 방법 B처럼 deps가 바뀔 때마다 effect가 다시 실행되는데 이전 interval을 지우지 않으면, 1초마다 interval이 하나씩 늘어나서 숫자가 점점 빨리 올라간다.

### 4. StrictMode는 왜 effect를 두 번 실행하는가
React 18 이상의 **개발 모드 + StrictMode**에서는 마운트 직후 다음 순서로 동작한다.

```
mount(effect 실행) → 강제 unmount(cleanup 실행) → 다시 mount(effect 실행)
```

그래서 effect는 2번, cleanup은 그 사이에 1번 실행된다. **프로덕션 빌드에서는 한 번만 실행된다.**

**일부러 그러는 이유: cleanup이 빠진 버그를 개발 중에 미리 드러내기 위해서다.**
- effect가 "setup → cleanup → setup"을 거쳐도 결과가 한 번 실행한 것과 같아야 올바른 effect다.
- 예를 들어 위 코드에서 cleanup을 빼먹고 함수형 업데이트를 쓰면, StrictMode에서는 interval이 2개 생겨서 **1초에 2씩 올라간다.** 이렇게 버그가 바로 눈에 보인다.
- React는 앞으로 컴포넌트를 state를 유지한 채 숨겼다가 다시 보여주는 기능(예: `<Activity>`, Fast Refresh)을 지원하려고 한다. 이런 기능에서는 실제로 unmount → mount가 반복되므로, 모든 effect가 이를 견딜 수 있어야 한다.

> StrictMode의 이중 실행을 "막는 방법"(예: ref 플래그로 두 번째 실행 건너뛰기)을 찾기보다, **cleanup을 제대로 작성하는 것**이 정답이다.

## 추가 심화 질문

### Q1. 이벤트 핸들러에서도 stale closure가 생기나요?
보통은 생기지 않는다. 이벤트 핸들러는 렌더마다 새로 만들어져서 그 렌더의 최신 값을 참조하기 때문이다. 문제는 **함수가 만들어진 렌더보다 오래 살아남을 때** 생긴다. `setInterval`, `setTimeout`, `addEventListener`로 등록한 콜백, `useCallback(fn, [])`으로 고정한 함수, `await` 이후에 state를 읽는 코드가 대표적이다.

```tsx
const handleClick = async () => {
  await sleep(3000);
  alert(count); // 클릭한 시점의 count. 3초 동안 바뀐 값은 보이지 않는다
};
```

### Q2. `react-hooks/exhaustive-deps` 린트 규칙은 왜 있나요?
effect나 `useCallback`에서 사용하는 값을 deps에서 빠뜨리면 바로 이번 문제(stale closure)가 생긴다. 이 규칙은 빠진 의존성을 경고해 준다. 경고를 `// eslint-disable`로 끄기보다, 함수형 업데이트나 `useRef`로 **의존성 자체를 줄이는 쪽**으로 고치는 게 좋다.

### Q3. `useEffectEvent`는 무엇인가요?
방법 C(`useRef`로 최신 콜백 보관)를 React가 공식 API로 만든 것이다. effect 안에서 최신 props/state를 읽되, 그 값이 바뀌어도 effect를 다시 실행하지 않고 싶을 때 쓴다. (React 19.2에서 안정화)

## 참고
- [React 공식 문서: Synchronizing with Effects](https://react.dev/learn/synchronizing-with-effects)
- [React 공식 문서: useEffect](https://react.dev/reference/react/useEffect)
- [React 공식 문서: StrictMode](https://react.dev/reference/react/StrictMode)
- [React 공식 문서: State as a Snapshot](https://react.dev/learn/state-as-a-snapshot)
- [React 공식 문서: Separating Events from Effects (useEffectEvent)](https://react.dev/learn/separating-events-from-effects)
- [Dan Abramov: Making setInterval Declarative with React Hooks](https://overreacted.io/making-setinterval-declarative-with-react-hooks/)
