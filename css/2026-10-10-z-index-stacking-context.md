# z-index와 쌓임 맥락(Stacking Context)

## 질문
```html
<header class="header">로고 / 메뉴</header>

<main class="content">
  <div class="card">
    <button>옵션 ▾</button>
    <ul class="dropdown">
      <li>수정</li>
      <li>삭제</li>
    </ul>
  </div>
</main>
```

```css
.header {
  position: sticky;
  top: 0;
  z-index: 10;
}

.content {
  position: relative;
  z-index: 1;
}

.card {
  position: relative;
}

.dropdown {
  position: absolute;
  top: 100%;
  z-index: 9999;
}
```

스크롤하면 `.dropdown`이 sticky `.header` 아래로 숨는다. `z-index: 9999`인데도.

1. `9999`가 `10`보다 큰데 왜 드롭다운이 헤더 아래에 깔리는가? 쌓임 맥락(stacking context)으로 설명하라.
2. 쌓임 맥락을 새로 만드는 CSS 속성을 아는 만큼 써라.
3. 드롭다운이 헤더 위로 올라오게 고치는 방법을 두 가지 이상 쓰고, 각각의 트레이드오프를 설명하라.
4. `position: static`인 요소에 `z-index: 100`을 주면 효과가 있는가? 예외는?
5. (보너스) 디자인 시스템에서 모달, 드롭다운, 토스트의 z-index를 어떻게 관리하면 좋은가? 모달을 보통 `body` 바로 아래에 렌더링하는 이유는?

## 내 답변 (처음 생각)
1. "쌓임 맥락"이라는 한국어는 낯설다. stacking context도 머릿속에는 있었지만 말로 설명해 본 적은 거의 없다. stacking context 때문이다. z-index로 모든 게 해결되는 게 아니라, 그룹을 지어서 그 안에서 z-index를 다툰다. 종속된 부모 그룹이 있어서 헤더 아래에 깔린다.
2. `position`, `z-index`가 가장 일반적이지만 `display`도 영향을 주는 걸로 안다. `opacity`, `transform` 같은 것들도 있지 않을까?
3. - portal. 다만 portal은 맨 위에 쌓이는 거라서 브라우저 크기나 마우스 위치가 달라질 수 있어 계산 로직이 필요할 수 있다.
   - `.content`의 z-index를 조정한다. 트레이드오프는 잘 모르겠다.
4. static인 요소에는 z-index가 효과가 없지 않나? 기억이 명확하지 않다. `position` 속성도 정리해서 공부해 두고 싶다.
5. 레이어 계층을 나눠서 토큰화해서 쓴다. 다들 맨 위에 올리고 싶으면 `z-index: 9999`를 쓸 텐데, 이렇게 무분별하게 쓰는 걸 막을 수 있고 규칙이 있으면 개발할 때도 편하다. 모달을 `body` 바로 아래에 두는 이유는 모달 안에서도 dropdown 같은 게 올라갈 수 있는데, `body` 아래에 있어야 방해를 받지 않기 때문이다.

## 보완된 개념

### 1. "그룹 안에서만 다툰다"가 정확한 설명이다
처음 답변의 "그룹을 지어서 그 안에서 z-index를 다툰다"는 핵심을 정확히 짚었다. 면접에서는 이렇게 말하면 된다.

> **쌓임 맥락(stacking context)**은 z-index를 비교하는 **독립된 그룹**이다. z-index는 같은 쌓임 맥락 안에 있는 형제끼리만 비교된다. 쌓임 맥락을 만든 요소는 바깥에서 볼 때 **자식 전체를 포함한 하나의 덩어리**로 취급된다.

```
root 쌓임 맥락
├── .header   (z-index: 10)  ← 독립된 덩어리
└── .content  (z-index: 1)   ← 독립된 덩어리 (position + z-index로 쌓임 맥락 생성)
     └── .card
          └── .dropdown (z-index: 9999)  ← .content 안에서만 의미 있는 값
```

root에서 비교하는 건 `.header(10)`과 `.content(1)`뿐이다. `.dropdown`의 9999는 `.content` 안에서만 의미가 있고, 바깥에서 보면 `.content` 덩어리 전체가 1층에 있다.

**버전 번호처럼 생각하기**
- `.header` → `10`
- `.dropdown` → `1.9999`
- `10 > 1.9999`이므로 헤더가 위에 온다. 앞자리(부모의 z-index)에서 이미 승부가 났기 때문에, 뒷자리(9999)는 아무리 커도 소용없다.

> 용어: "쌓임 맥락"은 MDN 한국어 문서의 번역어다. 면접에서는 "스태킹 컨텍스트"라고 말해도 충분히 통한다.

### 2. 쌓임 맥락을 만드는 속성
`opacity`, `transform`을 떠올린 건 정확하다. `display`는 **단독으로는 만들지 않는다.** 대신 부모가 `display: flex`/`grid`일 때 그 **자식**이 `z-index`를 가지면 쌓임 맥락이 생긴다. (4번과 연결)

| 분류 | 조건 |
|---|---|
| 기본 | 루트 요소 `<html>` |
| position | `relative`/`absolute` + `z-index`가 `auto`가 아닌 값 |
| position | `fixed`, `sticky` (**z-index가 없어도** 항상 생성) |
| flex/grid | flex/grid 아이템 + `z-index`가 `auto`가 아닌 값 |
| 시각 효과 | `opacity` < 1 |
| 시각 효과 | `transform`, `filter`, `backdrop-filter`, `perspective`, `clip-path`, `mask`가 `none`이 아닌 값 |
| 시각 효과 | `mix-blend-mode`가 `normal`이 아닌 값 |
| 명시적 | `isolation: isolate` |
| 기타 | `will-change`에 위 속성 지정, `contain: paint`/`layout`, `container-type: size`/`inline-size` |

**실무에서 자주 당하는 경우:** 카드에 hover 애니메이션으로 `transform: translateY(-2px)`나 `opacity: 0.9`를 넣으면, 그 순간 카드가 쌓임 맥락이 된다. 그래서 hover할 때만 드롭다운이 다른 요소 아래로 숨는 버그가 생긴다.

### 3. 해결 방법과 트레이드오프

**방법 A: `.content`의 `z-index` 제거 (`auto`로)**
```css
.content {
  position: relative;
  /* z-index: 1; 제거 */
}
```
- `.content`가 더 이상 쌓임 맥락을 만들지 않는다. 그래서 `.dropdown`의 9999가 root 쌓임 맥락에서 `.header`의 10과 직접 비교된다.
- 트레이드오프: `z-index: 1`이 **원래 왜 있었는지** 먼저 확인해야 한다. 배경 장식 요소 위에 올리려고 넣었다면 그쪽이 깨진다. 또 나중에 누군가 `.content`나 `.card`에 `transform` 하나만 넣어도 다시 같은 버그가 생긴다. 구조가 바뀌면 쉽게 깨지는 해결책이다.

**방법 B: `.content`의 `z-index`를 올리기 (예: 20) → 비추천**
처음 답변의 "content의 z-index 조정"을 올리는 쪽으로 하면 생기는 문제다.
- `.content` 덩어리 **전체**가 헤더 위로 올라간다. 스크롤하면 본문이 sticky 헤더를 덮어 버린다.
- 드롭다운 하나를 위해 레이아웃 전체의 층 순서를 바꾸는 셈이다.

**방법 C: Portal로 `body` 바로 아래에 렌더링 (디자인 시스템의 표준)**
```tsx
import { createPortal } from 'react-dom';

function Dropdown({ anchorRect, children }) {
  return createPortal(
    <ul style={{ position: 'fixed', top: anchorRect.bottom, left: anchorRect.left, zIndex: 1000 }}>
      {children}
    </ul>,
    document.body
  );
}
```
- 어떤 쌓임 맥락에도 갇히지 않는다. 부모의 `overflow: hidden`에 잘리지도 않는다.
- 트레이드오프 (처음 답변에서 짚은 부분):
  - DOM 위치가 버튼과 떨어지므로 `getBoundingClientRect()`로 **위치를 직접 계산**해야 한다. 스크롤, 리사이즈 때 다시 계산해야 하고, 화면 아래 공간이 부족하면 위로 뒤집는(flip) 처리도 필요하다. 그래서 보통 **Floating UI** 같은 라이브러리를 쓴다.
  - DOM 순서가 바뀌어서 **키보드 포커스 순서**가 어긋난다. 열릴 때 포커스를 드롭다운으로 옮기고, 닫힐 때 버튼으로 돌려주는 처리가 필요하다.
  - 참고: React의 이벤트는 portal이어도 **React 트리 기준**으로 버블링된다. DOM상으로는 `body` 아래에 있어도, React에서는 원래 부모의 `onClick`까지 이벤트가 올라간다.

**방법 D: Top layer 사용 (Popover API, `<dialog>`)**
```html
<button popovertarget="menu">옵션 ▾</button>
<ul id="menu" popover>
  <li>수정</li>
  <li>삭제</li>
</ul>
```
- `popover` 속성이나 `dialog.showModal()`로 연 요소는 브라우저의 **top layer**에 올라간다. top layer는 모든 쌓임 맥락과 z-index **위에** 있는 별도 층이라서, portal 없이도 갇히지 않는다.
- 바깥 클릭이나 Esc로 닫기 같은 동작도 브라우저가 기본으로 처리해 준다.
- 트레이드오프: 위치 계산은 여전히 필요하다. CSS Anchor Positioning(`anchor-name`, `position-anchor`)으로 해결할 수 있지만, 브라우저 지원 범위를 확인해야 한다.

| 방법 | 장점 | 단점 |
|---|---|---|
| A. 부모 z-index 제거 | 코드 한 줄 | 구조 변경에 약함, 원래 의도 확인 필요 |
| B. 부모 z-index 올리기 | 간단 | 본문이 헤더를 덮음 (비추천) |
| C. Portal | 어디서든 안전 | 위치 계산, 포커스 관리 필요 |
| D. Top layer | z-index 관리 불필요, 기본 동작 제공 | 위치 계산 필요, 최신 API |

### 4. `position: static`에서는 z-index가 무시된다. 예외는 flex/grid 아이템
처음 답변(효과 없음)이 맞다. 예외는 하나다. **flex/grid 컨테이너의 자식**은 `position: static`이어도 `z-index`가 적용되고, 쌓임 맥락도 생긴다. 2번에서 `display`를 떠올린 게 이 부분과 연결된다.

```css
.parent { display: flex; }
.child  { z-index: 2; } /* position 없이도 동작 */
```

**`position` 속성 정리**

| 값 | 기준 위치 | 문서 흐름 | `top`/`left` 등 | z-index | 쌓임 맥락 |
|---|---|---|---|---|---|
| `static` (기본값) | 없음 | 유지 | 무시 | 무시 (flex/grid 아이템 예외) | X |
| `relative` | **원래 자기 위치** | 유지 (원래 자리 비워 둠) | 원래 위치에서 이동 | 적용 | z-index 지정 시 |
| `absolute` | **가장 가까운 positioned 조상** (`static`이 아닌 조상, 없으면 초기 컨테이닝 블록) | **빠짐** | 기준 요소에서 배치 | 적용 | z-index 지정 시 |
| `fixed` | **뷰포트** | **빠짐** | 뷰포트에서 배치 | 적용 | **항상** |
| `sticky` | **가장 가까운 스크롤 컨테이너** | 유지 | 임계값(`top: 0` 등) 지정 필수 | 적용 | **항상** |

**꼭 기억할 함정 3가지**
1. **`absolute`의 기준:** 이번 예시에서 `.card`에 `position: relative`를 준 이유가 이것이다. `.dropdown`이 `.card` 기준으로 배치되게 하려고 준 것이다. (`relative`는 "기준점 만들기" 용도로 가장 많이 쓴다)
2. **`fixed`가 뷰포트 기준이 아니게 되는 경우:** 조상에 `transform`, `filter`, `perspective`가 있으면 그 조상이 기준이 된다. 그래서 `transform` 애니메이션이 걸린 요소 안에 모달을 `fixed`로 두면 화면 전체를 덮지 못한다.
3. **`sticky`가 동작하지 않는 경우:** `top` 같은 임계값을 지정하지 않았거나, 조상에 `overflow: hidden`/`auto`가 있어서 스크롤 컨테이너가 바뀌었거나, 부모 요소의 높이가 sticky 요소와 같아서 움직일 공간이 없을 때.

### 5. z-index 토큰과 모달을 `body` 아래에 두는 진짜 이유
**토큰화**
처음 답변이 정확하다. 실제 디자인 시스템에서도 레이어별로 값을 정해 둔다.

```typescript
// 예시: 레이어 순서가 이름에 드러나고, 사이에 끼워 넣을 여유를 둔다
export const zIndex = {
  base: 0,
  dropdown: 1000,
  sticky: 1100,
  overlay: 1200, // 모달 배경(dim)
  modal: 1300,
  popover: 1400,
  toast: 1500,
  tooltip: 1600,
} as const;
```
- 값 사이에 간격을 두는 이유: 나중에 새 레이어를 중간에 끼워 넣을 수 있도록.
- 토큰만으로는 부족하다. 1번처럼 **쌓임 맥락에 갇히면 토큰 값이 아무리 커도 소용없다.** 그래서 토큰과 portal(또는 top layer)을 **함께** 쓴다.

**모달을 `body` 바로 아래에 두는 이유**
처음 답변(모달 안의 드롭다운이 방해받지 않게)도 일리가 있지만, 핵심은 **모달 자신이 호출된 위치에 갇히지 않게 하는 것**이다. 모달은 깊은 곳(카드 안 버튼 등)에서 열리는 경우가 많은데, 그 자리에서 렌더링하면 이런 문제가 생긴다.
1. **조상의 쌓임 맥락에 갇힌다.** 조상 중 하나라도 `transform`, `opacity`, `z-index`를 가지면 모달 z-index가 소용없어진다. (이번 문제와 같은 원리)
2. **조상의 `overflow: hidden`에 잘린다.**
3. **조상의 `transform`이 `position: fixed`를 깨뜨린다.** (4번 함정 2)
4. **접근성 처리가 쉬워진다.** 모달이 앱 루트의 형제로 있으면, 모달이 열렸을 때 앱 루트 전체에 `inert`나 `aria-hidden`을 걸어서 배경을 읽거나 포커스하지 못하게 막기 쉽다. (10/3 ARIA 노트와 연결)

## 추가 심화 질문

### Q1. `isolation: isolate`는 언제 쓰나요?
다른 시각 효과 없이 **쌓임 맥락만** 만들고 싶을 때 쓴다. 예를 들어 컴포넌트 내부에서 `z-index: -1`로 배경 장식을 깔았는데, 그 장식이 컴포넌트 밖(페이지 배경 뒤)으로 빠져나가는 걸 막고 싶을 때 컴포넌트 루트에 `isolation: isolate`를 준다. 디자인 시스템 컴포넌트의 z-index가 바깥과 섞이지 않게 **캡슐화**하는 용도로 유용하다.

### Q2. z-index가 같거나 없으면 누가 위에 오나요?
같은 쌓임 맥락 안에서는 대략 이 순서로 위에 쌓인다.
1. 쌓임 맥락의 배경과 테두리
2. 음수 z-index
3. 일반 블록 요소
4. float 요소
5. 인라인 요소
6. `z-index: auto`/`0`인 positioned 요소
7. 양수 z-index

같은 단계라면 **HTML에서 나중에 나온 요소**가 위에 온다. 그래서 z-index 없이 `position: relative`만 줘도 positioned가 아닌 형제보다 위로 올라온다.

### Q3. 쌓임 맥락은 어떻게 디버깅하나요?
- Chrome DevTools의 **Layers** 패널에서 레이어 구조를 3D로 볼 수 있다.
- Elements 패널에서 의심되는 조상의 Computed 스타일 중 `transform`, `opacity`, `filter`, `z-index`를 확인한다.
- 조상을 하나씩 따라 올라가면서 표의 속성 중 하나라도 있는지 찾는 게 가장 빠르다.

## 참고
- [MDN: 쌓임 맥락 (Stacking context)](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_positioned_layout/Stacking_context)
- [MDN: z-index](https://developer.mozilla.org/en-US/docs/Web/CSS/z-index)
- [MDN: position](https://developer.mozilla.org/en-US/docs/Web/CSS/position)
- [MDN: isolation](https://developer.mozilla.org/en-US/docs/Web/CSS/isolation)
- [MDN: Popover API](https://developer.mozilla.org/en-US/docs/Web/API/Popover_API)
- [MDN: Top layer](https://developer.mozilla.org/en-US/docs/Glossary/Top_layer)
- [React 공식 문서: createPortal](https://react.dev/reference/react-dom/createPortal)
- [Floating UI](https://floating-ui.com/)
