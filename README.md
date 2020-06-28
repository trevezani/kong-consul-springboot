# kong-consul-springboot

The full example is present in [microservices-tutorial-springboot](https://github.com/trevezani/microservices-tutorial-springboot)

***

## Building, Testing and Running

* building the microservice:
```
mvn clean install -f service/parent
mvn clean install -f service/internal-shared
mvn clean package -Pconsul -f service/census-zipcode

mvn docker:build -f service/census-zipcode/census-zipcode-infrastructure
```
* running consul and kong:
```
docker network create --driver=bridge --subnet=81.20.0.0/21 kong-net

docker run -d --name consul \
       --network=kong-net \
       --ip 81.20.7.240 \
       -p 8300:8300 \
       -p 8301:8301 \
       -p 8301:8301/udp \
       -p 8302:8302 \
       -p 8400:8400 \
       -p 8500:8500 \
       -p 8600:8600 \
       -p 8600:8600/udp \
       consul:1.8.0 consul agent -server -recursor 127.0.0.11 -dev -log-level=debug -bootstrap -node -ui -client=0.0.0.0

docker run -d --name kong-database \
       --network=kong-net \
       -p 5432:5432 \
       -e "POSTGRES_USER=kong" \
       -e "POSTGRES_DB=kong" \
       -e "POSTGRES_PASSWORD=k0ngc0nsvl" \
       postgres:9.6

docker run --rm -it \
       --network=kong-net \
       -e "KONG_DATABASE=postgres" \
       -e "KONG_PG_HOST=kong-database" \
       -e "KONG_PG_PASSWORD=k0ngc0nsvl" \
       kong:2.0.4 kong migrations bootstrap

docker volume create kong_data

docker run -d --name kong \
       --network=kong-net \
       --ip 81.20.7.250 \
       -p 8000:8000 \
       -p 8443:8443 \
       -p 8001:8001 \
       -p 8444:8444 \
       -e "KONG_DATABASE=postgres" \
       -e "KONG_PG_HOST=kong-database" \
       -e "KONG_PG_PASSWORD=k0ngc0nsvl" \
       -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
       -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
       -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
       -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
       -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
       -e "KONG_DNS_RESOLVER=81.20.7.240:8600" \
       -e "KONG_LOG_LEVEL=debug" \
       -v kong_data:/etc/kong/ \
       kong:2.0.4 

curl -i http://localhost:8001/

curl -i http://localhost:8000/
```

Link: [http://localhost:8500/ui](http://localhost:8500/ui)

* running kong admin:
```
docker run --rm -it \
       --network=kong-net \
       pantsel/konga -c prepare -a postgres -u postgresql://kong:k0ngc0nsvl@kong-database:5432/konga_db

docker run -d --name konga \
       --network=kong-net \
       --ip 81.20.7.252 \
       -p 1337:1337 \
       -e "NODE_ENV=production" \
       -e "DB_ADAPTER=postgres" \
       -e "DB_HOST=kong-database" \
       -e "DB_USER=kong" \
       -e "DB_PASSWORD=k0ngc0nsvl" \
       -e "DB_DATABASE=konga_db" \
       -e "KONGA_HOOK_TIMEOUT=120000" \
       pantsel/konga
```

Link: [http://localhost:1337/](http://localhost:1337/)

* running the microservice:
```
docker run -d --name census-zipcode-1 \
       --network=kong-net \
       -p 1411:1411 \
       -e "spring.profiles.active=consul" \
       -e "spring.cloud.consul.host=consul" \
       -e "spring.cloud.consul.discovery.serviceName=censuszipcode" \
       -e "server.port=1411" \
       -e "PORT_EXTERNAL=1411" \
       census-zipcode:0.0.1-SNAPSHOT
       
docker run -d --name census-zipcode-2 \
       --network=kong-net \
       -p 1412:1412 \
       -e "spring.profiles.active=consul" \
       -e "spring.cloud.consul.host=consul" \
       -e "spring.cloud.consul.discovery.serviceName=censuszipcode" \
       -e "server.port=1412" \
       -e "PORT_EXTERNAL=1412" \
       census-zipcode:0.0.1-SNAPSHOT
```

Once the microservices are running, you can call:
```
curl http://localhost:1411/zipcode/37188

curl http://localhost:1412/zipcode/37188
```

Looking at the distribution:

```
docker inspect  kong-net   
```

* Adding the kong configuration:
```
curl -X POST http://localhost:8001/upstreams \
     --data "name=censuszipcode" \
     --data 'healthchecks.active.healthy.interval=5' \
     --data 'healthchecks.active.unhealthy.interval=5' \
     --data 'healthchecks.active.unhealthy.http_failures=5' \
     --data 'healthchecks.active.healthy.successes=5'     

curl -X POST http://localhost:8001/upstreams/censuszipcode/targets \
     --data "target=censuszipcode.service.consul:1411" \
     --data "weight=100"
curl -X POST http://localhost:8001/upstreams/censuszipcode/targets \
     --data "target=censuszipcode.service.consul:1412" \
     --data "weight=100"

curl -X POST http://localhost:8001/services/ \
     --data "name=censuszipcoderouter" \
     --data "host=censuszipcode" \
     --data "path=/zipcode"

curl -X POST http://localhost:8001/services/censuszipcoderouter/routes/ \
     --data "paths[]=/(?i)zipcode"
```


Checking the health check:
```
docker logs -f kong --tail 20 | grep healthcheck
```


Now it's possible test the gateway:
```
curl http://localhost:8000/zipcode/
curl http://localhost:8000/zipcode/37188
```

* Monitoring:
```
docker volume create prometheus_data \
       --opt type=tmpfs \
       --opt device=tmpfs \
       --opt o=uid=65534

docker run -d --name prometheus \
       --network=kong-net \
       -p 9090:9090 \
       -v prometheus_data:/data \
       -v $PWD/monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro \
       prom/prometheus \
       --config.file="/etc/prometheus/prometheus.yml" \
       --storage.tsdb.path="/data" \
       --storage.tsdb.retention.time="74h" \
       --web.enable-lifecycle

docker volume create grafana_data

docker run -d --name grafana \
       --network=kong-net \
       -p 3000:3000 \
       -v grafana_data:/var/lib/grafana \
       grafana/grafana     

curl -X POST http://localhost:8001/services/censuszipcoderouter/plugins \
     --data "name=prometheus"       
```

Links: [[Prometheus]](http://localhost:9090/) [[Grafana]](http://localhost:3000/)

Grafana Dashboard:

[https://grafana.com/grafana/dashboards/7424](https://grafana.com/grafana/dashboards/7424)

## References

[https://docs.konghq.com/2.0.x/loadbalancing/](https://docs.konghq.com/2.0.x/loadbalancing/)
