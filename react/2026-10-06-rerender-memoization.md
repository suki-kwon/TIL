# React 리렌더링과 memo / useMemo / useCallback

## 질문
```tsx
const Child = React.memo(function Child({
  onClick,
  style,
}: {
  onClick: () => void;
  style: React.CSSProperties;
}) {
  console.log('Child render');
  return <button onClick={onClick} style={style}>+1</button>;
});

function Parent() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('');

  const handleClick = () => setCount((c) => c + 1);

  return (
    <>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <p>{count}</p>
      <Child onClick={handleClick} style={{ color: 'red' }} />
    </>
  );
}
```

1. React 컴포넌트가 리렌더링되는 조건을 아는 만큼 써라.
2. `input`에 글자를 입력하면 콘솔에 `Child render`가 찍히는가? `React.memo`로 감쌌는데도 그렇다면 왜인가?
3. 입력할 때 `Child`가 리렌더링되지 않도록 고쳐라. 어떤 훅을 어디에 쓰는지, 왜 필요한지 설명하라.
4. 모든 함수와 객체를 `useCallback`/`useMemo`로 감싸는 게 좋은가? 비용과 트레이드오프를 설명하라.
5. (보너스) `memo`, `useMemo`, `useCallback` 없이 컴포넌트 구조만 바꿔서 입력할 때 `Child`가 리렌더링되지 않게 만들 수 있는가?

## 내 답변 (처음 생각)
1. - `useState` 같은 상태가 업데이트될 때
   - props 값이 바뀔 때
   - 부모 컴포넌트가 리렌더링될 때
   - `useContext`로 구독하는 context가 업데이트될 때
2. 네. 넘겨주는 함수가 매번 새로 생성되기 때문이다.
3. `useCallback`으로 감싸줘야 한다. 함수의 인스턴스를 재사용하기 때문이다.
4. 그건 아니다. 함수의 연산이 매우 복잡한 경우가 아니라면 그냥 두는 게 낫다. 가독성 측면에서도 그렇다.
5. 상태 분리...? 잘 모르겠다.

## 보완된 개념

### 1. "props가 바뀔 때"는 독립된 리렌더링 조건이 아니다
props는 **부모가 렌더링될 때만** 새로 만들어진다. 부모가 렌더링되지 않았는데 자식의 props만 바뀌는 일은 없다. 그래서 리렌더링 조건은 사실 3가지다.

| 조건 | 설명 |
|---|---|
| 자신의 state 변경 | `setState`. 단, `Object.is`로 비교해서 같은 값이면 건너뛴다 (어제 노트의 `setCount(0 + 1)` 사례) |
| 부모의 리렌더링 | **props가 바뀌었든 안 바뀌었든** 자식도 모두 다시 렌더링된다 |
| 구독 중인 context 값 변경 | `useContext`로 읽는 Provider의 `value`가 바뀔 때. `memo`로도 막을 수 없다 |

"props가 바뀌면 리렌더링된다"는 **`memo`로 감싼 컴포넌트에만** 해당한다. `memo`는 "부모가 렌더링되더라도 props가 같으면 건너뛰어라"라는 **예외 규칙**이다. 그러니 순서를 바꿔서 기억하면 된다. 기본은 "부모가 렌더링되면 자식도 렌더링된다"이고, `memo`가 붙은 경우에만 props를 비교한다.

### 2. 새로 만들어지는 건 함수만이 아니다. `style` 객체도 마찬가지다
처음 답변은 `handleClick`만 짚었는데, **`style={{ color: 'red' }}`도 렌더마다 새 객체**가 된다.

`memo`는 props를 하나씩 **얕게 비교(`Object.is`)**한다. 원시값(문자열, 숫자)은 값으로 비교하지만, 함수·객체·배열은 **참조(주소)**로 비교한다. 내용이 같아도 새로 만들어졌으면 다른 값이다.

```javascript
Object.is({ color: 'red' }, { color: 'red' }); // false
Object.is(() => {}, () => {});                 // false
Object.is('red', 'red');                       // true
```

입력할 때 일어나는 일:
```
글자 입력 → setText → Parent 리렌더
  → handleClick 새 함수, style 새 객체 생성
  → memo가 이전 props와 비교: onClick 다름, style 다름 → Child 리렌더
```

### 3. `useCallback`만으로는 해결되지 않는다
`useCallback`만 적용하면 `onClick`은 같아지지만 `style`이 여전히 매번 새 객체라서 `memo` 비교에서 진다. **props 중 하나라도 다르면 리렌더링된다.**

실제 React 18로 마운트 후 3글자를 입력하고 `Child` 렌더 횟수를 셌다:

```
original       (아무것도 안 함)        → Child 렌더 4번 (마운트 1 + 입력 3)
onlyCallback   (useCallback만)         → Child 렌더 4번  ← 효과 없음
fixed          (useCallback + style 고정) → Child 렌더 1번 (마운트만)
colocated      (memo 없이 구조 변경)    → Child 렌더 1번 (마운트만)
```

**고친 코드**
```tsx
// style이 항상 똑같다면 컴포넌트 밖 상수로 빼는 게 가장 간단하다 (useMemo 불필요)
const buttonStyle = { color: 'red' };

function Parent() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('');

  // 함수형 업데이트 덕분에 deps가 []여도 stale closure가 생기지 않는다 (10/5 노트)
  const handleClick = useCallback(() => setCount((c) => c + 1), []);

  return (
    <>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <p>{count}</p>
      <Child onClick={handleClick} style={buttonStyle} />
    </>
  );
}
```

- `style`이 props나 state에 따라 바뀐다면 그때 `useMemo(() => ({ color }), [color])`를 쓴다.
- **어제와의 연결:** `useCallback(() => setCount(count + 1), [])`처럼 쓰면 어제 본 stale closure가 그대로 생긴다. `useCallback`의 deps도 `useEffect`의 deps와 똑같이 동작한다.

**"함수의 인스턴스를 재사용한다"를 정확히 말하면**
`useCallback`을 써도 **함수는 렌더마다 새로 만들어진다.** `useCallback(() => ..., [])`의 인자인 화살표 함수는 렌더할 때마다 생성된다. 다만 React가 deps가 같으면 그 새 함수를 버리고 **이전에 저장해 둔 함수를 반환**할 뿐이다. 그래서 `useCallback`은 생성 비용을 아끼는 도구가 아니라 **참조를 같게 유지하는 도구**다.

### 4. 다 감싸면 안 되는 이유: 기준은 "연산이 복잡한가"가 아니다
결론(다 감쌀 필요 없다)은 맞다. 다만 기준이 조금 다르다. "연산이 복잡하면 쓴다"는 `useMemo`의 용도 중 하나일 뿐이고, `useCallback`에는 해당하지 않는다. (위에서 봤듯이 함수 생성 비용은 줄어들지 않는다)

**메모이제이션이 의미 있는 경우는 딱 세 가지다.**
1. **비싼 계산** 결과를 재사용할 때 (`useMemo`). 예: 수천 개 항목 필터링·정렬
2. **`memo`로 감싼 자식**에게 함수나 객체를 넘길 때 (`useCallback`/`useMemo`)
3. **다른 훅의 deps**로 쓰이는 함수나 객체일 때. 예: `useEffect(() => {...}, [options])`에서 `options`가 매번 새 객체면 effect가 매 렌더마다 다시 실행된다

**이 경우가 아닌데 감싸면 생기는 비용**
- 자식이 `memo`가 아니면 **아무 효과가 없다.** 어차피 부모가 렌더링되면 자식도 렌더링된다.
- deps 배열을 매번 비교하고, 이전 값을 메모리에 들고 있어야 한다.
- deps를 잘못 쓰면 **stale closure 버그**가 생긴다. (어제 내용)
- 처음 답변에서 말한 것처럼 코드가 읽기 어려워진다.
- `memo`와 `useCallback`/`useMemo`는 **세트**다. 위 실험처럼 하나라도 빠지면 전부 효과가 없다. 그래서 일부만 적용하면 쓸모없는 코드만 남기 쉽다.

> 참고: **React Compiler**는 빌드할 때 이런 메모이제이션을 자동으로 넣어준다. 이 컴파일러를 쓰는 프로젝트에서는 직접 `useMemo`/`useCallback`을 쓸 일이 크게 줄어든다.

### 5. 구조로 해결하기: 상태 분리(state colocation)
"상태 분리"라고 한 방향이 맞다. 핵심은 **state를 그 state가 필요한 컴포넌트까지 내려보내는 것**이다.

지금은 `text`가 `Parent`에 있어서, 입력할 때마다 `Parent`와 그 자식 전부가 다시 렌더링된다. 하지만 `text`를 쓰는 건 `input`뿐이다. `input`과 `text`를 별도 컴포넌트로 빼면, 입력할 때 그 컴포넌트만 렌더링된다.

```tsx
function SearchInput() {
  const [text, setText] = useState(''); // text는 여기서만 쓰인다
  return <input value={text} onChange={(e) => setText(e.target.value)} />;
}

function Parent() {
  const [count, setCount] = useState(0);
  return (
    <>
      <SearchInput />
      <p>{count}</p>
      {/* memo도, useCallback도, 상수 style도 없다 */}
      <Child onClick={() => setCount((c) => c + 1)} style={{ color: 'red' }} />
    </>
  );
}
```

위 실험의 `colocated` 결과처럼, `memo` 없이도 입력할 때 `Child`가 다시 렌더링되지 않는다. 입력은 `SearchInput`의 state만 바꾸고, `Parent`는 렌더링되지 않기 때문이다.

**`children`으로 넘기기 (state를 위로 올려야 할 때)**
state를 가진 컴포넌트가 어떤 자식을 **감싸야** 한다면, 그 자식을 `children`으로 받는다.

```tsx
function ScrollTracker({ children }: { children: React.ReactNode }) {
  const [scrollY, setScrollY] = useState(0); // 스크롤마다 바뀜
  // ...
  return <div onScroll={(e) => setScrollY(e.currentTarget.scrollTop)}>{children}</div>;
}

function Page() {
  return (
    <ScrollTracker>
      <ExpensiveTree /> {/* 스크롤해도 리렌더링되지 않는다 */}
    </ScrollTracker>
  );
}
```

`<ExpensiveTree />`라는 요소는 `ScrollTracker`가 아니라 **`Page`가 렌더링될 때 만들어진다.** `ScrollTracker`가 다시 렌더링되어도 `children`은 `Page`가 넘겨준 **같은 객체**이므로, React는 그 부분을 건너뛴다.

> 순서: **먼저 구조(상태 분리, children)를 고민하고, 그래도 안 되면 `memo`를 쓴다.** 구조로 해결하면 deps를 관리할 필요도, stale closure를 걱정할 필요도 없다.

## 추가 심화 질문

### Q1. `React.memo`의 두 번째 인자는 무엇인가요?
기본 얕은 비교 대신 직접 비교 함수를 넣을 수 있다. `true`를 반환하면 "같다"는 뜻이라 렌더링을 건너뛴다. (`shouldComponentUpdate`와 반대이므로 헷갈리지 않게 주의)

```tsx
const Child = memo(ChildImpl, (prev, next) => prev.user.id === next.user.id);
```

비교 함수에서 빠뜨린 prop은 갱신되지 않으므로, 함수 prop을 빠뜨리면 그 함수가 stale해지는 위험이 있다.

### Q2. Context 값이 객체일 때 생기는 문제는?
```tsx
<ThemeContext.Provider value={{ theme, setTheme }}>
```
Provider를 가진 컴포넌트가 렌더링될 때마다 `value`가 새 객체가 되고, 이 context를 구독하는 **모든 컴포넌트가 리렌더링된다.** `memo`로 감싸도 막을 수 없다. 해결책은 `value`를 `useMemo`로 감싸거나, 자주 바뀌는 값과 거의 안 바뀌는 값을 별도 context로 나누는 것이다. 디자인 시스템의 Theme Provider에서 자주 마주치는 문제다.

### Q3. 리렌더링을 확인하는 방법은?
- React DevTools Profiler의 "Highlight updates when components render" 옵션
- Profiler에서 "Why did this render?" 확인
- 개발 중에는 `console.log`나 렌더 횟수를 세는 `useRef`

## 참고
- [React 공식 문서: memo](https://react.dev/reference/react/memo)
- [React 공식 문서: useCallback](https://react.dev/reference/react/useCallback)
- [React 공식 문서: useMemo](https://react.dev/reference/react/useMemo)
- [React 공식 문서: Render and Commit](https://react.dev/learn/render-and-commit)
- [React 공식 문서: React Compiler](https://react.dev/learn/react-compiler)
- [Dan Abramov: Before You memo()](https://overreacted.io/before-you-memo/)
