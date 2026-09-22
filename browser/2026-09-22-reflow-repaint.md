# 리플로우(Reflow) vs 리페인트(Repaint)

## 질문
브라우저에서 리플로우와 리페인트의 차이는 무엇인가? 이 둘을 최소화하기 위한 실무 방법은?

## 내 답변 (처음 생각)
- 리플로우를 '리렌더링'과 비슷한 개념으로 생각함
- 리페인트는 리플로우 없이 픽셀만 다시 그리는 것
- 최소화 방법으로 컴포넌트 단위 리렌더링 최적화(React memo 등)를 떠올림
- → React 관점 최적화와 브라우저 렌더링 파이프라인 개념을 혼동하고 있었음

## 보완된 개념

**리플로우(Reflow)**
- 요소의 크기·위치 등 레이아웃(geometry)이 바뀔 때, 브라우저가 레이아웃을 다시 계산하는 과정
- 트리거 예: width/height, margin, display 변경, DOM 추가/삭제, offsetHeight 등 레이아웃 값 읽기

**리페인트(Repaint)**
- 레이아웃은 그대로, 색상·배경·그림자 같은 시각 스타일만 바뀔 때 픽셀을 다시 그리는 과정
- 트리거 예: color, background-color, visibility 변경

**중요한 구분**: 리플로우/리페인트는 브라우저 렌더링 파이프라인 개념이고, React의 "리렌더링"은 Virtual DOM 비교 후 실제 DOM을 업데이트하는 과정 — 그 결과로 브라우저가 리플로우/리페인트를 수행하게 되는 흐름. 서로 다른 층위임.

## 최소화 방법
- `transform`, `opacity`만 애니메이션에 사용 (레이아웃 안 건드리고 GPU 합성 레이어 처리)
- DOM 읽기/쓰기 분리해서 배치 처리 (레이아웃 스래싱 방지)
- 스타일 변경은 클래스 토글로 한 번에 처리
- 긴 리스트는 DocumentFragment나 가상화(virtualization) 활용
- `will-change`, `contain` CSS 속성으로 브라우저에 힌트 제공

## 참고
- [MDN Glossary: Reflow (한국어)](https://developer.mozilla.org/ko/docs/Glossary/Reflow)
- [MDN Glossary: Repaint (영문)](https://developer.mozilla.org/en-US/docs/Glossary/Repaint)
