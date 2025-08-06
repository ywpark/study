# WebView에서 Native 함수 여부 체크 이슈

---

## 상황

---

1. WebView 에서 JavaScript 로 Native 에 함수가 존재하는지 확인해야 하는 경우가 발생하여
JavaScript 로 구현처리 함
2. Android 에는 문제가 없었지만 iOS에서 JavaScript 상에서 Exception 이 발생하는 것을 확인

## 처리 방법

---

- AS-IS
    
    ```jsx
    
    // 1. Android 인 경우
    
    function (funcName) {
        return typeof window["app"][funcName] === 'function';
    }
    
    // 2. iOS 인 경우
    /**
     * 함수가 없는 경우에 window["webkit"]["messageHandlers"][funcName] 이 부분이 
     * undefined 로 객체가 return 되고 그 뒤에 postMessage propperty 로 접근하면서
     * TypeError 가 JavaScript 상에서 발생
     */ 
    function (funcName) {
        return typeof window["webkit"]["messageHandlers"][funcName].postMessage === 'function';
    }
    ```
    

- TO-BE
    
    ```jsx
    
    // 1. Android 인 경우
    
    function (funcName) {
        return typeof window["app"][funcName] === 'function';
    }
    
    // 2. iOS 인 경우
    /**
     *
     * ?.(Optional Chaining) 연산자를 사용하여 undefined 여부에 따라 postMessage 
     * property 에 접근하도록 제어
     *
     * 함수가 존재하지 않는 경우에 return 되는 값은 "undefined" String 값으로 나옴 
     */ 
    function (funcName) {
        return typeof window["webkit"]["messageHandlers"][funcName]?.postMessage === 'function';
    }
    ```