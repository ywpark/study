# Spotless ??

---

> 💡 코드의 오류나 버그가 있는지 확인하고, 정해진 규칙을 잘 지키고 있는지에 대한 것들을 개발하면서 확인 및 점검을 하기 위해 사용하는 도구
> 린트(Lint) 혹은 린터(Linter) 라고도 불린다.
>
> Spotless 는 위의 규칙(Code Formatter) 을 체크하는 도구 중 하나로 생각된다.


> 💡 JavaScript  에 대한 부분은 제외하였다. 
> (관련해서는 ESLint 로 자료를 찾아보면 많이 볼 수 있다)
> 
> 참고 자료) 
> SpringBoot 에서 Javascript ESLint 적용 하기 <br>
> [https://velog.io/@hellohyeon/Spring-BootBabel-Prettier-ESLintAirbnb-세팅하기](https://velog.io/@hellohyeon/Spring-BootBabel-Prettier-ESLintAirbnb-%EC%84%B8%ED%8C%85%ED%95%98%EA%B8%B0)

## Spotless ?

---

- [GitHub 주소](https://github.com/diffplug/spotless)
- Keep your code spotless
    - 번역해보면 [ 너의 코드를 흠 없이 유지하라 ] 이 정도일거 같다
- 개발자가 늘어날 수록 코드 컨벤션이 쉽게 지켜지지 않고 문서만 가지고도 쉽지 않다.
    - 변수, 함수 등 네이밍 규칙처럼 개발자 스스로가 지켜야하는 것들을 말하는 것이 아님
    (이런 것들은 코드리뷰를 통해서 검증 절차가 없는 이상 힘들지도)
- 해당 플러그인을 활용하여 build 시 코드 컨벤션이 지켜지지 않은경우 build가 실패함

## 배경 및 목적

---

- 프로젝트 인원이 많아져도 최소한의 코드 컨벤션을 유지하도록 하자
- Java 프로젝트가 대다수 이며, 보통 Google java 포맷을 활용하기에 적합
- Space, Tab 등으로 인한 불필요한 커밋 및 diff 가 발생하지 않도록 유지
- 나만 지키는 것은 의미가 없고, 협업하는 개발자들이 최대한 지킬 수 있도록 해야함

## 환경

---

- Windows 11
- openjdk 11
- gradle 6.8.2
- SpringBoot
- intelliJ

## Gradle 설정

---

- spotless 는 gradle 플러그인으로 설치

```groovy
# build.gradle

plugins {
	id('com.diffplug.spotless') version('6.25.0')
}
```

- spotless 를 통해 체크할 규칙 정의
    - 아래 내용이 일반적인 부분이고 좀 더 디테일한 설정은 가이드 문서를 참고해야 한다.

```groovy
# build.gradle

spotless {
    java {
		  
     // googleJavaFormat().aosp()

     // import 순서 정의
     importOrder(
        'java',
        'javax',
        'lombok',
        'org.springframework',
        'com.msit',
        '',
        '\\#com.msit',
        '\\#'
    )
    
    // 사용하지 않는 import 삭제
    removeUnusedImports()
    
    // 구글 Java 코드 스타일을 따른다
    // 별도로 java-google-style.xml 파일을 등록 안해도 된다.
    /*
        googleJavaFormat() : 탭은 2개의 공백
        googleJavaFormat().aosp() : 탭은 4개의 공백
        [참고] https://github.com/google/google-java-format/issues/525
        https://velog.io/@cekim/briefing-spotless
    */
    googleJavaFormat().aosp()

    // 라인 끝에 공백을 제거
    trimTrailingWhitespace()

    // 파일 끝에 새로운 라인 추가
    endWithNewline()

	}
}
```

> 💡 googleJavaFormat() 을 통해서 google 의 java Code Style 을 적용할 경우 importOrder()
> 의 사용 순서를 주의해야 한다. 
> 
> 개발자가 정의한 Order 순서가 적용되지 않고 Google import 순서가 적용된다.
> 그렇기에 상황에 맞게 선택을 해야 한다.
> 
> importOrder() 에서 정의한 규칙으로 최종 적용하고자 한다면
> googleJavaFormat() → importOrder() 순서로 작업을 진행하게 하면 된다.

- spotless 기능 실행

```groovy
./gradlew spotlessCheck : 정의된 규칙이 적용되어있는지 체크

./gradlew spotlessApply : 정의한 규칙을 소스코드에 반영
```

> 💡 spotlessApply : 프로젝트의 모든 코드에 대해 지정된 코드 스타일을 적용
> spotlessJavaApply : Java 코드에만 코드 스타일을 적용

## 개선 방향

---

- ~~IntelliJ 에서 Commit 작업 시 spotless 자동 적용하기~~
    - ~~[git commit 시 특정 task 실행하기](https://newcubator.com/blog/web/run-gradle-task-before-commit-in-intellij)~~
    - ~~해당 Task 기본 설정 후에 작업이 적용되지 않은 경우라면 spotlessApply Task 진행 중에 오류가 발생되었을 가능성이 있으니 직접 spotlessApply 를 실행해서 확인을 해봐야 한다.~~
    ~~( 2월21일 보여준 예제에서 적용이 안된 사유는 import 구문에 주석 처리되어 있는 부분이 존재하여서 발생됨 )~~
- Git Hook을 활용한 자동화 적용
    - [Got Hooks 설명](https://www.atlassian.com/ko/git/tutorials/git-hooks)
    - [예제](https://wnwngus.tistory.com/71)
    
    ```bash
    # pre-commit
    
    #!/bin/sh
    
    stagedFiles=$(git diff --staged --name-only)
    
    ./gradlew spotlessApply
    
    for file in $stagedFiles; do
      if test -f "$file"; then
        git add "$file"
      fi
    done
    ```
    
- EditConfig 파일을 통한 다양한 Editor , IDE , OS 에서 동일한 코드 스타일을 추구
    - [EditorConfig 공식사이트](https://editorconfig.org/)
    - 자신이 사용하는 IDE가 해당 파일을 지원하는지 위의 공식 사이트에서 확인 필요

## 참고자료

---

- [https://ko.wikipedia.org/wiki/린트_(소프트웨어)](https://ko.wikipedia.org/wiki/%EB%A6%B0%ED%8A%B8_(%EC%86%8C%ED%94%84%ED%8A%B8%EC%9B%A8%EC%96%B4))
- [https://melonicedlatte.com/2022/06/11/235800.html](https://melonicedlatte.com/2022/06/11/235800.html)
- [git 에서 CRLF 개행 문자 차이로 인한 문제 해결하기 (lesstif.com)](https://www.lesstif.com/gitbook/git-crlf-20776404.html)
    - Windows 만 사용하기에 딱히 문제는 없을 것으로 보임
- javaScript 스타일 가이드
    - [https://github.com/naver/eslint-config-naver/blob/master/STYLE_GUIDE.md](https://github.com/naver/eslint-config-naver/blob/master/STYLE_GUIDE.md) 
    ( 네이버, 현재는 운영 안함 )
    - [GitHub - parksb/javascript-style-guide: Airbnb JavaScript 스타일 가이드](https://github.com/ParkSB/javascript-style-guide?tab=readme-ov-file) ( Airbnb 가이드 )
- [https://github.com/diffplug/spotless/blob/main/plugin-gradle/README.md](https://github.com/diffplug/spotless/blob/main/plugin-gradle/README.md) ( spotless 가이드 )
- [https://velog.io/@cekim/briefing-spotless](https://velog.io/@cekim/briefing-spotless)