# 모바일(IOS)에서 Back버튼 이슈

---

## 환경

---

- Flutter 1.0.0

## 문제상황

---

- IOS에서 Javascript 에서 history.back() 을 통하여 이전페이지로 돌아가는 기능 구현
- 이전페이지로 돌아간 이후에 APP 과 통신하는 메뉴를 클릭하면 함수를 중복으로 호출하는 
이슈 발견
- WebView 를 디버깅 해보면 이전페이지로 간 경우에 필수 Script 를 Load 하지 않는 내역 확인되었지만 실제 function 은 정상적으로 실행됨 ( referrence 오류 발생 안함 )
- 호출하는 특정 함수에 debug 를 해보았지만 잡히지 않음 ( breakpoint 에 안걸림 )

## 해결책

---

### WebView 의 History 를 사용하여 이전페이지 이동 적용

- 테스트를 하다보니 WebView 상에서 load 를 하면 정상적으로 작동하는 것을 확인
- 기존에 Javascript 를 통해 이전페이지로 돌아가던 방식을 Native 의 WebView 에서 이전페이지
로 이동하도록 적용

## 시도 했던 방법

---

### BFCache 부분을 수정

- 보통 브라우저에서 성능향상을 위해서 위 부분이 적용되어 있음
- history.back() 을 하는 경우 BFCache 이슈가 있어서 Cache 처리가 안되도록 해서 해봤지만
동일한 오류 발생