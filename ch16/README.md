# 16장 쿠버네티스 DNS



## 사전 지식

* 8.8.8.8 IP 이란?

    * > https://ko.wikipedia.org/wiki/%EA%B5%AC%EA%B8%80_%ED%8D%BC%EB%B8%94%EB%A6%AD_DNS

    * 구글에서 제공해주는 무료 DNS 서버

    * 8.8.8.8, 8.8.4.4 IP 를 사용

* A, AAAA 레코드란?

    * DNS 레코드 타입 종류입니다.
    * DNS 레코드 타입 종류
        * A
            * 도메인에 연결된 ipv4 주소
        * AAAA
            * 도메인에 연결된 ipv6 주소
        * CNAME
            * 도메인에 부여한 2차 도메인

* fully qualified hostname

    * 절대 도메인 네임, 전체 도메인 네임, FQDN
    * ex. www.tistory.com, onlywis.tistory.com
    * 위 주소 중 www / onlywis 부분은 "호스트", tistory.com 은 "도메인"
    * 보통은 마지막에 dot (.) 을 붙여서 나타내지만 생략하기도 한다



## 16.1 쿠버네티스 DNS

쿠버네티스는 클러스터 안에서만 사용하는 DNS를 설정할 수 있고, IP 대신 도메인을 사용하여 통신할 수 있습니다.

도메인을 사용하게되면 특정 파드나 디플로이먼트를 재생성할 때 자동으로 변경된 파드의 IP를 도메인에 등록해주게 되어 자연스럽게 새로 실행한 파드로 연결할 수 있습니다.

IP로 통신하게 되어있다면 IP가 변경됨에 따라서 수정한 후 적용해야 하는 번거로움이 있을겁니다.



전문적인 서비스 디스커버리 시스템을 통해서 파드의 IP를 조회할 수도 있겠지만, 파드 삭제/생성과 같이 간단한 상황에서는 DNS를 활용할 수 있습니다.

처음에는 `kube-dns` 라는 DNS를 사용했지만, 버전 1.13 부터는 `CoreDNS` 를 기본으로 사용하고 있습니다.



## 16.2 클러스터 안에서 도메인 사용하기

쿠버네티스의 도메인에는 특정한 규칙이 있습니다.

* 서비스: '서비스이름.네임스페이스.svc.cluster.local'
    * 네임스페이스: aname / 서비스: bservice
    * -> bservice.aname.svc.cluster.local
* 특정 파드: '파드IP주소.네임스페이스이름.pod.cluster.local'
    * 네임스페이스: default / 파드: cpod (10.10.10.10)
    * -> 10-10-10-10.default.pod.cluster.local

파드의 IP를 그대로 도메인에 사용할 수 있지만, IP를 사용하는 것보다는 도메인을 사용하는 것이 좋습니다. 다음과 같이 파드 템플릿에 호스트네임과 서브 도메인을 설정할 수 있습니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubernetes-simple-app
  labels:
    app: kubernetes-simple-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubernetes-simple-app
  template:
    metadata:
      labels:
        app: kubernetes-simple-app
    spec:
      hostname: appname
      subdomain: default-subdomain
      containers:
      - name: kubernetes-simple-app
        image: arisu1000/simple-container-app:latest
        ports:
        - containerPort: 8080
```



* 파드 도메인: '호스트네임이름.서브도메인이름.네임스페이스이름.svc.cluster.local'
    * -> appname.default-subdomain.default.svc.cluster.local

도메인의 마지막이 pod.cluster.local 이 아니라 svc.cluster.local 입니다.



nslookup 명령으로 파드 dns 질의를 해봅니다.

```bash
kubectl get pods -o wide
kubectl exec kubernetes-simple-app-dc7dd6598-rxxk7 -- nslookup appname.default-subdomain.default.svc.cluster.local

# dns 세팅 확인
kubectl exec kubernetes-simple-app-dc7dd6598-rxxk7 -- cat /etc/resolv.conf

# 네임서버의 IP가 kube-dns 의 cluster-ip 임을 확인
```



## 16.3 DNS 질의 구조

파드마다 안에서 도메인 이름 질의 정책을 설정할 수 있습니다.

* Default: 파드는 파드가 실행되고 있는 "노드"로부터 DNS 설정을 상속받습니다.
    * **Default 는 기본 DNS 정책이 아닙니다. dnsPolicy 가 설정되어 있지 않다면 "ClusterFirst" 가 기본값으로 사용됩니다.**
* ClusterFirst: Cluster DNS 를 우선하고, `www.kubernetes.io` 와 같이 클러스터 도메인 suffix 구성과 일치하지 않는 DNS 쿼리는 노드에서 상속된 업스트림 네임서버로 전달됩니다.
* ClusterFirstWithHostNet: hostNetwork 에서 실행중인 파드의 경우 DNS 정책을 ClusterFirstWithHostNet 으로 설정해야 합니다.
* None: 클러스터 안 DNS 설정을 무시합니다. dnsConifig 필드를 통해 별도의 DNS 설정이 필요합니다.



### 16.3.1 kube-dns 질의 구조

kube-dns 파드 내부에서는 3가지 컨테이너가 존재합니다.

* kubedns
    * 쿠버네티스 DNS 변경을 감지하는 컨테이너
* dnsmasq
    * DNS 캐시 컨테이너
* sidecar
    * dnsmasq 를 헬스체크하는 컨테이너

![스크린샷 2021-12-10 오전 10.18.07](/Users/user/Library/Application Support/typora-user-images/스크린샷 2021-12-10 오전 10.18.07.png)

### 16.3.2 CoreDNS 질의 구조

coredns 파드는 Corefile 이라는 CoreDNS 자체의 설정 파일에 맞게 DNS를 설정합니다.

```bash
kubectl describe configmap coredns -n kube-system

Name:         coredns
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Data
====
Corefile:
----
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    hosts {
       192.168.59.1 host.minikube.internal
       fallthrough
    }
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}


BinaryData
====

Events:  <none>

```

:53 하위 항목은 도메인의 루트 영역( . )의 53 포트 (DNS 기본 포트) 에 대한 내용이라는 뜻입니다.

중괄호 내부는 사용할 플러그인들을 명시하는데, 쿼리는 각 플러그인들의 순서대로 처리됩니다.

* errors
    * 표준 출력으로 에러 로그를 남깁니다.
* health
    * http://localhost:8080/health 로 CoreDNS의 헬스체크를 할 수 있습니다.
    * lameduck 은 프로세스를 비정상 상태로 만든다는 뜻이고, 종료되기까지 5초를 기다립니다.
* ready
    * 8181 포트의 http 엔드포인트로 요청을 보내면 모든 플러그인이 준비되었다는 신호 200 OK 를 반환합니다.
* kubernetes
    * pods insecure
        * kube-dns 와의 하위 호환성을 제공합니다.
        * pods verified 로 변경하면 같은 네임스페이스에 속한 파드끼리만 A 레코드에 관한 DNS 쿼리에 응답할 수 있습니다.
    * fallthrough
        * 도메인을 찾는데 실패했을때의 동작을 정의합니다. 보통 도메인 찾기를 실패하면 NXDOMAIN 에러가 발생합니다.
        * 이때 fallthrough 설정이 in-addr.arpa와 ip6.arpa 되어 있다면 IPv4, IPv6 주소 체계에 맞는 값을 설정하며 리버스 쿼리 (IP 주소로 도메인을 찾는 쿼리)를 할 수 있음을  뜻합니다.
    * ttl
        * 응답에 대한 사용자 정의 TTL. 기본 5s, 최소 0s, 최대 3600s
* prometheus
    * :9153/metrics 주소로 프로메테우스 형식의 메트릭 정보를 제공
* forward
    * 쿠버네티스 클러스터 도메인으로 설정되지 않은 DNS 쿼리를 /etc/resolv.conf 에 설정된 외부 DNS 서버로 보내서 처리합니다.
* cache
    * DNS 쿼리의 캐시 유지 시간
* loop
    * 순환 참조 구조가 있는지 찾아서 CoreDNS 프로세스를 중지합니다.
* reload
    * Corefile 이 변경되었는지 감지해서 자동으로 설정 내용을 반영합니다.
    * 보통 ConfigMap 을 수정하더라도 파드가 재시작되기 전에는 반영되지 않습니다.
    * reload 설정을 해두면 파드를 재시작하지 않더라도 2분 정도 후 반영됩니다.
* loadbalance
    * 도메인에 설정된 레코드가 여러개 있을때 라운드로빈으로 요청이 전달됩니다. 도메인 A에 10.10.10.10 / 10.10.10.20 레코드 2개가 등록되어있을때 번갈아가면서 요청을 보냅니다.



## 16.4  파드 안에 DNS 직접 설정하기

파드가 사용할 DNS 를 직접 설정할 수도 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-test
spec:
  containers:
  - name: dns-test
    image: arisu1000/simple-container-app:latest
  dnsPolicy: ClusterFirst
  dnsConfig:
    nameservers:
      - 8.8.8.8
    searches:
      - default.svc.cluster.local
      - example.com
    options:
      - name: name01
        value: value01
      - name: name02


```

* nameservers
    * 파드에서 사용할 DNS IP
    * 1~3개 설정 가능
* searches
    * DNS 를 검색할 때 사용하는 기본 도메인
    * a.b 라고만 검색해도 a.b.svc.cluster.local 까지도 검색해준다
    * 최대 6개
* options
    * 원하는 DNS 관련 옵션
    * name 은 필수, value 는 옵션

dnsConfig 로 설정한 값은 /etc/resolv.conf 에 추가됩니다.

```bash
kubectl exec kubernetes-simple-app-dc7dd6598-rxxk7 -- cat /etc/resolv.conf
```
