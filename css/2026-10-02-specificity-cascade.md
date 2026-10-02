# CSS Specificity와 Cascade

## 질문
CSS의 우선순위(Specificity)와 캐스케이딩(Cascade)이 무엇인지 설명하라. 디자인 시스템을 만들 때 컴포넌트 스타일끼리 충돌하지 않게 하는 방법들(BEM, CSS Modules, CSS-in-JS, `:where()` 등)과, 각 방법이 specificity 문제를 어떻게 다루는지도 설명하라.

## 내 답변 (처음 생각)

css 우선순위란 같은 요소에 스타일이 중복으로 적용되어 있을 때 어떤 것을 먼저 적용할지에 대한 규칙.
- `!important`가 가장 우선순위 높음
- 그 다음은 인라인 스타일(`style=""`)
- 그 다음은 id, class 순서
- 그리고 부모한테 상속받은 속성
- 같은 class 스타일이라도 뎁스가 더 깊은 class명으로 지정한 스타일이 더 우선순위 높음
- 이 순위들이 캐스케이딩 규칙이라고 생각함

BEM / CSS Modules / CSS-in-JS:
- BEM: 네이밍 컨벤션을 엄격하게 지켜서 유니크하게 class명을 지정하는 방식
- CSS Modules: 빌드할 때 클래스 이름에 고유한 해시 문자열을 자동으로 생성해주는 방식. 예전에 많이 씀
- CSS-in-JS: 객체 내부에 스타일을 정의하는 방식 (styled-components, emotion). 런타임이나 빌드 시에 클래스명을 자동 생성해줌

## 보완된 개념

### Cascade와 Specificity는 다른 층위의 개념
"!important > 인라인 > id > class" 순서는 하나의 규칙이 아니라 **캐스케이드의 서로 다른 단계들이 섞여 있는 것**이다. `!important`는 출처·중요도 단계, 인라인 스타일은 별도 단계, id > class만 Specificity 단계에 해당한다. 전체 캐스케이드 알고리즘은 아래 순서로 단계적으로 비교하며, 앞 단계에서 승부가 나면 거기서 끝난다.

1. **출처(Origin) + 중요도(Importance)**
   - 일반 선언: 브라우저 기본 스타일 < 사용자 스타일 < 작성자 스타일
   - `!important` 선언은 위 순서가 **거꾸로 뒤집혀서** 일반 선언보다 위에 쌓인다: 작성자 `!important` < 사용자 `!important` < 브라우저 기본 `!important`
   - 그래서 전체에서 가장 강한 건 브라우저 기본 스타일의 `!important`다. (사용자가 접근성 때문에 지정한 `!important`를 사이트가 덮어쓰지 못하게 하려는 설계)
2. **컨텍스트(Context)**: Shadow DOM 안팎의 스타일이 충돌할 때 비교
3. **인라인 스타일(Element-attached styles)**: `style=""`로 직접 붙인 선언
4. **Cascade Layers (`@layer`)**: 어느 레이어에 속했는지 비교 (아래 심화 질문 Q1)
5. **Specificity**: 여기까지 동점이면 선택자의 명시도를 비교
6. **소스 순서(Order of appearance)**: 그것도 동점이면 나중에 작성/로드된 규칙이 이김

즉 Specificity는 캐스케이드 알고리즘 안에 들어있는 비교 기준 중 하나일 뿐이다. "뎁스가 깊은 클래스명이 우선순위 높다"는 설명은 정확히 이 Specificity 단계 얘기다.

### 상속(Inheritance)은 별개의 메커니즘
캐스케이드는 "충돌하는 값들 중 뭘 쓸지" 정하는 것이고, 상속은 "애초에 값이 지정 안 된 속성에 뭘 채울지" 정하는 것이다 (상속 속성이면 부모의 계산값을, 아니면 초기값을 씀). 캐스케이드 결과로 값이 이미 정해지면 상속은 끼어들지 않는다.

**상속받은 값은 Specificity가 아예 없다.** 부모에 아무리 강한 선택자로 지정해도, 자식에게 직접 매칭되는 규칙이 하나라도 있으면 그게 이긴다.

```css
#app { color: red; }  /* ID 선택자지만 자식에게는 "상속"으로만 전달됨 */
* { color: blue; }    /* 명시도 0이지만 자식에게 "직접" 매칭됨 */
```
```html
<div id="app">
  <p>파란색</p> <!-- 상속값(red)보다 직접 매칭된 * (blue)가 이김 -->
</div>
```

상속/캐스케이드를 직접 제어하는 키워드도 있다.
- `inherit`: 부모의 계산값을 강제로 상속 (상속 안 되는 속성도)
- `initial`: CSS 명세상의 초기값으로 (예: `display`는 `inline`)
- `unset`: 상속 속성이면 `inherit`, 아니면 `initial`처럼 동작
- `revert`: 브라우저 기본 스타일 값으로 되돌림 (예: `display: revert`는 `div`면 `block`)
- `revert-layer`: 이전 cascade layer의 값으로 되돌림

### Specificity의 정확한 구조
ID / 클래스·속성·가상클래스 / 태그·가상요소, 이렇게 3단계로 나뉘고 보통 `(ID, 클래스, 태그)` 형태로 `(0,1,0)`처럼 표기한다. 인라인 스타일은 예전 자료에서 `(1,0,0,0)`처럼 맨 앞 자리로 설명하기도 하지만, 현재 명세에서는 Specificity가 아니라 그보다 앞선 별도의 캐스케이드 단계다. 이 3단계는 순위가 아니라 **서로 섞이지 않는 별개의 자릿수**다. 클래스 100개를 합쳐도 ID 1개를 절대 못 이긴다 (10진수처럼 자릿수가 밀리는 게 아니라, 비교 자체가 자릿수별로 따로 이루어짐).

"뎁스가 깊다"보다는 **"선택자 안에 클래스가 몇 개 결합되어 있는가"**가 더 정확한 표현이다. `.card .title`은 실제 DOM 깊이와 무관하게 클래스 2개가 결합된 선택자라서 명시도가 더 높은 것이다.

## BEM / CSS Modules / CSS-in-JS 디테일

셋 다 "specificity 계산 규칙을 바꾸는 것"이 아니라, **specificity를 낮고 평평하게 유지해서 충돌 자체를 줄이는 전략**이라는 공통점이 있다.

### BEM
`.block__element--modifier` 형태로, 모든 선택자를 클래스 하나짜리로만 쓰도록 강제하는 네이밍 컨벤션. 중첩 선택자(`.card .title`)나 ID 선택자를 쓰지 않으니 모든 규칙의 specificity가 항상 동일(0,1,0)해진다. 충돌이 나면 specificity 비교 없이 소스 순서로만 결정되어 결과가 예측 가능해진다. 단점은 빌드 도구의 강제력이 없어 사람이 규칙을 안 지키면 그대로 깨진다는 것.

### CSS Modules
빌드 타임에 해시를 붙여 클래스명 자체를 전역에서 고유하게 만든다. 이건 specificity 문제가 아니라 **네임스페이스(전역 이름 충돌) 문제**를 푸는 것이다 — 서로 다른 컴포넌트가 둘 다 `.title`을 써도 빌드 후엔 `Card_title_a1b2`, `Modal_title_c3d4`처럼 이름 자체가 달라 애초에 충돌하지 않는다. 보통 클래스 선택자 하나만 쓰므로 specificity도 BEM처럼 낮고 평평하게 유지되는 게 일반적이다.

### CSS-in-JS (styled-components, emotion)
런타임 또는 빌드 시점에 고유 클래스명을 자동 생성한다.

소스 순서 문제를 **완전히** 해결해주는 건 아니다. 스타일은 컴포넌트가 렌더링될 때 `<style>` 태그에 주입되는데, 코드 스플리팅으로 어떤 컴포넌트가 늦게 로드되면 주입 순서가 바뀌어서 덮어쓰기 결과가 달라질 수 있다. 순서 문제를 실제로 피하게 해주는 장치는 **스타일 합성(composition)**이다. 예를 들어 emotion은 `css={[baseStyle, overrideStyle]}`처럼 여러 스타일을 넘기면 **하나의 클래스로 합쳐서** 만들기 때문에, 선언 순서가 배열 순서대로 보장되고 클래스끼리 경쟁할 일이 없다.

최근엔 런타임 CSS-in-JS를 피하는 흐름이 있다.
- 렌더링할 때마다 스타일을 계산하고 주입하는 **런타임 비용**
- **React Server Components와 잘 맞지 않음**: 런타임 방식은 Context와 클라이언트 렌더링에 의존하는데, 서버 컴포넌트에서는 이를 쓸 수 없다

그래서 vanilla-extract, Linaria, Panda CSS 같은 **빌드 타임 추출 방식(zero-runtime CSS-in-JS)**이나 Tailwind 같은 유틸리티 CSS로 옮겨가는 경우가 많다.

## 새로 배운 개념: `:where()`

`:is()`처럼 여러 선택자를 묶어주지만, **specificity가 항상 0**이라는 차이가 있다.

```css
/* 일반적인 방식 - specificity (0,0,1,0) */
.button { color: blue; }

/* :where()로 감싸면 - specificity 0 */
:where(.button) { color: blue; }
```

디자인 시스템 라이브러리를 만들 때, 기본(base) 스타일을 `:where()`로 감싸두면 라이브러리를 쓰는 사람이 클래스 하나만 덮어써도 손쉽게 오버라이드할 수 있다. 안 그러면 사용자가 `!important`를 쓰거나 더 긴 선택자를 억지로 만들어야 하는 "specificity 전쟁"이 생기는데, `:where()`는 이를 설계 단계에서 막아준다. 예를 들어 `@tailwindcss/typography` 플러그인의 `prose` 스타일은 `.prose :where(p)`처럼 내부 요소 선택자를 `:where()`로 감싸서, 사용자가 유틸리티 클래스 하나로 쉽게 덮어쓸 수 있게 만든다. CSS reset을 만들 때도 기본 스타일을 `:where()`로 감싸는 패턴이 자주 쓰인다.

## 추가 심화 질문

### Q1. Cascade Layers(`@layer`)는 무엇이고, `:where()`와는 어떻게 다른가요?
`@layer`는 스타일을 **레이어 단위로 묶어서 우선순위를 명시적으로 정하는** 기능이다. 레이어 비교는 Specificity보다 **먼저** 일어나므로, 뒤 레이어의 클래스 하나가 앞 레이어의 ID 선택자도 이긴다.

```css
/* 레이어 순서를 먼저 선언 (뒤로 갈수록 우선순위 높음) */
@layer reset, base, components, utilities;

@layer base {
  #header .title { color: black; }  /* (1,1,0)지만 base 레이어 */
}

@layer utilities {
  .text-red { color: red; }         /* (0,1,0)이지만 utilities 레이어 → 이김 */
}
```

규칙 세 가지:
- 레이어 순서는 처음 선언된 순서를 따르고, **나중 레이어가 이긴다**
- **레이어에 속하지 않은 스타일(unlayered)이 모든 레이어를 이긴다** → 외부 라이브러리를 레이어에 넣어두면, 내 코드는 별도 작업 없이 라이브러리 스타일을 덮어쓸 수 있다
- **`!important`를 쓰면 레이어 순서가 거꾸로 뒤집힌다** → 앞 레이어(`reset`)의 `!important`가 가장 강하다. 출처 순서가 `!important`에서 뒤집히는 것과 같은 원리로, 기반 레이어가 "절대 바뀌면 안 되는 값"을 보호할 수 있게 하려는 설계

```css
/* 외부 라이브러리를 낮은 레이어로 가져오기 */
@import url('library.css') layer(library);
```

**`:where()`와 비교**
| | `:where()` | `@layer` |
|---|---|---|
| 단위 | 선택자 하나 | 파일, 라이브러리, 코드 블록 |
| 방식 | 명시도를 0으로 만듦 | 명시도 비교 전에 레이어 순서로 결정 |
| 용도 | 컴포넌트 기본 스타일을 덮어쓰기 쉽게 | reset → base → components → utilities처럼 전체 구조 설계 |

둘은 같이 쓸 수 있다. Tailwind v4도 내부적으로 `@layer theme, base, components, utilities`를 사용한다.

### Q2. `:is()`, `:not()`, `:has()`의 명시도는 어떻게 계산되나요?
`:where()`와 문법은 비슷하지만, 이 셋은 **괄호 안 선택자 중 가장 높은 명시도**를 가져간다. 실제로 어떤 선택자에 매칭됐는지와 상관없다.

```css
:is(#main, .content) p { color: red; }
/* .content 안의 p에만 매칭돼도 명시도는 #main 기준 (1,0,1) */

:where(#main, .content) p { color: blue; }
/* 명시도 (0,0,1) — :where() 부분은 0 */

.button:not(.primary) { }   /* (0,2,0) — :not 자체는 0, 괄호 안 .primary가 더해짐 */
.card:has(> img) { }        /* (0,1,1) — 괄호 안 img가 더해짐 */
```

그래서 `:is()`에 ID를 섞어 쓰면 의도치 않게 명시도가 확 올라가서 덮어쓰기 어려워질 수 있다. 명시도를 올리고 싶지 않을 때는 `:where()`를 쓴다.

### Q3. 상속값은 명시도가 몇인가요?
명시도가 **아예 없다** (0보다도 약하다). 그래서 `* { }` 같은 명시도 0 선택자도 직접 매칭되면 부모가 ID로 지정한 상속값을 이긴다. 위 "상속은 별개의 메커니즘" 섹션의 예시 참고. 부모의 값을 따라가게 하고 싶다면 자식에 `color: inherit`을 명시하면 된다.

### Q4. Shadow DOM이나 `@scope`로 스타일을 격리하는 방법은?
- **Shadow DOM**: Web Components에서 쓰는 방식. Shadow root 안의 스타일은 바깥으로 새어나가지 않고, 바깥 스타일도 안으로 들어오지 않는다 (단, `color`나 `font` 같은 **상속 속성은 경계를 넘어 상속된다**). 바깥에서 내부를 꾸미게 하려면 CSS 변수(custom properties)나 `::part()`를 열어둔다. 캐스케이드의 "컨텍스트" 단계가 이 경우를 다룬다.
- **`@scope`**: 특정 DOM 영역 안에서만 적용되는 스타일을 만드는 CSS 기능. 클래스명 해시(CSS Modules) 없이도 범위를 제한할 수 있다.

```css
@scope (.card) to (.card-content) {
  img { border-radius: 8px; }  /* .card 안, .card-content 바깥의 img에만 적용 */
}
```
`to (...)`로 "여기서부터는 적용 안 함"이라는 하한(donut scope)도 정할 수 있다. 비교적 최근 기능이라 실무 도입 전에 브라우저 지원 범위를 확인하는 게 좋다.

## 참고
- [MDN: Cascade](https://developer.mozilla.org/en-US/docs/Web/CSS/Cascade)
- [MDN: Specificity](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_cascade/Specificity)
- [MDN: Inheritance](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_cascade/Inheritance)
- [MDN: :where()](https://developer.mozilla.org/en-US/docs/Web/CSS/:where)
- [MDN: @layer](https://developer.mozilla.org/en-US/docs/Web/CSS/@layer)
- [MDN: :is()](https://developer.mozilla.org/en-US/docs/Web/CSS/:is)
- [MDN: @scope](https://developer.mozilla.org/en-US/docs/Web/CSS/@scope)
