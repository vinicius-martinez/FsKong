# Kong API Gateway

Neste repositório estarão disponíveis nosso *Workshop* de implementação fazendo uso da tecnologia [Kong API Gateway](https://konghq.com/kong/)

## Pré Requisitos

- [Docker Desktop Win/Mac 3.x](https://www.docker.com/products/docker-desktop)

## Workshop

0. [Instalação](#workshop-kong-install)
1. [Expor API](#workshop-kong-api)
2. [Rate Limit](#workshop-kong-ratelimit)
3. [Proxy Caching](#workshop-kong-caching)
4. [Key Authentication](#workshop-kong-keyauth)

## Implementação

### 0 - Instalação <a name="workshop-kong-install">

* Realizar o *pull* da imagem do gateway e fazer o *docker tag* da mesma para facilitar execução:

  ```
  docker pull kong-docker-kong-gateway-docker.bintray.io/kong-enterprise-edition:2.3.2.0-alpine
  docker tag <IMAGE_ID> kong-ee
  ```

* Criar uma *docker newtwork* para comunicação dos *containers*: `docker network create kong-ee-net`

* Realizar o deployment do *Database*:

  ```
  docker run -d --name kong-ee-database \
    --network=kong-ee-net \
    -p 5432:5432 \
    -e "POSTGRES_USER=kong" \
    -e "POSTGRES_DB=kong" \
    -e "POSTGRES_PASSWORD=kong" \
    postgres:9.6

  docker run --rm --network=kong-ee-net \
    -e "KONG_DATABASE=postgres" \
    -e "KONG_PG_HOST=kong-ee-database" \
    -e "KONG_PG_PASSWORD=kong" \
    -e "KONG_PASSWORD=kong" \
    kong-ee kong migrations bootstrap
  ```

* Realizar o deployment do **API Gateway**

  ```
  docker run -d --name kong-ee --network=kong-ee-net \
    -e "KONG_DATABASE=postgres" \
    -e "KONG_PG_HOST=kong-ee-database" \
    -e "KONG_PG_PASSWORD=kong" \
    -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
    -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
    -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_LISTEN=0.0.0.0:8001" \
    -e "KONG_PORTAL_GUI_HOST=0.0.0.0:8003" \
    -e "KONG_ADMIN_GUI_URL=http://0.0.0.0:8002" \
    -p 8000:8000 \
    -p 8443:8443 \
    -p 8001:8001 \
    -p 8444:8444 \
    -p 8002:8002 \
    -p 8445:8445 \
    -p 8003:8003 \
    -p 8004:8004 \
    kong-ee
  ```

* Validar se a instalação foi bem-sucedida: `curl -i -X GET --url http://0.0.0.0:8001/services`

* Acessar o console administrativo: `http://localhost:8002/overview`


### 1 - Expor API <a name="workshop-kong-api">

* Criar um [Service](https://docs.konghq.com/getting-started-guide/2.3.x/expose-services/#what-are-services-and-routes) apontando para o *mockbin.org*:

  ```
  curl -i -X POST http://localhost:8001/services \
  --data name=mockbin_service \
  --data url='http://mockbin.org'

  HTTP/1.1 201 Created
  Date: Thu, 25 Feb 2021 19:16:08 GMT
  Content-Type: application/json; charset=utf-8
  Connection: keep-alive
  Access-Control-Allow-Origin: http://0.0.0.0:8002
  X-Kong-Admin-Request-ID: 6gruZKZUL2qSx5CGjaK7vph3gSANEVpC
  vary: Origin
  Access-Control-Allow-Credentials: true
  Content-Length: 361
  X-Kong-Admin-Latency: 47
  Server: kong/2.3.2.0-enterprise-edition

  {"host":"mockbin.org","id":"e22daacd-9d9c-4cb5-9e7c-b797f6857611","protocol":"http","read_timeout":60000,"tls_verify_depth":null,"port":80,"updated_at":1614280568,"ca_certificates":null,"created_at":1614280568,"connect_timeout":60000,"write_timeout":60000,"name":"mockbin_service","retries":5,"path":null,"tls_verify":null,"tags":null,"client_certificate":null}%
  ```

* Validar se o **Service** foi criado:

  ```
  curl -i http://localhost:8001/services/mockbin_service
  HTTP/1.1 200 OK
  Date: Thu, 25 Feb 2021 19:17:06 GMT
  Content-Type: application/json; charset=utf-8
  Connection: keep-alive
  Access-Control-Allow-Origin: http://0.0.0.0:8002
  X-Kong-Admin-Request-ID: rPnInzeDTEyYFdRpOSoZVwylKLCSTw3l
  vary: Origin
  Access-Control-Allow-Credentials: true
  Content-Length: 361
  X-Kong-Admin-Latency: 11
  Server: kong/2.3.2.0-enterprise-edition

  {"host":"mockbin.org","id":"e22daacd-9d9c-4cb5-9e7c-b797f6857611","protocol":"http","read_timeout":60000,"tls_verify_depth":null,"port":80,"updated_at":1614280568,"ca_certificates":null,"created_at":1614280568,"connect_timeout":60000,"write_timeout":60000,"name":"mockbin_service","retries":5,"path":null,"tls_verify":null,"tags":null,"client_certificate":null}%
  ```

* Criar uma **Route** para esse **Service**:

  ```
  curl -i -X POST http://localhost:8001/services/mockbin_service/routes \
  --data 'paths[]=/mock' \
  --data name=mocking

  HTTP/1.1 201 Created
  Date: Thu, 25 Feb 2021 19:19:40 GMT
  Content-Type: application/json; charset=utf-8
  Connection: keep-alive
  Access-Control-Allow-Origin: http://0.0.0.0:8002
  X-Kong-Admin-Request-ID: QSrlD34fHoZ5dsj24kRQQtnPLa4LLZti
  vary: Origin
  Access-Control-Allow-Credentials: true
  Content-Length: 479
  X-Kong-Admin-Latency: 34
  Server: kong/2.3.2.0-enterprise-edition

  {"strip_path":true,"tags":null,"updated_at":1614280780,"destinations":null,"headers":null,"protocols":["http","https"],"methods":null,"service":{"id":"e22daacd-9d9c-4cb5-9e7c-b797f6857611"},"snis":null,"hosts":null,"name":"mocking","path_handling":"v0","paths":["/mock"],"preserve_host":false,"regex_priority":0,"response_buffering":true,"sources":null,"id":"c418aebf-2016-4495-b077-f2a850de3635","https_redirect_status_code":426,"request_buffering":true,"created_at":1614280780}%
  ```

* Validar se a *API* está funcionando adequadamente:

  ```
  curl -i -X GET http://localhost:8000/mock/request

  HTTP/1.1 200 OK
  Content-Type: application/json; charset=utf-8
  Transfer-Encoding: chunked
  Connection: keep-alive
  Date: Thu, 25 Feb 2021 19:21:15 GMT
  Set-Cookie: __cfduid=dd400578e5767214224db64a709a6740b1614280875; expires=Sat, 27-Mar-21 19:21:15 GMT; path=/; domain=.mockbin.org; HttpOnly; SameSite=Lax
  Access-Control-Allow-Origin: *
  Access-Control-Allow-Methods: GET
  Access-Control-Allow-Headers: host,connection,accept-encoding,x-forwarded-for,cf-ray,x-forwarded-proto,cf-visitor,x-forwarded-host,x-forwarded-port,x-forwarded-path,x-forwarded-prefix,user-agent,accept,cf-connecting-ip,cdn-loop,cf-request-id,x-request-id,via,connect-time,x-request-start,total-route-time
  Access-Control-Allow-Credentials: true
  X-Powered-By: mockbin
  Vary: Accept, Accept-Encoding
  Etag: W/"46d-MkY6jKql00m1qEPgudIBoq9anlI"
  Via: kong/2.3.2.0-enterprise-edition
  CF-Cache-Status: DYNAMIC
  cf-request-id: 087c3c76240000f724043f7000000001
  Report-To: {"endpoints":[{"url":"https:\/\/a.nel.cloudflare.com\/report?s=DldoU%2FY88t48KtXuFSMZjdsattcwYWuPPYunQDwA7tkCYECJdUcFseRQRXDz5dc9eJw371bKvyj9%2BpJm5gKYuytdS8l5SEefJXw5OA%3D%3D"}],"group":"cf-nel","max_age":604800}
  NEL: {"report_to":"cf-nel","max_age":604800}
  Server: cloudflare
  CF-RAY: 6273c9d03fa3f724-GRU
  X-Kong-Upstream-Latency: 302
  X-Kong-Proxy-Latency: 286

  {
    "startedDateTime": "2021-02-25T19:21:15.752Z",
    "clientIPAddress": "172.18.0.1",
    "method": "GET",
    "url": "http://localhost/request",
    "httpVersion": "HTTP/1.1",
    "cookies": {},
    "headers": {
      "host": "mockbin.org",
      "connection": "close",
      "accept-encoding": "gzip",
      "x-forwarded-for": "172.18.0.1,187.2.139.101, 172.68.26.116",
      "cf-ray": "6273c9d03fa3f724-GRU",
      "x-forwarded-proto": "http",
      "cf-visitor": "{\"scheme\":\"http\"}",
      "x-forwarded-host": "localhost",
      "x-forwarded-port": "80",
      "x-forwarded-path": "/mock/request",
      "x-forwarded-prefix": "/mock",
      "user-agent": "curl/7.64.1",
      "accept": "*/*",
      "cf-connecting-ip": "187.2.139.101",
      "cdn-loop": "cloudflare",
      "cf-request-id": "087c3c76240000f724043f7000000001",
      "x-request-id": "e2816311-2693-4bed-83e0-6ea5e62ac816",
      "via": "1.1 vegur",
      "connect-time": "0",
      "x-request-start": "1614280875746",
      "total-route-time": "0"
    },
    "queryString": {},
    "postData": {
      "mimeType": "application/octet-stream",
      "text": "",
      "params": []
    },
    "headersSize": 637,
    "bodySize": 0
  }%
  ```

### 2 - Rate Limit <a name="workshop-kong-ratelimit">

* Habilitar o [Rate Limiting](https://docs.konghq.com/hub/kong-inc/rate-limiting/) na **Route** recém criada:

  ```
  curl -i -X POST http://localhost:8001/routes/mocking/plugins \
  --data name=rate-limiting \
  --data config.minute=5 \
  --data config.policy=local

  HTTP/1.1 201 Created
  Date: Thu, 25 Feb 2021 19:34:28 GMT
  Content-Type: application/json; charset=utf-8
  Connection: keep-alive
  Access-Control-Allow-Origin: http://0.0.0.0:8002
  X-Kong-Admin-Request-ID: qRCepFvsS52wGleQW59rBTzthE7ZLHDz
  vary: Origin
  Access-Control-Allow-Credentials: true
  Content-Length: 550
  X-Kong-Admin-Latency: 29
  Server: kong/2.3.2.0-enterprise-edition

  {"created_at":1614281668,"id":"b8a97a0c-5759-4c03-b486-5ab8d92b3b5f","tags":null,"enabled":true,"protocols":["grpc","grpcs","http","https"],"name":"rate-limiting","consumer":null,"service":null,"route":{"id":"c418aebf-2016-4495-b077-f2a850de3635"},"config":{"minute":5,"redis_host":null,"redis_timeout":2000,"limit_by":"consumer","hour":null,"policy":"local","month":null,"redis_password":null,"second":null,"day":null,"hide_client_headers":false,"path":null,"redis_database":0,"year":null,"redis_port":6379,"header_name":null,"fault_tolerant":true}}
  ```

* Testar o rate limit: `while true; do http :8000/mock/request; sleep 1;  done`

### 3 - Proxy Caching <a name="workshop-kong-caching">

* Definir globalmente uma política de *caching*:

  ```
  curl -i -X POST http://localhost:8001/plugins \
  --data name=proxy-cache \
  --data config.content_type="application/json; charset=utf-8" \
  --data config.cache_ttl=30 \
  --data config.strategy=memory

  HTTP/1.1 201 Created
  Date: Thu, 25 Feb 2021 19:40:56 GMT
  Content-Type: application/json; charset=utf-8
  Connection: keep-alive
  Access-Control-Allow-Origin: http://0.0.0.0:8002
  X-Kong-Admin-Request-ID: 5NG0YDYelKLh3Q5Zq1Bx1tmYRUhKteDU
  vary: Origin
  Access-Control-Allow-Credentials: true
  Content-Length: 495
  X-Kong-Admin-Latency: 17
  Server: kong/2.3.2.0-enterprise-edition

  {"created_at":1614282056,"id":"3b94b961-2bc3-4b6c-970a-3134fc698a0d","tags":null,"enabled":true,"protocols":["grpc","grpcs","http","https"],"name":"proxy-cache","consumer":null,"service":null,"route":null,"config":{"request_method":["GET","HEAD"],"cache_control":false,"vary_query_params":null,"storage_ttl":null,"response_code":[200,301,404],"cache_ttl":30,"vary_headers":null,"strategy":"memory","content_type":["application/json; charset=utf-8"],"memory":{"dictionary_name":"kong_db_cache"}}}
  ```

* Invocar o serviço duas vezes: `http :8000/mock/request`

* Avalie o *output* dos headers: **X-Cache-Key, X-Cache-Status, X-Content-Type-Options, X-Kong-Proxy-Latency, X-Kong-Upstream-Latency**

  * caso queira limpar o cache execute: `curl -i -X DELETE http://localhost:8001/proxy-cache`

### 4 - Key Authentication <a name="workshop-kong-keyauth">

* Habilite o plugin [Key Authentication](https://docs.konghq.com/hub/kong-inc/key-auth/)

  ```
  http :8001/routes/mocking/plugins \
    name=key-auth

  HTTP/1.1 201 Created
  Access-Control-Allow-Credentials: true
  Access-Control-Allow-Origin: http://0.0.0.0:8002
  Connection: keep-alive
  Content-Length: 404
  Content-Type: application/json; charset=utf-8
  Date: Thu, 25 Feb 2021 19:57:23 GMT
  Server: kong/2.3.2.0-enterprise-edition
  X-Kong-Admin-Latency: 12
  X-Kong-Admin-Request-ID: V1kEnXrKcE9FZm8ZlAUKONhK4zfMU5Cy
  vary: Origin

  {
      "config": {
          "anonymous": null,
          "hide_credentials": false,
          "key_in_body": false,
          "key_in_header": true,
          "key_in_query": true,
          "key_names": [
              "apikey"
          ],
          "run_on_preflight": true
      },
      "consumer": null,
      "created_at": 1614283043,
      "enabled": true,
      "id": "7bc02de1-9d96-4c26-bc42-c1452d743a15",
      "name": "key-auth",
      "protocols": [
          "grpc",
          "grpcs",
          "http",
          "https"
      ],
      "route": {
          "id": "c418aebf-2016-4495-b077-f2a850de3635"
      },
      "service": null,
      "tags": null
  }
  ```

* Tente acessar o serviço novamente:

  ```
  http :8080/mock

  HTTP/1.1 401 Unauthorized
  Connection: keep-alive
  Content-Length: 45
  Content-Type: application/json; charset=utf-8
  Date: Thu, 25 Feb 2021 19:58:32 GMT
  Server: kong/2.3.2.0-enterprise-edition
  WWW-Authenticate: Key realm="kong"
  X-Kong-Response-Latency: 108

  {
      "message": "No API key found in request"
  }
  ```

* Criar um **Consumer** para a *API*:

  ```
  curl -i -X POST http://localhost:8001/consumers/ \
  --data username=consumer \
  --data custom_id=consumer

  HTTP/1.1 201 Created
  Date: Thu, 25 Feb 2021 20:10:40 GMT
  Content-Type: application/json; charset=utf-8
  Connection: keep-alive
  Access-Control-Allow-Origin: http://0.0.0.0:8002
  X-Kong-Admin-Request-ID: 99wcolamSfewAjHAH3pemLH0FTtBEb3s
  vary: Origin
  Access-Control-Allow-Credentials: true
  vary: Origin
  Content-Length: 135
  X-Kong-Admin-Latency: 33
  Server: kong/2.3.2.0-enterprise-edition

  {"custom_id":"consumer","created_at":1614283840,"id":"c889532e-f1f2-44af-81b8-51400e48cf89","tags":null,"username":"consumer","type":0}
  ```

* Crie uma **key** para o **Consumer** recém criado:

  ```
  http :8001/consumers/consumer/key-auth \
  key=apikey

  HTTP/1.1 201 Created
  Access-Control-Allow-Credentials: true
  Access-Control-Allow-Origin: http://0.0.0.0:8002
  Connection: keep-alive
  Content-Length: 164
  Content-Type: application/json; charset=utf-8
  Date: Thu, 25 Feb 2021 20:13:13 GMT
  Server: kong/2.3.2.0-enterprise-edition
  X-Kong-Admin-Latency: 7
  X-Kong-Admin-Request-ID: 92w7OoT4Mx6AYqarQBW1pXShcJOwbJ6I
  vary: Origin

  {
      "consumer": {
          "id": "c889532e-f1f2-44af-81b8-51400e48cf89"
      },
      "created_at": 1614283993,
      "id": "e792a52e-cdc5-4eaf-a49c-3b10fad4ee40",
      "key": "apikey",
      "tags": null,
      "ttl": null
  }
  ```

* Agora faça uma nova requisição contendo a **Key**:

  ```
  http :8000/mock/request apikey:apikey
  HTTP/1.1 200 OK
  Access-Control-Allow-Credentials: true
  Access-Control-Allow-Headers: host,connection,accept-encoding,x-forwarded-for,cf-ray,x-forwarded-proto,cf-visitor,x-forwarded-host,x-forwarded-port,x-forwarded-path,x-forwarded-prefix,user-agent,accept,apikey,x-consumer-id,x-consumer-custom-id,x-consumer-username,x-credential-identifier,cf-connecting-ip,cdn-loop,cf-request-id,x-request-id,via,connect-time,x-request-start,total-route-time
  Access-Control-Allow-Methods: GET
  Access-Control-Allow-Origin: *
  CF-Cache-Status: DYNAMIC
  CF-RAY: 627419027e9951e6-GRU
  Connection: keep-alive
  Content-Encoding: gzip
  Content-Type: application/json; charset=utf-8
  Date: Thu, 25 Feb 2021 20:15:19 GMT
  Etag: W/"559-k9yF/flQkjStFei3ldw9JYafuQY"
  NEL: {"report_to":"cf-nel","max_age":604800}
  RateLimit-Limit: 5
  RateLimit-Remaining: 4
  RateLimit-Reset: 41
  Report-To: {"endpoints":[{"url":"https:\/\/a.nel.cloudflare.com\/report?s=CaQ1Cz82G5Nn%2Ft4bhtViWqLVaTE47jcfm59ob0R5d3CCg7jTAR%2FqTbmYFs0u1RpxUpvIv1oEiOeVWetfAXhl9qkZsliLAs1fXWkfiA%3D%3D"}],"max_age":604800,"group":"cf-nel"}
  Server: cloudflare
  Set-Cookie: __cfduid=d10e6bb58b8227938a245f4502c20dbde1614284119; expires=Sat, 27-Mar-21 20:15:19 GMT; path=/; domain=.mockbin.org; HttpOnly; SameSite=Lax
  Transfer-Encoding: chunked
  Vary: Accept, Accept-Encoding
  Via: kong/2.3.2.0-enterprise-edition
  X-Cache-Key: 400e1aa41ad36470eb75463af8d52e53
  X-Cache-Status: Miss
  X-Kong-Proxy-Latency: 108
  X-Kong-Upstream-Latency: 328
  X-Powered-By: mockbin
  X-RateLimit-Limit-Minute: 5
  X-RateLimit-Remaining-Minute: 4
  cf-request-id: 087c6df589000051e687bc5000000001

  {
      "bodySize": 0,
      "clientIPAddress": "172.18.0.1",
      "cookies": {},
      "headers": {
          "accept": "*/*",
          "accept-encoding": "gzip",
          "apikey": "apikey",
          "cdn-loop": "cloudflare",
          "cf-connecting-ip": "187.2.139.101",
          "cf-ray": "627419027e9951e6-GRU",
          "cf-request-id": "087c6df589000051e687bc5000000001",
          "cf-visitor": "{\"scheme\":\"http\"}",
          "connect-time": "1",
          "connection": "close",
          "host": "mockbin.org",
          "total-route-time": "0",
          "user-agent": "HTTPie/2.3.0",
          "via": "1.1 vegur",
          "x-consumer-custom-id": "consumer",
          "x-consumer-id": "c889532e-f1f2-44af-81b8-51400e48cf89",
          "x-consumer-username": "consumer",
          "x-credential-identifier": "e792a52e-cdc5-4eaf-a49c-3b10fad4ee40",
          "x-forwarded-for": "172.18.0.1,187.2.139.101, 172.68.24.248",
          "x-forwarded-host": "localhost",
          "x-forwarded-path": "/mock/request",
          "x-forwarded-port": "80",
          "x-forwarded-prefix": "/mock",
          "x-forwarded-proto": "http",
          "x-request-id": "a381a843-c608-43c0-8138-9caf144e214a",
          "x-request-start": "1614284119648"
      },
      "headersSize": 833,
      "httpVersion": "HTTP/1.1",
      "method": "GET",
      "postData": {
          "mimeType": "application/octet-stream",
          "params": [],
          "text": ""
      },
      "queryString": {},
      "startedDateTime": "2021-02-25T20:15:19.651Z",
      "url": "http://localhost/request"
  }
  ```

  * para desabilitar o **plugin** execute:

    ```
    http :8001/routes/mocking/plugins

    {
    "config": {
        "anonymous": null,
        "hide_credentials": false,
        "key_in_body": false,
        "key_in_header": true,
        "key_in_query": true,
        "key_names": [
            "apikey"
        ],
        "run_on_preflight": true
    },
    "consumer": null,
    "created_at": 1614283043,
    "enabled": true,
    "id": "7bc02de1-9d96-4c26-bc42-c1452d743a15",
    "name": "key-auth",
    "protocols": [
        "grpc",
        "grpcs",
        "http",
        "https"
    ],
    "route": {
        "id": "c418aebf-2016-4495-b077-f2a850de3635"
    },
    "service": null,
    "tags": null
    }

    http -f patch :8001/routes/mocking/plugins/{<plugin-id>} \
      enabled=false

    http -f patch :8001/routes/mocking/plugins/7bc02de1-9d96-4c26-bc42-c1452d743a15 \
      enabled=false

      HTTP/1.1 200 OK
      Access-Control-Allow-Credentials: true
      Access-Control-Allow-Origin: http://0.0.0.0:8002
      Connection: keep-alive
      Content-Length: 405
      Content-Type: application/json; charset=utf-8
      Date: Thu, 25 Feb 2021 20:19:35 GMT
      Server: kong/2.3.2.0-enterprise-edition
      X-Kong-Admin-Latency: 13
      X-Kong-Admin-Request-ID: qI5VvEWeKGqdQ2ngKHJlWMNYF0NoXZJf
      vary: Origin

      {
          "config": {
              "anonymous": null,
              "hide_credentials": false,
              "key_in_body": false,
              "key_in_header": true,
              "key_in_query": true,
              "key_names": [
                  "apikey"
              ],
              "run_on_preflight": true
          },
          "consumer": null,
          "created_at": 1614283043,
          "enabled": false,
          "id": "7bc02de1-9d96-4c26-bc42-c1452d743a15",
          "name": "key-auth",
          "protocols": [
              "grpc",
              "grpcs",
              "http",
              "https"
          ],
          "route": {
              "id": "c418aebf-2016-4495-b077-f2a850de3635"
          },
          "service": null,
          "tags": null
      }

    http :8000/mock/request
    ```
