# iOS Clipboard Copy 안되는 이슈

---

## 1. 현상

---

- iOS 에서 Clipboard 에 텍스트 Copy 가 되지 않는 현상 발생
- 서비스 중인 App 에서 특정 텍스트를 Copy 한 뒤 다른 App 에서 붙여넣기 기능이 안됨

## 2. 원인

---

- Copy 로직 흐름
    
    ```
    1. Html 버튼 클릭
    2. API 서버 호출
    3. 2번 항목에서 response 받은 Text 를 Copy 한다.
    ```
    

- 소스
    
    ```
    
    # 1. 버튼 이벤트 function
    
    function copy_recommend_code() {
    	
    	common_ajax('get', '/a/b', {}, (res) => {
    		if(res?.result === 'success') {
    			
    			const data = res?.a ?? {};
    			if(data.type !== '0') {
    				return;
    			}
    			copyClipboard(data.code);
    		}
    
    	}, true);
    
    }
    
    # 2. copy 소스 
    
    function copyClipboard(txt){
    	let t = document.createElement("textarea");
    	document.body.appendChild(t);
    	t.value = txt;
    	t.select();
    	document.execCommand('copy');
    	t.remove();
    }
    ```
    
    > `copyClipboard` 함수 내부 로직은 일반적으로 구현되는 방식.
    > 
    
    > `execCommand()`  함수의 경우 현재 `deprecated` 됨 <br>
    > https://developer.mozilla.org/en-US/docs/Web/API/Document/execCommand
    > 
    
    > `Clipboard API` 가 제공되지만 아직 제약이 많다 <br>
    > https://developer.mozilla.org/en-US/docs/Web/API/Clipboard_API
    > 

- 외부 라이브러리의 사용
    - `Web API` 문서를 보고 해당 API 를 활용하기 보다는 `Clipboard.js` 를 활용해 보았으나
    동일하게 작동하지 않았다.

- 결론
    - 위의 소스는 아직 까지는 문제가 없으나 Safari 에서 놓친 부분이 있던 것으로 확인
    - Safari 의 경우에는 사용자의 Action 에 의해 Trigger 된 것 만을 복사를 허용
    ( 아래 참고자료 확인)
    - 즉, 사용자가 버튼을 누르고 바로 copy 하는 것은 되지만 API 호출을 한 뒤에 copy 작업은
    허용 되지 않는다.

## 3. 해결

---

- Copy 로직 흐름을 아래와 같이 변경
    
    ```
    
    1. 화면 진입 시 Copy 되어야 하는 Text 조회 API 호출
       ( 별도의 HTMLElement 에 해당 값 저장해 놓음 )
    2. HTML 버튼 클릭
    3. 1번 항목의 Text 를 Copy 한다.
    ```
    

## 4. 참고 자료

---

[https://steadev.tistory.com/67](https://steadev.tistory.com/67)