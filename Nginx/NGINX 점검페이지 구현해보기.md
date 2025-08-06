# NGINX에서 점검페이지 적용

---

> 💡기존에 별도 서비스 점검(혹은 배포) 시에 사용자에게 별도 안내페이지가 존재하지 않았는데 이 부분을 개선하여 서비스에 안내페이지
>  혹은 JSON 메세지가 전달되도록 목표함


## 기능 도입 고려사항

---

- HTML 요청, JSON 요청 모두 대응
- WebView 를 활용한 서비스 이기 때문에 점검 페이지가 캐시되는 것 을 핸들링 가능
- 서비스 점검 시간을 쉽게 변경할 수 있음
- 사용자에게는 점검 페이지가 노출 되지만 내부 테스트는 진행될 수 있어야 됨
- 무료 여부

## 방법1) NGINX 에서 Health Check 기능 검토

---

Nginx 에서 Health Check 기능을 사용하기 위해서는 `ngx_http_upstream_check_module` 모듈을 설치해야 하는데 해당 모듈은 NGINX Plus 에서 제공되는 기능이며 별도 라이선스 구매가 필요하여 **제외**

## 방법2) NGINX 수동 설정을 활용하여 처리

---

미리 정의한 Nginx 의 CONFIG 파일과 별도의 Shell 파일을 활용하여 적용

### 1. 작업 순서

---

1. Shell 파일을 실행하여 서비스 점검 모드로 전환 **( script 실행 시 start 명령어 적용 )**
2. 필요한 서비스 업데이트 진행
3. 정의된 IP 대역에서 내부 테스트 진행
4. 테스트 완료 시 Shell 파일을 실행하여 점검 모드 해제 **( script 실행 시 end 명령어 적용 )**

### 2. 세부 구현 내용

---

- 서비스 점검 시 사용자에게 노출되는 HTML , JSON 파일을 생성하고, Nginx 에서는 생성된 파일 존재 유무에 따라서 서비스 점검 유무를 판단
- 서비스점검 중 웹사이트에 접근하는 IP에 따라서 점검 페이지가 노출되거나 테스트할 수 있도록 정상 접근이 됨
- HTTP Header 중 Accept 내용에 따라서 Nginx 에서 응답을 HTML 혹은 JSON 파일로 함
- 위의 작업을 좀 편하게 하기 위해서 별도의 Shell 파일을 활용함

### 3. 준비물

---

### 3-1. 서비스 점검 HTML 파일 생성

---

```bash
# /usr/share/nginx/html/html_maintenance.html

<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>서비스 점검 중</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            background-color: #f0f0f0;
            margin: 0;
            padding: 0;
        }

        .container {
            padding: 50px;
            background-color: white;
            border-radius: 10px;
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
            max-width: 600px;
            margin: 100px auto;
        }

        h1 {
            color: #d9534f;
        }

        p {
            font-size: 18px;
            color: #555;
            line-height: 1.6;
        }

        .time {
            font-weight: bold;
            font-size: 22px;
            color: #333;
            padding: 15px;
            background-color: #f9f9f9;
            border-radius: 5px;
            display: inline-block;
            margin-top: 10px;
        }

        .note {
            font-size: 16px;
            color: #888;
            margin-top: 20px;
        }

        .footer {
            font-size: 14px;
            color: #aaa;
            margin-top: 30px;
        }
    </style>
</head>
<body>

<div class="container">
    <h1>서비스 점검 중입니다</h1>
    <p>현재 시스템은 예정된 점검으로 인해 일시적으로 이용할 수 없습니다.</p>
    <p>점검 시간은 아래와 같습니다:</p>
    <p class="time">MAINTENANCE_START_TIME ~ MAINTENANCE_END_TIME</p>
    <p class="note">이용에 불편을 드려 죄송합니다. 양해해 주셔서 감사합니다.</p>
</div>

</body>
</html>
```

- `MAINTENANCE_START_TIME` : 서비스 점검 시작 시간 ( 예) 2025-01-01 10:00 )
- `MAINTENANCE_END_TIME` : 서비스 점검 종료 시간 ( 예) 2025-01-01 12:00 )

### 3-2. 서비스 점검 JSON 파일 생성

---

```bash
# /usr/share/nginx/html/json_maintenance.json

{
    "error": "service_unavailable",
    "message": "서비스 점검중 입니다.",
    "maintenance_start_time": "MAINTENANCE_START_TIME",
    "maintenance_end_time": "MAINTENANCE_END_TIME"
}
```

- `MAINTENANCE_START_TIME` : 서비스 점검 시작 시간 ( 예) 2025-01-01 10:00 )
- `MAINTENANCE_END_TIME` : 서비스 점검 종료 시간 ( 예) 2025-01-01 12:00 )

### 3-3. Nginx Config 수정

---

<a id="default"></a>
### 기본 적용 사항

---

```bash
# /etc/nginx/conf.d/default.conf

map $http_accept $maintenance_page {
  default             /maintenance.html
  "~application/json" /maintenance.json
}

server {
  location / {
    if(-d /usr/share/nginx/html/mro/user) {
		    return 503;
    }
  }
  
  error_page 503 $maintenance_page 

  location = /maintenance.html {
    root /usr/share/nginx/html;
    internal;
    add_header Cache-Control "no-store, no-cache, must-revalidate, max-age=0";
    add_header Pragma "no-cache";
    add_header Expires 0;
  }

  location = /maintenance.json {
    root /usr/share/nginx/html;
    internal;
    default_type application/json;
    add_header Cache-Control "no-store, no-cache, must-revalidate, max-age=0";
    add_header Pragma "no-cache";
    add_header Expires 0;
  }
}
```

- `Client` 에서 요청하는 `Header 의 ACCEPT` 을 기준으로 `HTML` 혹은 `JSON` 응답 정의
- KT 클라우드 기능을 사용하지 않고 Nginx 기능을 활용하기 위해서 **서비스 점검 유무를 특정
파일이 존재하는지 여부로 판단**
- WebView 상에서 캐시 방지를 위한 Header 설정
- 외부에서 `/maintenance.html` , `/maintenance.json` URL 로 접근 시 `404 ERROR` 반환을 적용하기 위해서 `internal` 키워드 적용

<a id="one"></a>
### 보완 1 차

---

> 💡[이 부분](#default) 에서 적용된 내용만 가지고는 서비스 오픈 전에 내부 테스트를 진행 할 수가 없다
> 모든 요청에 대해서 503 ERROR 를 발생 시키기 때문이다

> 💡보안 1차 내용은 별도로 테스트를 진행하지 않아서 실제 적용 시 다른 이슈가 발생 될 수 있음
>


```bash

server {
  location / {
    if(-d /usr/share/nginx/html/mro/user) {
		    
		    auth_basic "내부 테스트";
		    auth_basic_user_file /etc/nginx/.htpasswd;
		    
		    return 503;
    }
  }
}
```

- Nginx 의 사용자 인증 기능을 활용하여 인증이 성공한 경우에만 서비스를 정상적으로 이용
가능하고 인증이 실패한 경우에는 503 ERROR 가 발생
- **주의사항**
    - **반드시 HTTPS와 함께 사용해야 함**
        - HTTP 평문으로 하는 경우, Header 의 `Authorization` 에 아이디와 패스워드 정보가 
        노출되기 때문 ( Base64 인코딩 데이터 이기 때문에 쉽게 디코딩이 됨 )
    - **htpasswd 파일이 웹서버의 URI 공간에 있으면 안됨**
        - 사용자가 브라우저의 URI 로 접근이 불가해야 됨

### 보완 2차

---

> 💡[이 기능](#one) 을 활용하게 되면 내부 테스트를 진행하는 사용자와 서비스를 이용하는 사용자에게인증을 요구하는 화면이 노출될 수 있음
> 

```bash

# HTTP HEADER 를 통해서 Client 전달 할 형식 정의
map $http_accept $maintenance_page {
  default             /maintenance.html;
  "~application/json" /maintenance.json;
}

# 서비스 점검 중에 내부 테스트를 할 수 있는 아이피 관리
map $remote_addr $access_pass_ips {
  default            0;
  112.221.181.122    1; 
}

server {
  location / {
   
    set $IS_PASS '';
    if (-d /usr/share/nginx/html/mro/user) {
      set $IS_PASS 'D';
    }

    if ($access_pass_ips != 0) {
      set $IS_PASS '${IS_PASS}A';
    }

    if ($IS_PASS = 'D') {
      return 503;
    }
  }
  
  error_page 503 $maintenance_page;

  location = /maintenance.html {
    root /usr/share/nginx/html/mro/user;
    internal;
    add_header Cache-Control "no-store, no-cache, must-revalidate, max-age=0";
    add_header Pragma "no-cache";
    add_header Expires 0;
  }

  location = /maintenance.json {
    root /usr/share/nginx/html/mro/user;
    internal;
    default_type application/json;
    add_header Cache-Control "no-store, no-cache, must-revalidate, max-age=0";
    add_header Pragma "no-cache";
    add_header Expires 0;
  }
}
```

- 서비스에 접근할 수 있는 IP 목록을 작성하여  테스트 사용자 와 서비스 이용자에게 노출되는
화면을 다르게 적용
- **Nginx 는 중첩 IF 문이 허용되지 않기 때문에 주의 필요**

### 다른 대안

---

```bash

server {
  
  location /internal-test {
    proxy_pass http://localhost:8080;
  }
  
  location / {
    if(-d /usr/share/nginx/html/mro/user) {
		    return 503;
    }
  }
}
```

- 상황에 따라서는  테스트를 하기 위한 PATH 를 구성하는게 좋을 수도 있음
- 운영 중인 서비스에서는 PATH 가 추가되면 Application 소스를 수정해야 하기 때문에 이번에는
제외

### 3.4 Shell 파일 ( mro_script.sh )

---

```bash
#!/bin/bash

ORIGIN_PATH="/usr/share/nginx/html"
COPY_PATH="/usr/share/nginx/html/mro"
HTML_FILE_NAME="html_maintenance.html"
JSON_FILE_NAME="json_maintenance.json"

ALLOWED_NAMES=("user" "api" "web")

usage() {
  echo "Usage: $0 <function> [options]"
  echo ""
  echo "Functions:"
  echo "  start           서비스점검 페이지 적용하는 함수"
  echo "  end             서비스점검 종료하는 함수"
  echo ""
  echo "Options for start:"
  echo "  -n <name>       서비스 점검 페이지 적용할 서비스( user : 사용자 서버, api : API 서버, web : Web 서버 )"
  echo "  -s <start_date> 서비스 점검 시작일 ( YYYY-MM-dd HH:mm 형식)"
  echo "  -e <end_date>   서비스 점검 종료일 ( YYYY-MM-dd HH:mm 형식)"
  echo "  --help          사용법 및 옵션 확인"
  echo ""
  echo "Description:"
  echo "  서비스 점검 전에 실행하여 점검 안내 HTML 혹은 JSON 문구가 노출되도록 한다."
  echo "  서비스 점검이 완료되면 end 함수를 통해 생성되었던 리소스 정리 작업은 필수 이다."
  echo "Example:"
  echo " $0 start -n user -s '2025-01-01 01:00' -e '2025-01-01 03:00'"
  echo " $0 start -n 'user api' -s '2025-01-01 01:00' -e '2025-01-01 -3:00'"
  echo " $0 end"
}

if [[ "$1" == "--help" ]]; then
  usage
  exit 0
fi

start() {

 local NAMES=()
 local START_DT=""
 local END_DT=""

 while getopts "n:s:e:" opt; do
   case "$opt" in
     n)
        read -r -a NAMES <<< "$OPTARG"

        local valid_name_count=0
        for name in "${NAMES[@]}"; do
         if [[ ! "${ALLOWED_NAMES[*]}" =~ "${name}" ]]; then
           echo "Error: Invalid name founded. Allowed names are: ${ALLOWED_NAMES[*]}"
           return 1
         fi
        done
        ;;
     s) START_DT=$OPTARG ;;
     e) END_DT=$OPTARG ;;
     *) echo "Invalid option for start"; return 1 ;;
   esac
 done

 if [ ${#NAMES[@]} -eq 0 ]; then
   echo "Error: -n option is required."
   return 1
 fi

 if [ -z "$START_DT" ]; then
   echo "Error: -s (start date) option is required."
   return 1
 fi

 if [ -z "$END_DT" ]; then
   echo "Error: -e (end date) option is required."
   return 1
 fi

 for name in "${NAMES[@]}"; do

  local TARGET_DIR="$COPY_PATH/$name"

  [ ! -d "$TARGET_DIR" ] && mkdir -p "$TARGET_DIR"

  sed -e "s/MAINTENANCE_START_TIME/$START_DT/g" -e "s/MAINTENANCE_END_TIME/$END_DT/g" "$ORIGIN_PATH/$HTML_FILE_NAME" > "$TARGET_DIR/maintenance.html"
  sed -e "s/MAINTENANCE_START_TIME/$START_DT/g" -e "s/MAINTENANCE_END_TIME/$END_DT/g" "$ORIGIN_PATH/$JSON_FILE_NAME" > "$TARGET_DIR/maintenance.json"

  echo "Processed for name: $name -> Directory: $TARGET_DIR"
 done

}

end() {
 echo "리소스 정리 시작"

 for name in "${ALLOWED_NAMES[@]}"; do
  local DEL_TARGET_PATH="$COPY_PATH/$name"

  if [ -d "$DEL_TARGET_PATH" ]; then
    rm -rf "$DEL_TARGET_PATH"
    echo "DELETE PATH : $DEL_TARGET_PATH"
  else
    echo "$DEL_TARGET_PATH does not exist."
  fi
 done

 echo "리소스 정리 완료"

}

if declare -f "$1" > /dev/null; then
  FUNC="$1"
  shift
  "$FUNC" "$@"
else
  echo "Function '$1' not found."
  exit 1
fi

```

- `--help`  : 옵션으로 사용법 확인 가능

```bash
[root@ms-webwas-test01 maintenance]# ./mro_script.sh --help
Usage: ./mro_script.sh <function> [options]

Functions:
  start           서비스점검 페이지 적용하는 함수
  end             서비스점검 종료하는 함수

Options for start:
  -n <name>       서비스 점검 페이지 적용할 서비스( user : 사용자 서버, api : API 서버, web : Web 서버 )
  -s <start_date> 서비스 점검 시작일 ( YYYY-MM-dd HH:mm 형식)
  -e <end_date>   서비스 점검 종료일 ( YYYY-MM-dd HH:mm 형식)
  --help          사용법 및 옵션 확인

Description:
  서비스 점검 전에 실행하여 점검 안내 HTML 혹은 JSON 문구가 노출되도록 한다.
  서비스 점검이 완료되면 end 함수를 통해 생성되었던 리소스 정리 작업은 필수 이다.
Example:
 ./mro_script.sh start -n user -s '2025-01-01 01:00' -e '2025-01-01 03:00'
 ./mro_script.sh start -n 'user api' -s '2025-01-01 01:00' -e '2025-01-01 03:00'
 ./mro_script.sh end

```

## 4. 결과

---

- HTML 요청 시
    
    ![HTML_점검중.PNG](/assets/nginx_maintenance_page/1.png)
    
- JSON 요청 시
    
    ![JSON_점검페이지.png](/assets/nginx_maintenance_page/2.png)
    

## 부록

---

### 1. Nginx 설치된 모듈 확인

---

```bash

nginx -V 2>&1 | tr -- - '\n' | grep _module

http_ssl_module
http_v2_module
http_realip_module
http_addition_module
http_xslt_module=dynamic
http_image_filter_module=dynamic
http_sub_module
http_dav_module
http_flv_module
http_mp4_module
http_gunzip_module
http_gzip_static_module
http_random_index_module
http_secure_link_module
http_degradation_module
http_slice_module
http_stub_status_module
http_perl_module=dynamic
http_auth_request_module
mail_ssl_module
stream_ssl_module
```

### 2. VI 편집기에서 내용 복사

---

1. 복사 시작 해야 하는 라인으로 이동
    1. V ( Shift + v ) : 비주얼 라인 모드
2. 커서를 복사 범위까지 이동
    1. G ( Shift + g ) : 맨 마지막 라인까지 이동
3. 선택 된 범위 복사
    1. y : 복사