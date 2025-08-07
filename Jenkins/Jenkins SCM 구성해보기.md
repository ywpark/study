# Jenkins SCM 적용하기

---

> 💡 SCM(Software Configuration Management) ?
>  소프트웨어의 변경사항을 추적하고 통제하는 작업이다. 단순하게 생각해보면 config 파일의 이력 관리라고 생각하면 쉽다.
> 

## 0. 환경

---

- OpenJDK 11
- Jenkins 2.387.3
- Ubuntu 22.04.2

## 1. 왜??

---

- Jenkins 서버를 일반 데스크탑 PC에서 사용하고 있으며 언제 죽어도 이상하지 않음
- Pipeline 을  Web UI 로 작성하다보니 변경 관리 및 백업이 되어 있지 않아서 Jenkins 서버가
변경되는 경우에는 기존에 구성한 Pipeline 을 마이그레이션 하기가 쉽지 않음
- 트래픽이 늘어나고 비용이 증가함에 따라 결국에는 서버 환경이 컨테이너 방식으로 
변경되지 않을까 함
    - 컨테이너 방식으로 가면 일반적으로 ( k8s + docker ) 로 구성이 될 텐데 Groovy 와 같은
    DSL( Domain specific language ) 로 작성하게 되는데 똑같지는 않지만 눈에 익어놓으면
    좋을 것으로 보임
- IaC (Infrastructure as Code) 방식으로 인프라를 관리하는 방식이  늘어남에 따라 어느 정도
알아 둘 필요가 있음

## 2. 그러나….

---

- Jenkins 에 맞는 DSL 구문을 공부해야 함
- 기초적인 설정은 Jenkins 에서 Web UI 를 통해 정의해야 함
  - 이 부분도 코드화 관리가 가능한지는 별도로 알아보지 않아서 다른 방법이 있는지 까지는 확인되지 않음

## 3. 작성

---

> 💡 Pipleline 을 구성하고 작성할 때 아래와 같은 문법이 존재하며, 둘을 혼용해서 작성할 수는 없다.
>
>  여기서 작성 문법은 다루지 않는다.
> 

- **Scripted 문법**
    - [https://www.jenkins.io/doc/book/pipeline/syntax/#scripted-pipeline](https://www.jenkins.io/doc/book/pipeline/syntax/#scripted-pipeline)
    - 최당단이 `node` 로 시작
- **Declarative 문법**
    - [https://www.jenkins.io/doc/book/pipeline/syntax/#declarative-pipeline](https://www.jenkins.io/doc/book/pipeline/syntax/#declarative-pipeline)
    - 최상단이 `pipeline` 으로 시작
- **jenkinsfile 위치**
    
    ![7.png](/assets/jenkins/7.png)
    
- **Jenkins**
    
    ![8.png](/assets/jenkins/8.png)
    
- **Declarative 문법으로 작성**
    
    ```groovy
    pipeline {
    
        agent any
    
        parameters {
            choice(name : 'BUILD_ENV', choices: ['dev', 'qa', 'prod'], description: '배포 환경 파라미터 정의!!')
        }
    
        // 시스템 환경 변수를 설정
        environment {
            DEPLOY_SCRIPT = ""
        }
    
        // jenkins 가 관리하 도구 정의
        tools {
        
            jdk 'OpenJDK 11'
            gradle 'gradle-Ver7.1.1'
    
        }
    
        stages {
    
            stage('Initialize') {
                steps {
                    script {
                        if(params.BUILD_ENV == 'dev') {
                            DEPLOY_SCRIPT = 'dev_script.sh'
                        }
                        else if(params.BUILD_ENV == 'qa') {
                            DEPLOY_SCRIPT = 'qa_script.sh'
                        }
                        else if(params.BUILD_ENV == 'prod') {
                            DEPLOY_SCRIPT = 'prod_script.sh'
                        }
                        else {
                            error "Unknown environment :: ${params.BUILD_ENV}"
                        }
    
                        echo "Using deploy script :: ${DEPLOY_SCRIPT}"
    
                    }
                }
            }
    
            // Git 에 대한 정보를 Jenkins 에서 관리한다
            // 좀더 디테일하게 정의해야 하는 경우에 stage('Checout') 으로 진행
            stage('Checkout SCM') {
                steps {
                    checkout scm
    
                    echo "Checkout SCM 왔음" // steps block 안에 있어야 함
    
                    sh 'chmod +x gradlew'
                }
            }
    
            stage('Build') {
                steps {
                
                    sh "./gradlew clean build --exclude-task test -Pprofile=${params.BUILD_ENV}"
    
                    echo "Build 왔음"
                }
            }
    
            stage('Test') {
                steps {
                    echo "Test 왔지만 아직은 할거 없음"
                }
            }
    
            stage('Package') {
                steps {
                    sh "./gradlew bootJar -Pprofile=${params.BUILD_ENV}"
                    echo "Package 왔음"
                }
            }
    
            stage('Archive') {
                steps {
                   archiveArtifacts artifacts: 'build/libs/*.jar', allowEmptyArchive: true
                   echo "Archive 왔음"
                }
            }
    
            stage('Deploy') {
                steps {
                    echo "Deploy 왔음"
                    sshPublisher(
                        publishers: [
                            sshPublisherDesc(
                                // jenkins 설정에서 생성 한 name 과 동일하게 적용 ( SSH Server Name 항목 )
                                configName: 'test',
                                transfers: [
                                    sshTransfer(
                                        sourceFiles: 'build/libs/deploy.jar',
                                        removePrefix: 'build/libs',
                                        remoteDirectory: '',
                                        execCommand: "cd /home/ubuntu; sh ${DEPLOY_SCRIPT}"
                                    )
                                ],
                                usePromotionTimestamp: false,
                                useWorkspaceInPromotion: false,
                                verbose: true
                            )
                        ]
                    )
                }
            }
        }
    
        post {
            always {
                echo 'Cleaning up workspace...'
                cleanWs()
            }
            success {
                echo 'build & deploy 성공'
            }
            failure {
                echo 'build & deploy 실패'
            }
        }
    }
    ```
    

## 4. 오류 사항

---

- **JDK 를 못 찾는 이슈**
    - 에러 내용
        - `No JDK named OpenJDK 11 found`
    - 작업 내역
        - Code 정의
            
            ```groovy
            #잘 모르는 상태에서 ChatGPT 를 활용한 실수
            environment {
                JAVA_HOME = tool name: 'OpenJDK 11', type: 'JDK'
                PATH = "${JAVA_HOME}\\bin;${env.PATH}"
            }
            ```
            
    - 해결
        - Jenkins 설정 추가
            
            ![9.png](/assets/jenkins/9.png)
            
            > 💡 젠킨스 Web UI 에서 설정하는 경우에는 아래 설정이 필요없지만
            >  SCM 으로 관리할 때는 명시적으로 정의해주어야 한다.
            > 
            > 아래 메뉴에서 설정을 해주어야 한다. <br>
            >  젠킨스 Web -> 젠킨스 관리 -> Global Tool Configuration -> JDK ADD
            > 위에서 작성한 Name 그대로 작성해야 한다.
            > 
            >  java Home 경로 찾을 때 방법 <br>
            > ( 심볼릭 링크 원본을 찾아서 jenkins 에 등록해야 함 ) <br>
            >  STEP 1. which java <br>
            >  STEP 2. readlink -f [step1 에서 나온 경로]
            > 
            
        - Code 정의
            
            ```groovy
            tools {
            		jdk 'OpenJDK 11'
            }
            ```
            

- **Gradle 버전 정의**
    
    <aside>
    💡 별도로 오류는 발생하지 않았고 작성법만 기록
    
    </aside>
    
    - jenkins 설정 추가
        
        ![10.png](/assets/jenkins/10.png)
        
    - Code 정의
        
        ```groovy
        tools {
        		gradle 'gradle-Ver7.1.1'
        }
        ```
        

- **Gradle Build 이슈**
    
    > 💡 build 작업 시 gradle 에 실행 권한이 없어서 오류가 발생.
    > 그렇기에 실행 권한을 부여하는 shell script 추가
    > 
    > 번외) <br>
    >   권한 관련 시스템 설정 같은 경우에는 별도의 shell 파일을 만들어서 <br>
    > 해당 shell 파일을 실행하는 방식으로 관리할 수 있음.
    > 

    - 해결
        
        ```groovy
        stage('Checkout SCM') {
        		steps {
                sh 'chmod +x gradlew'
        		}
        }
        ```
        

- **변수 활용 이슈**
    - 에러 내용
        - `Bad substitution`
    - 작업 내역
        - Code 정의
            
            ```groovy
            stage('Build') {
            	steps {
            		sh './gradlew clean build -Pprofile=${params.BUILD_ENV}'
            	}
            }
            ```
            
    - 해결
        - Code 정의
            
            ```groovy
            stage('Build') {
            	steps {
            		sh "./gradlew clean build --exclude-task test -Pprofile=${params.BUILD_ENV}"
            	}
            }
            ```
            
            <aside>
            💡 아래 사이트에서 힌트를 얻어서 수정
            
             https://www.baeldung.com/ops/jenkins-pipeline-sh-bad-substitution-fix
            
            </aside>
            

      

## 5. 참고 자료

---

- [https://lovethefeel.tistory.com/96](https://lovethefeel.tistory.com/96)