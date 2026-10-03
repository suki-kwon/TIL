# 시맨틱 HTML과 ARIA

## 질문
웹 접근성(a11y) 관점에서 시맨틱 HTML이 왜 중요한가? `<div onClick={...}>`로 버튼을 만드는 것과 `<button>`을 쓰는 것의 차이는? ARIA가 무엇이고 어떤 상황에서 써야 하는가?

## 내 답변 (처음 생각)

HTML 요소는 브라우저에서 자동으로 의미가 해석되기 때문에 (굳이 우리가 ARIA를 지정하지 않아도) 시맨틱 HTML을 쓰는 게 좋다. 시맨틱 HTML 요소를 쓰면 자동으로 브라우저에서 의미가 해석되기 때문에 SEO에도 좋고 웹 접근성에도 좋다. 그리고 코드 가독성이나 유지보수 관점에서도 좋다.

## 보완된 개념

### 접근성 트리(Accessibility Tree)
"브라우저가 자동으로 의미를 해석한다"는 건 구체적으로, 브라우저가 DOM(과 CSS)을 바탕으로 **접근성 트리**를 만든다는 뜻이다. 스크린 리더 같은 보조기술은 DOM을 직접 읽는 게 아니라, OS의 접근성 API를 통해 이 접근성 트리를 읽는다. `<button>`을 쓰면 브라우저가 자동으로 이 트리에 "역할: button" 정보를 채워주지만, `<div onClick={...}>`는 "역할: generic(의미 없음)"으로 들어간다.

CSS도 접근성 트리에 영향을 준다. `display: none`이나 `visibility: hidden`인 요소는 접근성 트리에서도 빠진다 (아래 심화 질문 Q2).

### 시맨틱 HTML이 중요한 이유: 탐색 구조
스크린 리더 사용자는 페이지를 처음부터 끝까지 순서대로 듣지 않는다. 시각 사용자가 화면을 훑어보듯이, **heading이나 랜드마크 단위로 건너뛰며** 원하는 곳을 찾는다. 그래서 시맨틱 요소는 단순한 "의미 표시"가 아니라 페이지의 목차/내비게이션 역할을 한다.

- **랜드마크**: `<header>`(banner), `<nav>`(navigation), `<main>`(main), `<aside>`(complementary), `<footer>`(contentinfo) 등은 각각 랜드마크 role을 가진다. 스크린 리더에서 "메인 콘텐츠로 바로 이동" 같은 탐색이 가능해진다.
- **heading 계층**: `h1`~`h6`을 글자 크기가 아니라 **문서 구조** 기준으로 써야 한다. 스크린 리더는 heading 목록을 뽑아 목차처럼 보여주는데, 레벨이 뒤죽박죽이면 구조를 파악할 수 없다.
- **SEO**: 검색 엔진 크롤러도 같은 구조 정보(heading, `<main>`, `<article>` 등)를 활용해 콘텐츠의 핵심을 파악한다. 접근성과 SEO가 같이 좋아지는 이유가 이것이다.

```html
<!-- 모든 게 div면 스크린 리더 입장에선 구조가 없는 페이지 -->
<div class="header">...</div>
<div class="content">...</div>

<!-- 랜드마크로 구조가 드러남 -->
<header>...</header>
<nav aria-label="주 메뉴">...</nav>
<main>
  <h1>상품 상세</h1>
  <section>
    <h2>리뷰</h2>
  </section>
</main>
<footer>...</footer>
```

### div onClick vs button — 구체적 차이

| 기능 | `<button>` | `<div onClick>` |
|---|---|---|
| role="button" (스크린 리더가 "버튼"이라고 읽음) | 자동 | 없음 (`role="button"` 직접 추가해야) |
| 키보드 포커스 가능 (Tab으로 이동) | 자동 (`tabindex` 불필요) | 안 됨 (`tabindex="0"` 직접 추가해야) |
| Enter/Space로 클릭 실행 | 자동 | 안 됨 (`onKeyDown` 직접 구현해야) |
| `disabled` 상태 처리 | 자동 (포커스·클릭 모두 차단) | `aria-disabled` + 클릭 막기 직접 구현 |
| 폼 안에서 submit 동작 | 자동 (기본 `type="submit"`) | 전혀 없음 |

`<div onClick>`으로 버튼을 만들면, 마우스 사용자에게는 똑같이 보여도 키보드만 쓰는 사람, 스크린 리더 쓰는 사람한테는 존재하지 않는 요소와 같다. 이걸 수동으로 다 채우려면 결국 `<button>`이 공짜로 주는 기능을 JS로 재구현하는 셈이 된다.

> **주의**: `<button>`의 기본 `type`은 `submit`이다. 폼 안에 둔 "취소", "비밀번호 보기" 같은 버튼이 의도치 않게 폼을 제출하는 버그가 흔하므로, submit 용도가 아니면 `type="button"`을 명시하는 게 좋다.

### ARIA란

**ARIA = Accessible Rich Internet Applications**, W3C 웹 접근성 이니셔티브(WAI)의 스펙. HTML만으로는 표현 못 하는 복잡한 UI(탭, 콤보박스, 트리뷰 등)에 "이건 무슨 역할이고 지금 상태가 어떤지"를 보조기술에 알려주는 속성들의 집합이다.

세 가지로 분류된다.
- **Role (역할)**: 이 요소가 뭔지. `role="button"`, `role="dialog"`, `role="tablist"`
- **State (상태)**: 지금 상태, 자주 바뀜. `aria-expanded="true"`, `aria-checked="false"`, `aria-disabled="true"`
- **Property (속성)**: 구조적 관계나 부가 정보, 잘 안 바뀜. `aria-label`, `aria-labelledby`, `aria-describedby`, `aria-controls`

중요한 점: **ARIA는 시각적으로 아무것도 바꾸지 않는다.** 오직 접근성 트리에만 정보를 추가한다. `role="button"`을 넣어도 스타일은 그대로고, 클릭/키보드 동작은 직접 JS로 구현해야 한다. ARIA는 "이게 버튼이라고 약속"만 하는 것이지, 버튼처럼 동작하게 만들어주지는 않는다.

### 제1원칙: 가능하면 ARIA를 쓰지 마라

W3C 공식 문서([Using ARIA](https://www.w3.org/TR/using-aria/))에 5가지 규칙이 있고, 1번이 이것이다.

> **제1원칙**: 필요한 의미(semantics)와 동작(behavior)을 이미 갖춘 네이티브 HTML 요소나 속성이 있다면, 요소를 억지로 재활용하고 ARIA를 덧붙이는 대신 그냥 그 네이티브 요소를 써라.

이유: ARIA는 "약속"이지 "구현"이 아니기 때문이다. `<div role="button">`을 쓰면 스크린 리더한테는 버튼이라고 말해놓고, 정작 키보드 포커스/Enter 동작이 없으면 사용자가 "버튼이라는데 눌러도 반응이 없네?" 하고 더 혼란스러워진다. 그래서 W3C의 [ARIA Authoring Practices Guide(APG)](https://www.w3.org/WAI/ARIA/apg/)는 **"No ARIA is better than Bad ARIA"**(잘못된 ARIA는 아예 없는 것보다 나쁘다)라고 강조한다.

나머지 4개 규칙:
2. 네이티브 요소의 기본 의미를 ARIA로 함부로 바꾸지 마라 (예: `<h1 role="button">`)
3. ARIA로 만든 인터랙티브 컨트롤은 반드시 키보드로도 조작 가능해야 한다
4. 포커스 가능한 요소에 `role="presentation"`이나 `aria-hidden="true"`를 쓰지 마라 (키보드 포커스는 그 요소로 가는데 스크린 리더는 아무것도 읽지 않아서, 사용자는 "빈 곳"에 포커스가 간 것처럼 느낀다)
5. 모든 인터랙티브 요소는 접근 가능한 이름(accessible name)이 있어야 한다

### 먼저 네이티브 요소가 있는지 확인하기
예전엔 ARIA로 직접 만들어야 했던 위젯들 중 상당수가 이제 네이티브로 제공된다. 제1원칙에 따라 이것부터 검토한다.

| 위젯 | 네이티브 요소 | 공짜로 얻는 것 |
|---|---|---|
| 모달 | `<dialog>` + `showModal()` | `role="dialog"`, 포커스 가두기, 배경 inert 처리, Esc로 닫기, `::backdrop` |
| 아코디언 | `<details>` / `<summary>` | 펼침/접힘 상태 전달, 키보드 토글 |
| 팝오버/툴팁류 | `popover` 속성 | 바깥 클릭/Esc로 닫기, top layer 렌더링 |

```html
<dialog id="confirm">
  <h2>삭제할까요?</h2>
  <form method="dialog">
    <button value="cancel">취소</button>
    <button value="ok">삭제</button>
  </form>
</dialog>
<button type="button" onclick="confirm.showModal()">삭제</button>

<details>
  <summary>자주 묻는 질문 1</summary>
  <p>답변 내용</p>
</details>
```

다만 `<details>`는 스타일링 제약이 있고, 디자인 요구사항 때문에 커스텀 구현을 택하는 경우도 있다. 그럴 땐 아래처럼 ARIA로 상태를 직접 알려줘야 한다.

### ARIA는 언제 쓰나

#### 1. HTML에 대응 요소가 없는 복잡한 위젯
탭, 콤보박스(자동완성 검색), 트리뷰, 커스텀 셀렉트 등은 HTML 자체에 대응 요소가 없어서 ARIA 없이는 접근성을 만들 수 없다.

탭은 역할(role)뿐 아니라 **키보드 패턴까지** 구현해야 한다. APG 기준으로 탭 목록 안에서는 Tab 키가 아니라 **←/→ 화살표**로 탭을 이동하고, Tab 키는 탭 목록에서 패널로 빠져나가는 데 쓴다. 이를 위해 선택된 탭만 `tabindex="0"`, 나머지는 `tabindex="-1"`로 두는 **roving tabindex** 기법을 쓴다.

```html
<div role="tablist" aria-label="상품 정보">
  <button role="tab" id="tab-1" aria-selected="true" aria-controls="panel-1" tabindex="0">상세</button>
  <button role="tab" id="tab-2" aria-selected="false" aria-controls="panel-2" tabindex="-1">리뷰</button>
</div>
<div role="tabpanel" id="panel-1" aria-labelledby="tab-1">상세 내용</div>
<div role="tabpanel" id="panel-2" aria-labelledby="tab-2" hidden>리뷰 내용</div>
```

```js
// ←/→로 탭 이동 + roving tabindex
tablist.addEventListener('keydown', (e) => {
  if (e.key !== 'ArrowRight' && e.key !== 'ArrowLeft') return;
  const tabs = [...tablist.querySelectorAll('[role="tab"]')];
  const current = tabs.indexOf(document.activeElement);
  const next = (current + (e.key === 'ArrowRight' ? 1 : -1) + tabs.length) % tabs.length;
  selectTab(tabs[next]); // aria-selected, tabindex, 패널 hidden 갱신
  tabs[next].focus();
});
```

#### 2. 동적으로 바뀌는 상태를 알려줄 때
커스텀 아코디언이 열렸는지 닫혔는지는 시각적으로는 보이지만, 스크린 리더에게는 안 보인다. 토글할 때 `aria-expanded`와 `hidden`을 **둘 다** 갱신해야 한다.

```html
<button type="button" aria-expanded="false" aria-controls="faq-1">자주 묻는 질문 1</button>
<div id="faq-1" hidden>답변 내용</div>
```

#### 3. 동적 콘텐츠를 실시간으로 알려줄 때 (aria-live)
토스트 메시지, 장바구니 담김 알림처럼 포커스 이동 없이 화면이 바뀔 때, 스크린 리더가 자동으로 읽어주게 한다.

**실무에서 자주 틀리는 포인트**: live region은 **내용이 바뀌기 전에 이미 DOM에 존재**해야 한다. 스크린 리더는 "이미 등록된 live region의 내용 변화"를 감지하는 방식이라, 텍스트가 담긴 `<div aria-live>`를 통째로 새로 렌더링하면 대부분 읽히지 않는다.

```html
<!-- 페이지 로드 시점부터 빈 컨테이너로 존재 -->
<div role="status" aria-live="polite"></div>
```
```js
// 이후 텍스트만 바꿔 넣으면 읽힘
statusEl.textContent = '장바구니에 상품이 담겼습니다';
```

- `role="status"`: 암묵적으로 `aria-live="polite"` (현재 읽던 내용이 끝난 뒤 알림)
- `role="alert"`: 암묵적으로 `aria-live="assertive"` (즉시 끼어들어 알림, 에러처럼 급한 경우에만)

#### 4. 시각적 레이블이 불충분하거나 없을 때
```html
<!-- 아이콘만 있는 버튼: 버튼에 이름을 주고, 안의 아이콘은 숨김 -->
<button type="button" aria-label="메뉴 닫기">
  <svg aria-hidden="true">...</svg>
</button>

<!-- 장식용 아이콘도 숨김 -->
<svg aria-hidden="true">...</svg>
```

### 디자인 시스템 관점에서의 정리
버튼/인풋/체크박스/모달(`<dialog>`)처럼 네이티브로 가능한 건 네이티브 요소로 만들고, 탭이나 콤보박스처럼 네이티브에 없는 컴포넌트에만 ARIA + 키보드 핸들링을 직접 구현하는 것이 원칙이다. Radix, React Aria 같은 라이브러리들이 이런 "접근 가능한 커스텀 위젯"을 미리 만들어 제공하는 이유도 여기에 있다. 키보드 패턴(roving tabindex, 포커스 관리 등)까지 전부 직접 만들기는 비용이 크기 때문이다.

## 추가 심화 질문

### Q1. `aria-label`과 `aria-labelledby`는 어떻게 다르고, 접근 가능한 이름(accessible name)은 어떤 우선순위로 정해지나?
- `aria-label`: 문자열을 직접 이름으로 지정. 화면에 보이지 않는 텍스트.
- `aria-labelledby`: 화면에 **이미 보이는 다른 요소의 id**를 참조해서 그 텍스트를 이름으로 씀. 여러 id를 공백으로 나열해 이어 붙일 수도 있다.

접근 가능한 이름은 대략 아래 우선순위로 계산된다 ([Accessible Name and Description Computation](https://www.w3.org/TR/accname-1.2/)).
1. `aria-labelledby`
2. `aria-label`
3. 네이티브 레이블 (`<label for>`, `<img alt>`, `<caption>` 등)
4. 요소의 텍스트 콘텐츠 (button, link 등 콘텐츠로 이름을 갖는 role)
5. `title` 속성 (최후의 수단)

그래서 `<button aria-label="닫기">X</button>`이면 스크린 리더는 "X"가 아니라 "닫기"로 읽는다. 화면에 보이는 텍스트와 읽히는 이름이 다르면 음성 명령("닫기 클릭") 사용자가 혼란스러울 수 있으니, 가능하면 **보이는 텍스트를 이름으로 쓰는 쪽**(`aria-labelledby` 또는 텍스트 콘텐츠)이 낫다.

### Q2. `display: none`, `visibility: hidden`, `aria-hidden="true"`, `.sr-only`는 접근성 트리에서 어떻게 다른가?

| 방법 | 화면에 보임 | 스크린 리더가 읽음 | 공간 차지 |
|---|---|---|---|
| `display: none` | X | X | X |
| `visibility: hidden` | X | X | O |
| `aria-hidden="true"` | O | X | O |
| `.sr-only` (시각적으로만 숨김) | X | O | X (1px로 잘라냄) |

```css
/* 스크린 리더 전용 텍스트: 화면에선 안 보이지만 접근성 트리엔 남음 */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

`aria-hidden="true"`는 자식 요소까지 전부 접근성 트리에서 빼버리므로, 안에 포커스 가능한 요소가 있으면 규칙 4 위반이 된다. 모달이 열렸을 때 배경을 숨기는 용도로는 `aria-hidden`보다 `inert` 속성이 낫다 (포커스와 클릭까지 함께 막아줌).

### Q3. `disabled`와 `aria-disabled="true"`는 어떻게 다른가?
- `disabled`: 포커스 자체가 안 되고(Tab 순서에서 빠짐), 클릭 이벤트도 막힌다. 스크린 리더 사용자는 Tab으로 탐색할 때 그 버튼이 있다는 것조차 모를 수 있다.
- `aria-disabled="true"`: 비활성 상태라고 **알려만** 주고, 포커스는 그대로 가능하다. 클릭 차단은 JS로 직접 해야 한다.

그래서 "왜 비활성인지" 툴팁으로 설명해야 하는 버튼(예: "필수 항목을 입력하세요")은 `aria-disabled`로 포커스를 유지하는 쪽이 사용자에게 더 친절하다. 포커스가 안 되면 그 설명에 도달할 방법이 없기 때문이다.

## 참고
- [W3C: Using ARIA](https://www.w3.org/TR/using-aria/)
- [W3C: ARIA Authoring Practices Guide (APG)](https://www.w3.org/WAI/ARIA/apg/)
- [W3C APG: Tabs Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/tabs/)
- [MDN: ARIA](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA)
- [MDN: `<dialog>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/dialog)
- [MDN: `<details>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/details)
- [MDN: aria-expanded](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Attributes/aria-expanded)
- [MDN: aria-live](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Attributes/aria-live)
- [MDN: aria-label](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Attributes/aria-label)
