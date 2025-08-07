# JENKINS Git 브랜치 오류

> 💡 JENKINS 에서 Git 연동 중 발생 했던 오류

## 1. 히스토리

- 기존에 A 라는 브랜치를 생성하여 JENKINS 에 연동하고 있었음
- Git 정책이 변경되면서 A/xx.xxx 브랜치로 매번 생성하여 사용하고자 함
- Git 에 정상적으로 Push 되는 거까지 확인
- Jenkins Git branch 설정하는 부분에 */A/xx.xxx 로 하고 Build Now 함
- ERROR 발생하고 빌드 안됨

## 2. 에러 내용

```bash
00:00:00.374 From https://abc.com/v1/repos/a
00:00:00.374  ! [new branch]      release/0.9.1-20230224 -> origin/release/0.9.1-20230224  (unable to update local ref)
00:00:00.374 error: some local refs could not be updated; try running
00:00:00.374  'git remote prune https://abc.com/v1/repos/a' to remove any old, conflicting branches
00:00:00.374 
00:00:00.374 	at org.jenkinsci.plugins.gitclient.CliGitAPIImpl.launchCommandIn(CliGitAPIImpl.java:2671)
00:00:00.374 	at org.jenkinsci.plugins.gitclient.CliGitAPIImpl.launchCommandWithCredentials(CliGitAPIImpl.java:2096)
00:00:00.374 	at org.jenkinsci.plugins.gitclient.CliGitAPIImpl.access$500(CliGitAPIImpl.java:84)
00:00:00.374 	at org.jenkinsci.plugins.gitclient.CliGitAPIImpl$1.execute(CliGitAPIImpl.java:618)
00:00:00.374 	at hudson.plugins.git.GitSCM.fetchFrom(GitSCM.java:999)
00:00:00.374 	... 11 more
00:00:00.374 ERROR: Error fetching remote repo 'origin'
```

## 3. 해결 방법

- Git 에 접속해서 처리

> 💡 원인은 JENKINS 에서 관리하고 있는 Local Git Repo 에 A 라는 브랜치가 있다보니 이 부분
> 충돌이 나면서 에러 발생 
> 
> 그렇기에 JENKINS 의 프로젝트 폴더에서 git 업데이트 필요

```bash
# JENKINS 프로젝트로 이동
cd /var/lib/jenkins/workspace/프로젝트 명

# local 에서 Remote 브랜치 정리
git remote prune origin

# 위 명령어를 치기 전에 꼭 remote 한번 더 확인 필요 ( git remote -v )
# git 로그인 정보를 별도로 작성해야 하니 이 부분 또한 미리 준비하자
```

- Jenkins 접속해서 처리

> 💡 아래 화면 처럼 프로젝트 내에서 작업공간 초기화 를 해도 된다
> 

![6.png](/assets/jenkins/6.png)