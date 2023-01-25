# Kong의 특징
 
- Kubernetes-Native
  - 공식 Ingress Controller를 사용하여 모든 L4 + L7 트래픽을 라우팅하고 연결하는 네이티브 Kubernetes CRD로 Kong을 선언적으로 구성한다.
- 동적 부하 분산
  - 여러 업스트림 서비스에서 트래픽 부하를 분산한다.
- 해시 기반 부하 분산
  - 일관된 해싱 / 고정 세션으로 부하 분산
- Circuit-Breaker
  - 비정상 업스트림 서비스를 지능적으로 추적한다.
- 상태 확인
  - 업스트림 서비스의 능동 및 수동 모니터링
- 서비스 검색
  - Consul과 같은 타사 DNS 확인자에서 SRV 레코드를 확인한다.
- 서버리스
  - Kong에서 직접 AWS Lambda 또는 OpenWhisk 함수를 호출하고 보호한다.
- Websockets
  - WebSocket을 통해 업스트림 서비스와 통신한다.
- gRPC
  - gRPC 서비스와 통신하고 로깅 및 관찰 플러그인으로 트래픽을 관찰한다.
- 로깅
  - HTTP, TCP, UDP 또는 디스크를 통해 시스템에 대한 요청 및 응답을 기록한다. 그리고 시스템 로그도 기록한다.
- 보안
  - ACL, 봇 감지, IP 허용 /거부 등
- SSL
  - 기본 서비스 또는 API에 대한 특정 SSL 인증서를 설정한다.
- 모니터링
  - 실시간 모니터링은 주요로드 및 성능 서버 메트릭을 제공한다.
- 전달 프록시
  - Kong이 중간 투명 HTTP 프록시에 연결되도록 한다.
- 인증
  - HMAC, JWT, Basic 등 ... API에 OAuth 2.0 인증을 쉽게 추가한다.
- 속도제한
  - 많은 변수를 기반으로 요청을 차단하고 제한한다.
- 변환
  - HTTP 요청 및 응답을 추가, 제거 또는 조작한다.
- 캐싱
  - 프록시 계층에서 응답을 캐시하고 제공한다.
- CLI
  - 명령 줄에 Kong 클러스터를 제어한다.
- REST API
  - Kong은 최대한의 유연성을 위해 RESTful API로 작동할 수 있다.
- 지역 복제
  - 구성은 항상 여러 지역에서 최신 상태로 유지된다.
- 실패 감지 및 복구
  - Cassandra 노드 중 하나가 다운 되더라도 Kong은 영향을 받지 않는다.
- 클러스터링
  - 모든 Kong 노드는 클러스터에 자동 가입하여 노드 전체에서 구성을 업데이트 한다.
- 확장성
  - 본질적으로 분산된 Kong은 단순히 노드를 추가하여 수평 적으로 확장된다.
- 성능
  - Kong은 핵심에서 NGINX를 확장하고 사용하여 로드를 쉽게 처리한다.
- 플러그인
  - Kong 및 API에 기능을 추가하기 위한 확장 가능한 아키텍처이다. 

# Kong 설치 및 구성
### docker network 생성
docker를 이용하기 때문에 DB, Kong 컨테이너 간 통신을 위해 docker network를 생성하여 네트워크를 구성한다.
```shell
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/Kong_Configuration   master ±✚  docker network create kong-net
88a8a1853917bd73f1d2182fd90d1ca4882ffd6d34e5c6964ed652057bbb899e

 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/Kong_Configuration   master ±✚  docker network ls
NETWORK ID     NAME               DRIVER    SCOPE
c7973a35feec   bridge             bridge    local
78e56a36f72e   codevasp_default   bridge    local
353d6a7abcdb   docker_default     bridge    local
cf983a5cf8b6   host               host      local
88a8a1853917   kong-net           bridge    local
43d8c27a5728   minikube           bridge    local
af98312b01de   none               null      local
```

### Database 설치/구동
Kong은 현재 Cassandra와 PostgreSQL을 지원한다. 
```shell
docker run -d --name kong-database --network=kong-net -p 5432:5432 \
                   -e "POSTGRES_USER=kong" \
                   -e "POSTGRES_DB=kong" \
                   -e "POSTGRES_PASSWORD=minsoub123" \
                   postgres:9.6
                   
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/Kong_Configuration   master ±✚  docker run -d --name kong-database --network=kong-net -p 5432:5432 -e "POSTGRES_USER=kong" -e "POSTGRES_DB=kong" -e "POSTGRES_PASSWORD=minsoub123" postgres:9.6
Unable to find image 'postgres:9.6' locally
9.6: Pulling from library/postgres
0baeb81232d0: Pull complete 
5d0d2feedd82: Pull complete 
c04fdad564a5: Pull complete 
6e5ce0e066d5: Pull complete 
60d75bd410a9: Pull complete 
79ec45cd92a9: Pull complete 
9f5092dfc5d2: Pull complete 
9c8c27d05cfd: Pull complete 
99e77834409f: Pull complete 
77e96ba4658e: Pull complete 
f51113625a02: Pull complete 
b87ba8efb1f1: Pull complete 
501952e572e6: Pull complete 
29a08936e34c: Pull complete 
Digest: sha256:caddd35b05cdd56c614ab1f674e63be778e0abdf54e71a7507ff3e28d4902698
Status: Downloaded newer image for postgres:9.6
89f5fd541c3b772655f7d4ecd5849de7792047638905d293d1fedc78b5f06d83
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/Kong_Configuration   master ±✚  docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                    NAMES
89f5fd541c3b   postgres:9.6   "docker-entrypoint.s…"   56 seconds ago   Up 55 seconds   0.0.0.0:5432->5432/tcp   kong-database                   
```
### Database Setting
PostgreSQL에 kong의 bootstrap 환경을 migration 하여 DB 스키마를 최초 설정한다. 
```shell
docker run --rm  --network=kong-net \
                -e "KONG_DATABASE=postgres" \
                -e "KONG_PG_HOST=kong-database" \
                -e "KONG_PG_PASSWORD=minsoub123" \
                -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
                kong:latest kong migrations bootstrap
                
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/Kong_Configuration   master ±✚  docker run --rm  --network=kong-net \
                -e "KONG_DATABASE=postgres" \
                -e "KONG_PG_HOST=kong-database" \
                -e "KONG_PG_PASSWORD=minsoub123" \
                -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
                kong:latest kong migrations bootstrap
Bootstrapping database...
migrating core on database 'kong'...                
```

### Kong 설치 및 구동
DB 컨테이너와 연결하여 구동한다.
```shell
docker run -d --name kong --network=kong-net \
     -e "KONG_DATABASE=postgres" \
     -e "KONG_PG_HOST=kong-database" \
     -e "KONG_PG_PASSWORD=minsoub123" \
     -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
     -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
     -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
     -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
     -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
     -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
     -p 8000:8000 \
     -p 8443:8443 \
     -p 127.0.0.1:8001:8001 \
     -p 127.0.0.1:8444:8444 \
     kong:latest
     
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/Kong_Configuration   master ±✚  docker run -d --name kong --network=kong-net \
     -e "KONG_DATABASE=postgres" \
     -e "KONG_PG_HOST=kong-database" \
     -e "KONG_PG_PASSWORD=minsoub123" \
     -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
     -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
     -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
     -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
     -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
     -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
     -p 8000:8000 \
     -p 8443:8443 \
     -p 127.0.0.1:8001:8001 \
     -p 127.0.0.1:8444:8444 \
     kong:latest
0858ceb81774e5026982b105ead1e15a8a741f90953ddf4e57052cc7e045c9ee
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/Kong_Configuration   master ±✚  docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS                            PORTS                                                                                                NAMES
0858ceb81774   kong:latest    "/docker-entrypoint.…"   5 seconds ago   Up 4 seconds (health: starting)   0.0.0.0:8000->8000/tcp, 127.0.0.1:8001->8001/tcp, 0.0.0.0:8443->8443/tcp, 127.0.0.1:8444->8444/tcp   kong
43a82ea871f2   postgres:9.6   "docker-entrypoint.s…"   4 minutes ago   Up 4 minutes                      0.0.0.0:5432->5432/tcp                                                                               kong-database
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/Kong_Configuration   master ±✚       
```
### Kong 구동 확인
```shell
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/Kong_Configuration   master ±✚  curl -I http://localhost:8001
HTTP/1.1 200 OK
Date: Sat, 14 Jan 2023 11:53:03 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: *
Content-Length: 13169
X-Kong-Admin-Latency: 46
Server: kong/3.1.1
```

### Konga 설치 (Kong GUI 관리 툴)
#### Database Setting
Konga의 Bootstrap 환경을 위해 DB migration이 필요하다. network는 위에서 사용한 kong-net, DB 접속 정보 역시 kong-database로 입력하여 prepare 명령을 실행한다.
```shell
docker run --rm --network=kong-net pantsel/konga:latest -c prepare -a 'postgres' -u postgresql://kong:minsoub123@kong-database:5432/konga

 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/Kong_Configuration   master ±✚  docker run --rm --network=kong-net pantsel/konga:latest -c prepare -a 'postgres' -u postgresql://kong:minsoub123@kong-database:5432/konga
Unable to find image 'pantsel/konga:latest' locally
latest: Pulling from pantsel/konga
cbdbe7a5bc2a: Pull complete 
8f3938f7d3bd: Pull complete 
6e3c12f5dc10: Pull complete 
ce0cb7a9eeee: Pull complete 
a87657869d4f: Pull complete 
891b0102e38b: Pull complete 
Digest: sha256:c8172b75607d06d83d917387a2f4d95b9b855f64063ee60db8e6f1a1c97b8abf
Status: Downloaded newer image for pantsel/konga:latest
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested
debug: Preparing database...
Using postgres DB Adapter.
Database `konga` does not exist. Creating...
Database `konga` created! Continue...
debug: Hook:api_health_checks:process() called
debug: Hook:health_checks:process() called
debug: Hook:start-scheduled-snapshots:process() called
debug: Hook:upstream_health_checks:process() called
debug: Hook:user_events_hook:process() called
debug: Seeding User...
debug: User seed planted
debug: Seeding Kongnode...
debug: Kongnode seed planted
debug: Seeding Emailtransport...
debug: Emailtransport seed planted
debug: Database migrations completed!
```
#### Konga 실행
```shell
docker run -p 1337:1337 --network=kong-net -e "TOKEN_SECRET=minsoub@gmail.com" -e "DB_ADAPTER=postgres" -e "DB_URI=postgresql://kong:minsoub123@kong-database:5432/konga" -e "NODE_ENV=production" --name konga pantsel/konga
```

## zipkin 설치
```shell
docker run -d -p 9411:9411 openzipkin/zipkin
```

### zipkin plugin configuration
- https://docs.konghq.com/hub/kong-inc/zipkin/
```shell
curl -X POST http://localhost:8001/services/SERVICE_NAME|SERVICE_ID/plugins \
    --data "name=zipkin"  \
    --data "config.http_endpoint=http://your.zipkin.collector:9411/api/v2/spans" \
    --data "config.sample_ratio=0.001" \
    --data "config.include_credential=true"
    
curl -X POST http://localhost:8001/services/Notice-service/plugins \
    --data "name=zipkin"  \
    --data "config.http_endpoint=http://localhost:9411/api/v2/spans" \
    --data "config.sample_ratio=1.0" \
    --data "config.include_credential=true"    
    
    
{"route":null,"tags":null,"service":{"id":"0f5466d5-a1ae-4bd2-a142-0cdb17312370"},"protocols":["grpc","grpcs","http","https"],"created_at":1673849074,"consumer":null,"enabled":true,"name":"zipkin","config":{"header_type":"preserve","traceid_byte_count":16,"http_response_header_for_traceid":null,"http_span_name":"method","connect_timeout":2000,"send_timeout":5000,"read_timeout":5000,"static_tags":null,"sample_ratio":1,"tags_header":"Zipkin-Tags","default_header_type":"b3","http_endpoint":"http://localhost:9411/api/v2/spans","default_service_name":null,"local_service_name":"kong","include_credential":true},"id":"42947b1a-bc0e-45ad-9789-a75b8804425d"}%       
```