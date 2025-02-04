# 2025.02.04 Update

## 📗 목차

1. 클러스터는 정상 작동 된다.
2. 삽질했던 이유
3. Spring Boot에서 정상 연동 된다. 근데 안된다.(응?)
4. 로컬용 Redis Cluster는 docker-compose를 따로 만들어야 한다.(뭐 근데 원래 그런거였잖아...)

<br/>

## 😁 1. 클러스터는 정상 작동 된다.

- 정상 작동하는 파일인 `redis-cluster-docker-compose2.yml`를 참조하자.

## 👊🏻 2. 삽질했던 이유

- 클러스터는 애초에 docker-compose를 잘못 작성했던 것... 
- 오히려 긴 삽질은 따로 있었음.
- spring app에서 붙이려는데 로컬에서는 클러스터에 붙지만, 컨테이너에서는 안붙음.
- 그래서 application.yml 자체에는 문제가 없다고 봤음. 그러나.
- 로컬에서 잘 작동했던 이유는 원래 brew로 실행시켜놓은 redis에 붙어있던 것임.
- 즉 application.yml 설정은 동작하지 않았고, 로컬에 있던 기본 6379에 붙으니 잘 되었던 것.
- 문법을 아래와 같이 수정. 원래 `data:`가 없었음.

```yml
spring:
  data:
    redis:
      cluster:
        nodes: ...
```

## 😅 3. Spring Boot에서 정상 연동 된다. 근데 안된다.(응?)

- 결론: 로컬에서 코드 실행할 때는 안된다는 의미임.
- `redis-cluster-docker-compose2.yml`를 보면
- Redis Cluster가 각 node의 announce된 주소를 `redis-node-7100`의 형식으로 가지고 있기 때문에
- 로컬에서 아무리 127.0.0.1로 시도해도 `redis-node-7100` 형식으로 연결함. 그러니 연결이 될 수 없다. (아래 명령어를 말하는 거임...)

```
redis-cli --cluster create
redis-node-7100:7100 redis-node-7101:7101 redis-node-7102:7102
redis-node-7103:7103 redis-node-7104:7104 redis-node-7105:7105
--cluster-replicas 1
```

- Q. 각 서비스마다 `--cluster-announce-ip redis-node-7100`를 추가해서 그런거 아님?
- A. 응 맞음. 근데 상관 없음. 왜냐면 그걸 안해도 docker network의 내부IP값이 default임..
- &nbsp; `redis-node-7100`로 안하다 뿐이지 172.x.x.x 같은 IP로 접근할테니 어차피 안됨.
- Q. 그럼 어쩌라는 거임?

## 🤨 4. 로컬용 Redis Cluster는 docker-compose를 따로 만들어야 한다.(뭐 근데 원래 그런거였잖아...)

- docker compose 파일 안에 각 노드마다 `--cluster-announce-ip`를 127.0.0.1로 하면 됨.
- cluster-creator 서비스 안에 명령어 조합도 수정을 이런식으로 해야할 듯.

---

### docoker-compose 파일 설정

#### 컨테이너 외부에서 접근
```
--cluster-announce-ip 127.0.0.1
--cluster-announce-port 7001
--cluster-announce-bus-port 17001
```

### 클러스터 만들기
```
docker exec -it {7001 컨테이너 이름} redis-cli --cluster create \     
  redis-7001:7001 redis-7002:7002 redis-7003:7003 \
  redis-7004:7004 redis-7005:7005 redis-7006:7006 \
  --cluster-replicas 1
```

이후 yes 를 누르면... 알아서 cluster join 중 이라고 뜨면서 완료된다... 하지만
cluster가 제대로 join 되지 않고 무한 로딩이 진행되어 수동 join 을 아래와 같이 실시한다.


아래 명령으로 사용 중인 docker의 네트워크를 다 찾고
`docker network ls`


webtoon_default 라는 네트워크가 클러스터 대상이 되는 node들이 사용 중인 것 같다.
아래 명령어로 네트워크 사용중인 컨테이너 다 찾는다.
`docker network inspect webtoon_default`

아래 내용은 위 명령어의 결과의 일부다.
```
"2191b8563d982a9befba2e14d9628b442e5f816dec5983fcbd4ba09bc809c538": {
    "Name": "webtoon-redis-7006-1",
    "EndpointID": "5f7a7260255b6cb1fcf4c0a2209857fee0ba096b66d3669b22485a1d3ddf7a4e",
    "MacAddress": "02:42:ac:17:00:05",
    "IPv4Address": "172.23.0.5/16",
    "IPv6Address": ""
},
"2e965397b2a4626af1e15e144b4c0e0a1296b86ddb9912f2d9138bcd57275862": {
    "Name": "webtoon-redis-7004-1",
    "EndpointID": "1df2eceaac2a2a8fe885ea1c39b9e37a4ef830fbd2e2fe82a55a3ea0a45bd8e8",
    "MacAddress": "02:42:ac:17:00:07",
    "IPv4Address": "172.23.0.7/16",
    "IPv6Address": ""
},
"4a2c86f5f5ecdb0f99d528008d7afe8fc792dd6c9c35d90c0c43a87a3146d21e": {
    "Name": "webtoon-redis-7003-1",
    "EndpointID": "b4acfc102963042c6d309496c8da188405bee252efdae0f2ef2a6492f9095d2d",
    "MacAddress": "02:42:ac:17:00:03",
    "IPv4Address": "172.23.0.3/16",
    "IPv6Address": ""
},
"b0863500128453623641d9b2af8988223900729e9e73ff3ec628ce32a9159dca": {
    "Name": "webtoon-redis-7002-1",
    "EndpointID": "535d22709ede2286a8e405eba7710015b3bfa4c5e30e9eaa2395eb025d3b8faa",
    "MacAddress": "02:42:ac:17:00:02",
    "IPv4Address": "172.23.0.2/16",
    "IPv6Address": ""
},
"d91d78b397e9b2f0474fac4d7f6f804b6487e0a3e186055078c3c99f89c25331": {
    "Name": "webtoon-redis-7005-1",
    "EndpointID": "da520ec54be50e01c961f04b857c29884a4252050688d728ccd29c3269169c8e",
    "MacAddress": "02:42:ac:17:00:04",
    "IPv4Address": "172.23.0.4/16",
    "IPv6Address": ""
},
"eda05174e4543765b9f73138489ef5764351a656072e09942cdae9dc4dfd3a4d": {
    "Name": "webtoon-redis-7001-1",
    "EndpointID": "0c504456356764907f97a446744bdd9b7eabd627f23b33ba0e3c6eb75076a8f3",
    "MacAddress": "02:42:ac:17:00:06",
    "IPv4Address": "172.23.0.6/16",
    "IPv6Address": ""
},
```

자 이제 수동으로 node 연결하기 시작....
step 순서는 두칸씩 띄어 놨다.


7001에게 다른 노드들 인지시키기.
```
docker exec -it webtoon-redis-7001-1 redis-cli -p 7001 cluster meet 172.23.0.2 7002
docker exec -it webtoon-redis-7001-1 redis-cli -p 7001 cluster meet 172.23.0.3 7003
docker exec -it webtoon-redis-7001-1 redis-cli -p 7001 cluster meet 172.23.0.7 7004
docker exec -it webtoon-redis-7001-1 redis-cli -p 7001 cluster meet 172.23.0.4 7005
docker exec -it webtoon-redis-7001-1 redis-cli -p 7001 cluster meet 172.23.0.5 7006
OK
OK
OK
OK
OK
```


아래 처럼 7001은 cluster nodes 로 다른 노드들을 인지하고 있음..
```
% docker exec -it webtoon-redis-7001-1 redis-cli -p 7001 cluster nodes              
4ab749cd4778de98ff184b68e2b39c3613272b83 172.23.0.7:7004@17004 master - 0 1738398638627 0 connected
dc4378bc9f42d63839dff4891d84437a00a8798e 172.23.0.4:7005@17005 master - 0 1738398638627 0 connected
9e956323b943885ff64f484a7b249194b0949538 172.23.0.5:7006@17006 master - 0 1738398637609 0 connected
772e7c9ebeb4e619b5914d75d4a76d0b453cdd3f 127.0.0.1:7001@17001 myself,master - 0 0 1 connected
ee67696f107713b2e89251266533022f73553ef5 172.23.0.3:7003@17003 master - 0 1738398639139 0 connected
2ce8c1918412d95116c02070bb805b0483452269 172.23.0.2:7002@17002 master - 0 1738398639139 0 connected
```


그리고 7002, 7003은 master로서 슬롯을 할당받은 상태임. 슬롯 할당은 아래와 같이...
```
docker exec -it webtoon-redis-7001-1 redis-cli -p 7001 cluster addslots {0..5460}
docker exec -it webtoon-redis-7002-1 redis-cli -p 7002 cluster addslots {5461..10922}
docker exec -it webtoon-redis-7003-1 redis-cli -p 7003 cluster addslots {10923..16383}
OK
OK
OK
```


다시 cluster nodes 해보면 node들 정보와 함께 슬롯 할당되어 있는 것도 볼 수 있음..
```
% docker exec -it webtoon-redis-7001-1 redis-cli -p 7001 cluster nodes              
4ab749cd4778de98ff184b68e2b39c3613272b83 172.23.0.7:7004@17004 master - 0 1738398638627 0 connected
dc4378bc9f42d63839dff4891d84437a00a8798e 172.23.0.4:7005@17005 master - 0 1738398638627 0 connected
9e956323b943885ff64f484a7b249194b0949538 172.23.0.5:7006@17006 master - 0 1738398637609 0 connected
772e7c9ebeb4e619b5914d75d4a76d0b453cdd3f 127.0.0.1:7001@17001 myself,master - 0 0 1 connected 0-5460
ee67696f107713b2e89251266533022f73553ef5 172.23.0.3:7003@17003 master - 0 1738398639139 0 connected 10923-16383
2ce8c1918412d95116c02070bb805b0483452269 172.23.0.2:7002@17002 master - 0 1738398639139 0 connected 5461-10922
```

아직 7004, 7005, 7006은 master 상태임... 각각 자기가 master로 모실 node들을 meet하고, slave가 되도록 설정함..
```
docker exec -it webtoon-redis-7004-1 redis-cli -p 7004 cluster meet 172.23.0.6 7001
OK
docker exec -it webtoon-redis-7004-1 redis-cli -p 7004 cluster replicate 772e7c9ebeb4e619b5914d75d4a76d0b453cdd3f
OK
docker exec -it webtoon-redis-7005-1 redis-cli -p 7005 cluster meet 172.23.0.2 7002                            
OK
docker exec -it webtoon-redis-7005-1 redis-cli -p 7005 cluster replicate 2ce8c1918412d95116c02070bb805b0483452269 
OK
docker exec -it webtoon-redis-7006-1 redis-cli -p 7006 cluster meet 172.23.0.3 7003                                         
OK
docker exec -it webtoon-redis-7006-1 redis-cli -p 7006 cluster replicate  ee67696f107713b2e89251266533022f73553ef5
```

