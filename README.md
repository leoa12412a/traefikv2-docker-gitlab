# 使用traefikv2架設配合Let's Encrypt帶有SSL的Gitlab

這邊利用docker-compose進行架設，traefik版本為v2.4，gitlab版本為gitlab-ce:15.4.5-ce.0
- traefik.9kyd.com作為traefik的網址
- gitlab2.9kyd.com作為traefik的網址

建立基本檔案和資料夾(於/home/traefikv2-docker-gitlab/)
```
mkdir -p /home/traefikv2-docker-gitlab/traefik/data/configurations
touch /home/traefikv2-docker-gitlab/traefik/docker-compose.yml
touch /home/traefikv2-docker-gitlab/traefik/data/traefik.yml
touch /home/traefikv2-docker-gitlab/traefik/data/acme.json
touch /home/traefikv2-docker-gitlab/traefik/data/configurations/dynamic.yml
chmod 600 /home/traefikv2-docker-gitlab/traefik/data/acme.json
mkdir /home/traefikv2-docker-gitlab/gitlab
touch /home/traefikv2-docker-gitlab/docker-compose.yml
```
<br />
建立docker network<br />

```
docker network create proxy
```

<br />
編輯/home/traefikv2-docker-gitlab/traefik/docker-compose.yml，需要更改其中的traefik.9kyd.com<br />

```
version: '3.7'

services:
  traefik:
    image: traefik:v2.4
    container_name: traefik
    restart: always
    ports:
      - 80:80
      - 443:443
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./data/traefik.yml:/traefik.yml:ro
      - ./data/acme.json:/acme.json
      # Add folder with dynamic configuration yml
      - ./data/configurations:/configurations
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=proxy"
      - "traefik.http.routers.traefik-secure.entrypoints=websecure"
      - "traefik.http.routers.traefik-secure.rule=Host(`traefik.9kyd.com`)"
      - "traefik.http.routers.traefik-secure.middlewares=user-auth@file"
      - "traefik.http.routers.traefik-secure.service=api@internal"

networks:
  proxy:
    external: true
```

編輯/home/traefikv2-docker-gitlab/traefik/data/traefik.yml，需修改leo@9skin.com<br/>

```
api:
  dashboard: true

entryPoints:
  web:
    address: :80
    http:
      redirections:
        entryPoint:
          to: websecure

  websecure:
    address: :443
    http:
      middlewares:
        - secureHeaders@file
        - nofloc@file
      tls:
        certResolver: letsencrypt

pilot:
  dashboard: false

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
  file:
    filename: /configurations/dynamic.yml

certificatesResolvers:
  letsencrypt:
    acme:
      email: leo@9skin.com
      storage: acme.json
      keyType: EC384
      httpChallenge:
        entryPoint: web
```

編輯/home/traefikv2-docker-gitlab/traefik/data/configurations/dynamic.yml，需修改root:$apr1$hn3zvejt$YwPW.pgWVV4p9DTrgZNY./<br/>
此為traefik的登入密碼，可以使用<a href="https://zh-tw.rakko.tools/tools/20/">Htpasswd生成器</a>生成MD5(APR)的密碼<br />

```
# Dynamic configuration
http:
  middlewares:
    nofloc:
      headers:
        customResponseHeaders:
          Permissions-Policy: "interest-cohort=()"
    secureHeaders:
      headers:
        sslRedirect: true
        forceSTSHeader: true
        stsIncludeSubdomains: true
        stsPreload: true
        stsSeconds: 31536000

    #利用Htpasswd生成         
    user-auth:
      basicAuth:
        users:
          - "root:$apr1$hn3zvejt$YwPW.pgWVV4p9DTrgZNY./"

tls:
  options:
    default:
      cipherSuites:
        - TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
        - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
        - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
        - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
        - TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
        - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
      minVersion: VersionTLS12
```

編輯/home/traefikv2-docker-gitlab/gitlab/docker-compose.yml，修改GITLAB_OMNIBUS_CONFIG內的參數，和labels中的domain<br />

```
version: '3'

services:
  gitlab:
    container_name: gitlab
    image: gitlab/gitlab-ce:15.4.5-ce.0
    restart: always
    ports:
      - "2222:22"
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://gitlab2.9kyd.com'
        nginx['listen_https'] = false
        nginx['listen_port'] = 80
        gitlab_rails['smtp_enable'] = true
        gitlab_rails['smtp_address'] = 'smtp.gmail.com'
        gitlab_rails['smtp_port'] = 465
        gitlab_rails['smtp_user_name'] = '9skinmailer@gmail.com'
        gitlab_rails['smtp_password'] = '12345678'
        gitlab_rails['smtp_domain'] = 'gmail.com'
        gitlab_rails['smtp_authentication'] = 'login'
        gitlab_rails['smtp_enable_starttls_auto'] = true
        gitlab_rails['smtp_tls'] = false
        gitlab_rails['smtp_openssl_verify_mode'] = 'peer'
        gitlab_rails['gitlab_shell_ssh_port'] = 2222
    volumes:
      - ./volumes/config:/etc/gitlab
      - ./volumes/logs:/var/log/gitlab
      - ./volumes/data:/var/opt/gitlab
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=proxy"
      - "traefik.http.routers.nextcloud-secure.entrypoints=websecure"
      - "traefik.http.routers.nextcloud-secure.rule=Host(`gitlab2.9kyd.com`)"
      - "traefik.http.routers.nextcloud-secure.service=nextcloud-service"
      - "traefik.http.services.nextcloud-service.loadbalancer.server.port=80" 
networks:
  proxy:
    external: true
```

gitlab的root帳號要進入container內設定<br/>

```
gitlab-rake "gitlab:password:reset[root]"
```

參考網站 : https://hexo.aufomm.com/traefik/
