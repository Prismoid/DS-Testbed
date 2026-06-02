## Dockerの設定(Caddyの設定込み)

`Dockerfile`
```Dockerfile
FROM node:22-alpine

WORKDIR /app
COPY package.json .
RUN npm install

COPY app.js .
CMD ["node", "app.js"]
```

`docker-compose.yml`
```
services:
  caddy:
    image: caddy:2
    container_name: caddy
    restart: unless-stopped
    ports:
      - "31515:31515/tcp"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./certs:/certs:ro
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - app
    networks:
      - internal
      
  app:
    build: .
    ports:
      - "8080:3000"
    volumes:
      - ./data:/app/data
    environment:
      ADMIN_USER: admin
      ADMIN_PASS: change-me
    networks:
      - internal

networks:
  internal:

volumes:
  caddy_data:
  caddy_config: 
```

`Caddyfile`
```
https://domain-request.koshizukalab.dataspace.internal:31515 {
    tls /certs/drf.crt /certs/drf.key
    reverse_proxy app:3000 {
        header_up Host {host}
        header_up X-Forwarded-Proto https
        header_up X-Forwarded-Host {host}
        header_up X-Forwarded-Port 32443
    }
}
```

./certsのディレクトリに入れる秘密鍵と証明書(.crt)ファイルについて

```
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
-keyout drf.key \
-out drf.crt \
-subj "/CN=domain-request.koshizukalab.dataspace.internal" \
-addext "subjectAltName=DNS:domain-request.koshizukalab.dataspace.internal"
```
