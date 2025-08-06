# TOMCAT 에서 HEAP Option 설정 오류

---

<aside>
💡 Spring 프로젝트에서 외부 Tomcat을 사용하는 경우에 해당

</aside>

## 문제 상황)

- Tomcat 에 Heap 메모리 설정을 변경 적용하고자 했는데 의도하지 않는 Heap 메모리 옵션 설정
같이 적용되는 문제가 발생
    
    ![오류.PNG](/assets/tomcat_heap/1.png)
    
    > 빨간색 박스 : 의도하지 않은 Option
    노란색 박스 : 적용하고자 하는 Option
    > 
    
- 별도로 해당 option 을 적용한 곳을 찾을 수가 없었다

> 💡 일반적으로 option을 설정하는 setenv.sh 파일에도 해당 옵션은 존재하지 않았으며 catalina.sh 파일에서도 <br>
>   해당 Option 세팅 정보를 찾을 수가 없었다

## 해결 상황)

- Tomcat 을 실행할 때 systemctl 명령어를 통해서 tomcat 서비스를 실행 혹은 종료를 시키고 있음
- 해당 명령어를 실행할 때 환경변수를 세팅할 수 있다는 내용을 확인한 뒤 해당 서비스 파일을
오픈해보니 해당 파일에서 환경변수를 세팅하고 있는 것을 찾았고 그 부분을 삭제함
    
    > vi /etc/systemd/system/tomcat.service
    > 
- 삭제한 뒤에 [ systemctl daemon-reload ] 명령어 실행 후 tomcat 재 가동
- 그 뒤에 의도하지 않은 Option 보이지 않음
    
    ![service 캡쳐.PNG](/assets/tomcat_heap/2.png)