# 使用traefikv2架設配合Let's Encrypt帶有SSL的Gitlab
這邊利用docker-compose進行架設
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
