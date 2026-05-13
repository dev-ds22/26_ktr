---
layout: single
title: "[NginX] nginx 프록시 서버 만들기"
excerpt: "nginx 프록시 서버"

categories:
  - tech
tags:
  - [kafka, topic]

toc: false
toc_sticky: true

date: 2023-01-16
last_modified_at: 2023-01-16
---

# SSL 및 Reverse-Proxy 설정

1. vi /etc/nginx/nginx.conf
2. SSL 설정 추가
3. reverse-proxy 설정 추가
4. 포트 열기
  - firewall-cmd --zone=public --permanent --add-port={Post 번호}/tcp
5. 설정 내용을 반영.
  - firewall-cmd --reload
6. nginx 재시작
  - systemctl nginx stop
  - systemctl nginx start

### 참고 
- 활성화 상태의 zone 리스트
  - firewall-cmd --get-active-zones
- 서비스 리스트
  - firewall-cmd --get-service
- 특정 존에 있는 서비스 리스트
  - public 존에 있는 열린포트 확인
  - firewall-cmd --zone=public --list-all

## SSL 설정

```bash
  server {
    listen       445 ssl http2;
    listen       [::]:445 ssl http2;
    server_name  _;
    root         /usr/share/nginx/html;

    ssl_certificate "/var/lib/jenkins/secrets/server.crt";
    ssl_certificate_key "/var/lib/jenkins/secrets/server.key";

```

## reverse-proxy 설정

```bash
  location / {
    proxy_pass http://127.0.0.1:8013;
  }
```

## rewrite 필요시

2. rewrite
/api/~~~에서 /api 부분은 제외하고 요청을 보내고 싶다면 rewrite 구문을 사용하면 된다.

```bash
// nginx.conf

location /api {
    rewrite ^/api(.*)$ $1?$args break;
    proxy_pass   http://express-server;
    ...
}
```
- /api/posts?id=1 이라는 요청이 들어왔을 때, /api 이후에 오는 uri가 $1 이 된다.
(정규표현식 capturing group)
- nginx에서 proxy를 사용할 때 기본적으로 GET요청 시 같이 보내는 argument가 보내지지 않는데, &args 변수를 사용하면 같이 보낸 argument에 접근할 수 있다.
- /api/posts?id=1 은 /posts?id=1 로 rewrite 되어서 전송이 된다.

#### 참고 (/etc/nginx/nginx.conf)
```bash
  # For more information on configuration, see:
  #   * Official English Documentation: http://nginx.org/en/docs/
  #   * Official Russian Documentation: http://nginx.org/ru/docs/

  user nginx;
  worker_processes auto;
  error_log /var/log/nginx/error.log;
  pid /run/nginx.pid;

  # Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
  include /usr/share/nginx/modules/*.conf;

  events {
      worker_connections 1024;
  }

  http {
      log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';

      access_log  /var/log/nginx/access.log  main;

      sendfile            on;
      tcp_nopush          on;
      tcp_nodelay         on;
      keepalive_timeout   65;
      types_hash_max_size 4096;

      include             /etc/nginx/mime.types;
      # Load modular configuration files from the /etc/nginx/conf.d directory.
      # See http://nginx.org/en/docs/ngx_core_module.html#include
      # for more information.
      include /etc/nginx/conf.d/*.conf;

      server {
          listen       81;
          listen       [::]:81;
          server_name  _;
          root         /usr/share/nginx/html;


          # Load configuration files for the default server block.
          include /etc/nginx/default.d/*.conf;

          error_page 404 /404.html;
          location = /404.html {
          }

          error_page 500 502 503 504 /50x.html;
          location = /50x.html {
          }
      }

  # Settings for a TLS enabled server.
  #
      server {
          listen       445 ssl http2;
          listen       [::]:445 ssl http2;
          server_name  _;
          root         /usr/share/nginx/html;

          ssl_certificate "/var/lib/jenkins/secrets/server.crt";
          ssl_certificate_key "/var/lib/jenkins/secrets/server.key";
          ssl_session_cache shared:SSL:1m;
          ssl_session_timeout  10m;
          ssl_ciphers PROFILE=SYSTEM;
          ssl_prefer_server_ciphers on;

          # Load configuration files for the default server block.
          include /etc/nginx/default.d/*.conf;

          error_page 404 /404.html;
              location = /40x.html {
          }

          error_page 500 502 503 504 /50x.html;
              location = /50x.html {
          }

          location / {
              proxy_pass http://127.0.0.1:8013;
          }
      }
  }

```
  
---
  
# 이하는 설명
  

---
  


## 실제로 사용 가능한 nginx 프록시 서버 만들기 (docker-compose)
실제로 사용 가능한 nginx 프록시 서버를 만들기 위한 여정입니다. 필자는 보안에 대한 전문가도 아니고 nginx 전문가도 아닙니다만 그 과정을 최대한 종합적으로 정리해보고자 합니다. 본 글에서는 docker 를 활용하므로, docker 에 대한 기본적인 개념 및 docker-compose 를 간단하게 이용할 수 있는 정도의 지식만 있으면 됩니다. 또한 nginx 를 활용하므로 nginx 가 어떤 웹서버인지 간단하게나마 알면 좋습니다.

간단 개념 정리
CA : 인증서를 발급해주는 기관
OCSP Stapling : 인증서가 유효기간이 되기 전에 파기되었는지 아닌지 확인하는 한 가지 방법입니다. 파기되었는지의 여부는 CA에게 직접 물어보는 수 밖에 없습니다. 그래서 웹 서버에서 CA로 주기적으로 확인 요청을 보내고, CA는 웹 서버에게 “2015.11.11 확인됨” 이라는 메시지를 보내 놓습니다. 웹 서버는 이 메시지를 받은 날짜와 함께 잘 가지고 있다가 클라이언트가 웹 서버에게 요청을 한다면 그 때 인증서에 메시지를 스테이플러로 붙여서 (Stapling) 클라이언트에게 전달합니다. 자세한 내용은 https://rsec.kr/?p=386 참조
HSTS(HTTP Strict Transport Security) : 웹 사이트에 접속할 때, 강제적으로 HTTPS Protocol로만 접속하게 하는 기능
SSL / TLS 인증서 (HTTPS) 동작 과정 : 친절한 설명은 생활코딩 참조, 자세하고 기술적인 설명은 https://chp747.tistory.com/155 참조
목표
Nginx 를 리버스 프록시 서버로 사용한다. (실제 서버가 뒤에 숨겨져서 위치한다.)
모든 www는 www가 아닌 걸로 리다이렉트한다. (예: www.example.com → example.com ) 서브도메인이 있을 경우 당연히 리다이렉트하지 않는다.
ssl / tls 인증서를 사용한다. (https 를 지원한다) 발급은 Let’s Encrypt 인증서 활용 (+ certbot 활용)
모든 http 는 https 로 리다이렉트한다.
nginx 리버스 프록시 서버와 실제 서버는 단순한 http 로 통신한다.
docker-compose 로 프로그램들을 실행시킨다.
gzip 을 지원한다.
최종 configuration 파일 모습
모든 설정은 /etc/nginx/nginx.conf 파일에서 시작됩니다. 도커에서의 설정에 따라 이 conf 파일을 통째로 바꿔치기할 수 있고, 또는 /etc/nginx/sites-enabled/ 경로 안에 있는 conf 파일만 새로 추가할 것인지를 선택할 수 있습니다. 저 경로가 어떻게 나오는 것이냐구요? 보통 기본적으로 주어지는 nginx.conf 파일의 http context 내부에는 다음과 같은 include 를 확인할 수 있습니다. 그러므로 기본 nginx.conf 파일을 수정하지 않는다면 저 경로 안에 설정이 기본적으로 로딩된다는 점도 유추할 수 있습니다.

include /etc/nginx/conf.d/*.conf;
include /etc/nginx/sites-enabled/*;
이 글에서 만들 conf 파일은 총 3개입니다. nginx.conf 파일과, 해당 파일에서 include 하는 web.conf 파일, 그리고 각 server context 에서 반복적으로 사용하는 /etc/nginx/server-https-common.conf 파일입니다.

# nginx.conf
user nginx;
worker_processes 1;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;
events {
    worker_connections 1024;
}
http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # 웹 설정하는 부분
    include /etc/nginx/web.conf;

    # 로그 파일에 대한 포맷을 설정해주는 부분
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
    '$status $body_bytes_sent "$http_referer" '
    '"$http_user_agent" "$http_x_forwarded_for"';
    access_log /var/log/nginx/access.log main;
    
    # https://www.lesstif.com/system-admin/nginx-http-413-client-intended-to-send-too-large-body-86311189.html
    client_max_body_size 8M; 

    # gzip configuration
    gzip on;
    gzip_disable "msie6";
    gzip_min_length 10240;
    gzip_buffers 32 32k;
    gzip_comp_level 9;
    gzip_proxied any;
    gzip_types text/plain application/javascript application/x-javascript text/javascript text/xml text/css;
    gzip_vary on;

    sendfile on;
    keepalive_timeout 65;
}
# web.conf

upstream example {
    server web:4000;
}

# force to https, not http.
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    return 301 https://$host$request_uri;
}

# force www to non-www examle.com
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    server_name www.example.com;
    
    include /etc/nginx/server-https-common.conf;

    return 301 https://example.com$request_uri; 
}

# example no subdomain
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    server_name example.com;

    include /etc/nginx/server-https-common.conf;
    
    location / {
        proxy_set_header    X-Real-IP           $remote_addr;
        proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto   $scheme;
        proxy_set_header    Host                $host;
        proxy_set_header    X-Forwarded-Host    $host;
        proxy_set_header    X-Forwarded-Port    $server_port;
        proxy_set_header    X-NginX-Proxy       true;
        proxy_pass_header   Set-Cookie;
        proxy_http_version  1.1;
        proxy_cache_bypass  $http_upgrade;
        proxy_pass http://example/;
    }
}
# server-https-common.conf

# ssl configuration
ssl_certificate         /etc/ssl/fullchian.pem;
ssl_certificate_key     /etc/ssl/privkey.pem;
ssl_trusted_certificate /etc/ssl/chain.pem;
ssl_session_timeout     1d;
ssl_session_cache       shared:MozSSL:10m;  # about 40000 sessions
ssl_session_tickets     off;

# modern configuration
ssl_protocols               TLSv1.3;
ssl_prefer_server_ciphers   off;

# HSTS (ngx_http_headers_module is required) (63072000 seconds)
add_header Strict-Transport-Security    "max-age=63072000" always;

# other security settings
add_header X-Frame-Options              SAMEORIGIN;
add_header X-Content-Type-Options       nosniff;
add_header X-XSS-Protection             "1; mode=block";

# OCSP stapling
ssl_stapling on;
ssl_stapling_verify on;

# replace with the IP address of your resolver
# resolver 127.0.0.1;
그리고 이 nginx 리버스 프록시 서버는 docker 컨테이너로 돌아가므로, 인증서 위치와 호스트 이름 등을 설정해주기 위한 docker-compose.yml 파일을 만듭니다.

version: '3'
services:
  proxy:
    container_name: proxy
    hostname: proxy
    image: nginx:latest
    ports:
      - '80:80' # common web
      - '443:443' # https
    volumes:
      - ./proxy/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./proxy/web.conf:/etc/nginx/web.conf:ro
      - ./proxy/server-https-common.conf:/etc/nginx/server-https-common.conf:ro
      - /etc/letsencrypt/live/사이트이름:/etc/ssl:ro
    depends_on:
      - web
  web:
    image: 사용할-이미지-이름
    hostname: web
    ports:
      - '4000:4000'
web.conf 설정
기본적인 서버의 설정은 https://ssl-config.mozilla.org/ 에서 가져왔습니다. 여기에는 아주 유용한 것들이 많습니다. 여러가지 버전 중에 위 내용은 modern 을 선택한 것에서 좀 수정을 거친 것인데, intermediate 를 선택하면 더 길고 복잡해져서 그냥 이걸로 했습니다. 차이점은 바로 아래에서 확인할 수 있는데, 큰 차이점은 아닙니다.

# modern configuration
ssl_protocols TLSv1.3;
ssl_prefer_server_ciphers off;

# 위에 거 대신
# 아래 게 쓰입니다.    

# curl https://ssl-config.mozilla.org/ffdhe2048.txt > /path/to/dhparam
ssl_dhparam /path/to/dhparam;

# intermediate configuration
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;
upstream 은 여러 개의 서버를 하나로 묶는 역할입니다. upstream 에서 서버를 정의할 때 docker-compose.yml 에서 hostname으로 지정한 이름을 써도 됩니다. 실제 서버에서 사용하는 포트도 붙이도록 합시다. (예: web:4000) 아래 코드를 참조해주세요.

upstream example {
    server web:4000;
}
server 에서는 총 두 번의 리다이렉트가 있습니다. 강제로 https 로 연결하는 리다이렉트와 www를 없애는 리다이렉트입니다. 여기서 순서가 중요한데, 필자는 먼저 http 를 https 로 하는 리다이렉트를 먼저 수행하므로, www 도 https 로 연결될 수 있습니다. 그러므로 우리의 인증서에는 기본 도메인 뿐만 아니라 www 가 붙은 도메인도 포함되어 있어야 합니다! 인증서를 만들 때 주소를 한꺼번에 설정하여 하나의 파일에 몰아넣으면 되므로 주소에 따라 인증서를 다르게 설정해야 하거나 할 필요는 없습니다. 아래 코드를 참조해주세요.

# force to https, not http.
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    return 301 https://$host$request_uri;
}

# force www to non-www examle.com
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    server_name www.example.com;
    
    include /etc/nginx/server-https-common.conf;

    return 301 https://example.com$request_uri; 
}
간단 설명

– listen : 응답받을 ip 주소와 포트를 결정합니다.
– server_name : 응답받을 uri 를 결정합니다. listen 에는 해당하는데 server_name 을 찾을 수 없을 경우 해당 listen 의 default_server 설정이 되어 있는 server 블록으로 넘어갑니다. 위 예제에서는 모든 ip주소의 80 포트에 대해서 default_server 설정을 해놓았으므로, 80 포트에서 server_name 을 찾지 못한 모든 연결은 모두 해당 블록으로 넘어갑니다. 80 포트에 대해서 별다른 server_name 을 설정해놓은 것이 아무것도 없으므로 기본적으로 모두 해당 블록으로 넘어간다고 보면 됩니다.

자 그렇다면 이제 진짜 경로로 들어오게 되었을 때의 처리인데요, 이는 location context 안에 뭐가 많이 있습니다. 각각 설정이 겹치는 것도 있고 어떤 것이 필요하고 어떤 것이 필요하지 않은지에 대한 명확한 구분은 정말 하기 어렵더라구요. 인터넷에서는 아무리 찾아봐도 그 명확한 선이란 게 보이지 않아서 넣을 수 있는 건 다 넣었습니다. 아래를 참조해주세요.

    location / {
        proxy_set_header    X-Real-IP           $remote_addr;
        proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto   $scheme;
        proxy_set_header    Host                $host;
        proxy_set_header    X-Forwarded-Host    $host;
        proxy_set_header    X-Forwarded-Port    $server_port;
        proxy_set_header    X-NginX-Proxy       true;
        proxy_pass_header   Set-Cookie;
        proxy_http_version  1.1;
        proxy_cache_bypass  $http_upgrade;
        proxy_pass http://example/;
    }
server-https-common.conf 설정
인증서 경로 설정
본 글에서는 Let’s Encrypt 로 SSL 인증서를 발급합니다. 기본적으로 certbot 으로 인증서를 만든다면 /etc/letsencrypt/live/사이트이름 경로 내에 다음 4개의 pem 파일이 위치하게 됩니다.

cert.pem
chain.pem
fullchain.pem
privkey.pem
이 파일들을 nginx ssl 세팅과 연결시키려면 다음과 같이 하면 됩니다. (Let’s Encrypt Community 글 참조)

ssl_certificate should point to fullchain.pem
ssl_certificate_key should point to privkey.pem
ssl_trusted_certificate should point to chain.pem
그러므로 docker-compose.yml 에서 다음과 같이 경로 설정을 해두고,

version: '3'
services:
  proxy:
    # 중략
    volumes:
      - /etc/letsencrypt/live/사이트이름:/etc/ssl:ro
# 후략
conf 파일에 아래와 같이 설정하면 됩니다.

# ssl configuration
ssl_certificate         /etc/ssl/fullchian.pem;
ssl_certificate_key     /etc/ssl/privkey.pem;
ssl_trusted_certificate /etc/ssl/chain.pem;
헤더를 설정하는 부분과 그에 대한 설명은 다음과 같습니다.

# HSTS (ngx_http_headers_module is required) (63072000 seconds)
add_header Strict-Transport-Security    "max-age=63072000" always;

# other security settings
add_header X-Frame-Options              SAMEORIGIN;
add_header X-Content-Type-Options       nosniff;
add_header X-XSS-Protection             "1; mode=block";
Strict-Transport-Security : HTTPS 강제 연결을 하라고 브라우저에게 알리는 역할입니다. 자세한 내용은 Mozila 문서 참조.
X-Frame-Options: 해당 페이지를 외부의 어떤 사이트 내의 iframe 등에서 임베딩할 수 있는지에 대한 여부를 설정합니다. 자세한 내용은 Mozila 문서 참조.
X-Content-Type-Options nosniff : 브라우저로 하여금 웹서버가 보내는 MIME 형식 이외의 형식으로 해석을 확장하는 것을 제한하도록 합니다. 그러니까 text/css 형식으로 온 것만 css 로 사용하는 등 제한을 거는 것이죠. 자세한 내용은 Mozila 문서 참조.
X-XSS-Protection: 브라우저에게 xss 공격에 대한 대응을 설정할 수 있도록 합니다. 자세한 내용은 Mozila 문서 참조.




<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

  </pre>
</details>