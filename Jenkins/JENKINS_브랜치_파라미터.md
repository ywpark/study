# JENKINS 에서 GIT 브랜치 환경변수 정의 하기

> 💡 release 브랜치를 매번 버전 별로 생성하는 Git 전략을 가지고 있다. 그렇기에
> 기능 Jenkins 로 배포할 때 매번 Jenkins 가 바라보는 브랜치를 매번 변경해야 하는 작업이
> 존재하는데 이 부분을 좀 더 쉽게 하고자 아래와 같이 적용
> 

## 1. Jenkins 의 Git plugin 을 설치한다.

---

```bash
Git Parameter Plug-In 을 Jenkins 에 설치한다.
```

![1.png](/assets/jenkins/1.png)

## 2.  Jenkins Job 에서 [ 구성 ] 접근한다.

---

![2.png](/assets/jenkins/2.png)

## 3.  General 에서 [ 매개변수 ] 옵션을 설정한다.

---

![3.png](/assets/jenkins/3.png)

```
1. 아래 항목의 [ 매개변수 추가 ] SelectBox 를 통해서 Git Parameter 를 추가한다.
2. Name, Parameter Type, Default Value 값을 정의한다.
  - Name : 나중에 변수로 사용할 이름이다.
  - Parameter Type : 상황에 맞게 여러가지를 선택할 수 있는데 우리는 branch 기준으로 하기에
                    Branch 로 선택
  - Default Value : 기본값 정의
```

## 4. 기존 브랜치명을 변수명으로 변경한다.

---

![4.png](/assets/jenkins/4.png)

```
Branch Specifier 항목에 3번 항목에서 정의한 Name 값으로 작성한다.

예시) Name 항목을 feat 라고 작성하였다면 ${feat} 이라고 작성하면 된다.
```

## 5. 빌드 해보기

---

![5.png](/assets/jenkins/5.png)

```

1. [ 파라미터와 함께 빌드 ] 하기를 누르면 오른쪽에 현재 브랜치 목록이 노출됩니다.
2. 배포할 특정 브랜치를 선택한 후 하단의 [빌드하기]를 누르면 해당 브랜치로 배포 됩니다.
```