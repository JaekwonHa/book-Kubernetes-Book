## 6장 컨트롤러

컨트롤러란 다양한 목적에 맞게 파드들을 관리할 수 있는 기능



* 레플리케이션 컨트롤러
* 레플리카셋
* 디플로이먼트
* 데몬셋
* 스테이프풀셋
* 잡
* 크론잡





### 6.1 레플리케이션 컨트롤러

지정한 숫자만큼의 파드가 항상 클러스터 안에서 실행되도록 관리

최근에는 레플리케이션 컨트롤러는 deprecated 되고 레플리카셋으로 대체되었습니다.



### 6.2 레플리카셋

레플리케이션 컨트롤러와 같이 일정한 파드 수를 유지해주는 동작을 합니다.

* 레플리케이션 컨트롤러
    * 등호 기반의 셀렉터 지원
    * kubectl rolling-update 사용 가능
* 레플리카셋
    * 집합 기반의 셀렉터 (in, notin, exists) 지원
    * kubectl rolling-update 사용하려면 디플로이먼트를 사용해야함



```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
spec:
  template:												# 1
    metadata:
      name: nginx-replicaset
      labels:
        app: nginx-replicaset
    spec:													# 2
      containers:
      - name: nginx-replicaset
        image: nginx
        ports:
        - containerPort: 80
  replicas: 3											# 3
  selector:												# 4
    matchLabels:
      app: nginx-replicaset
```



1. 레플리카셋이 생성한 파드가 가지게 될 metadata
2. 생성될 파드의 스펙
3. 유지할 파드 수
4. 어떤 레이블의 파드를 선택할지 지정
    1. 1번에서 지정한 레이블과 동일해야하며, 다를시에 에러



#### 6.2.2 레플리카세트와 파드의 연관 관계

쿠버네티스가 레이블로 리소스를 관리하도록 하는 이유는 각 리소스들 간에 연관관계를 느슨하게 하기 위함입니다.

레플리카셋을 삭제하고자 할때 `--cascade=false` 옵션을 사용하면 레플리카셋에서 관리하는 파드는 삭제하지 않고, 레플리카셋만 삭제할 수 있습니다.

(서비스 운영 도중 무중단으로 레플리카셋을 마이그레이션 해야하는 일이 생길때 사용할 수 있을듯)



1. 레플리카셋 생성
2. 파드 삭제
3. 레플리카셋이 파드 생성하는 것 확인
4. 라벨 수정
5. 파드 생성 확인
6. 라벨 확인

```
k get pods -o=json |jq '.items[].metadata.labels.app'
```



## 6.3 디플로이먼트

디플로이먼트는 stateless 애플리케이션을 배포할때 사용하는 가장 기본적인 컨트롤러

파드 개수를 유지하는 것 뿐만 아니라 롤링 업데이트 배포, 배포 중단, 롤백 기능들을 제공합니다.



### 6.3.1 디플로이먼트 사용하기

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3										#1
  selector:
    matchLabels:
      app: nginx-deployment			#2
  template:
    metadata:
      labels:
        app: nginx-deployment
    spec:
      containers:
      - name: nginx-deployment	#3
        image: nginx
        ports:
        - containerPort: 8080

```



1. 유지할 파드 개수
2. 어떤 레이블의 파드를 선택할지 지정
3. 생성될 파드 스펙



디플로이먼트 업데이트 방법

1. kubectl set
2. kubectl edit
3. kubectl apply



```
k get deploy nginx-deployment -o=json |jq '.spec.template.spec.containers[0].image'
```



#### 6.3.2 디플로이먼트 롤백하기

```
k rollout history deploy nginx-deployment
k rollout history deploy nginx-deployment --revision=1
k rollout undo deploy nginx-deployment
k get deploy,rs,rc,pods

k get deploy nginx-deployment -o=json |jq '.spec.template.spec.containers[0].image'
```



```
annotation:
	kubernetes.io/change-cause: version 1.10.1
```



#### 6.3.3 파드 개수 조정하기

```
kubectl scale deploy nginx-deployment --replicas=5
```



#### 6.3.4 디플로이먼트 배포 정지, 배포 재개, 재시작하기

```
kubectl rollout pause deployment/nginx-deployment
kubectl edit deploy nginx-deployment
kubectl rollout history deploy/nginx-deployment
kubectl rollout resume deployment/nginx-deployment
```



쿠버네티스 1.15 부터는 아래 명령어로 전체 파드 재시작이 가능 (하루마다 모델 서버 갱신해야할때 image=latest 로 두고 리스타트로 갱신하는 방법도 가능할듯)

```
kubectl rollout restart deployment/nginx-deployment
```



#### 6.3.5 디플로이먼트 상태

디플로이먼트 상태는 진행, 완료, 실패 상태가 존재합니다

```
k rollout status deployment nginx-deployment
```



* "진행" Progressing
    * 디플로이먼트가 새로운 레플리카셋을 만들 때
    * 디플로이먼트가 새로운 레플리카셋의 파드 개수를 늘릴때
    * 디플로이먼트가 예전 레플리카셋의 파드 개수를 줄일때
    * 새로운 파드가 준비 상태가 되거나 이용 가능한 상태가 되었을때
* "완료" Complete
    * 디플로이먼트가 관리하는 모든 레플리카셋이 업데이트 완료되었을 때
    * 모든 레플리카셋이 사용 가능해졌을때
    * 예전 레플리카셋이 모두 종료되었을때
* "실패" Fail
    * 쿼터 부족
    * readinessProbe 진단 실패
    * 컨테이너 이미지 가져오기 에러
    * 권한 부족
    * 제한 범위 초과
    * 앱 실행 조건을 잘못 지정



템플릿에 .spec.progressDeadlineSeconds 항목을 조절하면 지정된 시간 동안 배포가 완료되지 못하면 실패 상태가 됩니다.



```
k describe deploy nginx-deployment
```



### 6.4 데몬셋

클러스터 전체 노드 마다 특정 파드를 실행해야 할때 사용하는 컨트롤러입니다.

노드마다 로그 수집기를 배포하거나, 노드 자체를 모니터링하는 모니터링용 데몬을 실행해야할 때 사용합니다.



```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system						#1
  labels:
    k8s-app: fluentd-logging				#2
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
    updateStrategy:
      type: RollingUpdate						#3
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      containers:
      - name: fluentd-elasticsearch
        image: fluentd/fluentd-kubernetes-deamonset:elasticsearch	#4
        env:
          - name: testenv
            value: value
```



1. 로그 수집기는 쿠버네티스 관리용 파드나 설정에 해당하므로 kube-system 네임스페이스를 사용하도록 함
2. 오브젝트를 식별하는 레이블 설정
3. 데몬셋 파드를 업데이트 하는 방법 설정
    1. OnDelete, RollingUpdate 설정 가능
        1. RollingUpdate 는 바로 적용되고 일정 비율만큼 순차적으로 업데이트
        2. OnDelete 는 바로 적용되지 않고, 파드를 지워주면 새로 생성되면서 업데이트
4. 컨테이너 이미지 설정



```
k get daemonset -n kube-system
```



#### 6.4.2 데몬셋의 파드 업데이트 방법 변경하기



1. testenv 의 value 를 value01 로 변경
2. 변경 사항 확인
3. 업데이트 방법을 OnDelete 로 변경
4. 바로 변경 사항 적용되는 것 확인 (기존에는 RollingUpdate 였으므로)
5. testenv 의 value 를 value03 로 변경
6. 변경 사항 확인
7. 파드 삭제
8. 변경 사항 확인



### 6.5 스테이트풀셋

앞서 살펴봤던 레플리케이션 컨트롤러, 레플리카셋, 디플로이먼트는 모두 stateless 파드를 관리하는 용도입니다.

하지만 아파치 주키퍼, 레디스, mysql 등의 stateful 한 애플리케이션을 배포해야하는 경우도 있습니다.

특정 볼륨(스토리지)에는 특정 파드만 접근해야한다거나, 레플리카셋의 파드처럼 동적으로 IP 가 변하는게 아니라 예측 가능한 주소를 가져야 한다거나, 인스턴스가 고유한 식별자를 가져야 한다던가, 인스턴스의 위치, 순서가 고정되어 있어야 한다거나(스케일 업, 스케일 다운의 순서에 영향을 주는 등..) 하는 요구사항이 있을 수 있습니다.

* 스토리지
* 네트워크
* 식별자
* 순서성



레플리카셋과의 차이점은 레플리카셋은 최소한 X개 보장(at least X guarantee) 를 제공하지만 스테이트풀셋은 최대한 하나(at most one)을 보장합니다.

레플리카셋 2 의 경우 항상 2개 이상의 인스턴스를 실행 상태로 유지하려고 합니다. 상황에 따라 2개 이상의 파드가 실행 될 수 있습니다.

스테이트풀셋의 경우 중복 파드가 없도록 가능한 모든 체크를 수행하고, 이전 인스턴스가 종료되지 않았다면 파드를 다시 시작하지 않습니다.



```yaml
---
apiVersion: v1
kind: Service									
metadata:
  name: nginx-statefulset-service							#1
  labels:
    app: nginx-statefulset-service
spec:
  ports:
    - port: 80
      name: web
  clusterIP: None
  selector:
    app: nginx-statefulset-service
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web																		#2
spec:
  serviceName: "nginx-statefulset-service"
  replicas: 3
  selector:
    matchLabels:
      app: nginx-statefulset
  template:
    metadata:
      labels:
        app: nginx-statefulset
    spec:
      terminationGracePeriodSeconds: 10				#3
      containers:
      - name: nginx-statefulset
        image: nginx
        ports:
          - containerPort: 80
            name: web

```



1. 스테이트풀셋에서 사용할 서비스
    1. 파드이름.서비스이름 으로 도메인이 생성됨
2. 스테이트풀셋의 이름을 설정
3. graceful shutdown 의 대기 시간을 설정



파드이름에 UUID 가 붙는 것이 아닌 넘버링이 붙음

작은 순서부터 실행되고, 큰 순서부터 삭제됨



#### 6.5.2 파드를 순서없이 실행하거나 종료하기

스테이트풀셋의 기본 동작은 순서대로 파드를 관리하는 것이지만 .spec.podManagement.Policy 필드를 수정하여 순서를 없앨 수 있습니다.

* OrderedReady
* Parallel



#### 6.5.3 스테이트풀셋의 파드 업데이트하기

파드 업데이트시에 기본적으로 마지막 파드부터 재시작을 시작합니다.

.spec.updateStrategy.rollingUpdate.partition 필드값을 사용하면 변경 사항이 있을때 파티션 넘버보다 큰 값을 가진 파드들만 업데이트를 합니다.

파티션 값을 0 -> 1 로 변경 후 testenv 값을 수정하면 web-0 파드는 업데이트가 발생하지 않음

또한 `statefulset.kubernetes.io/pod-name=web-0` 레이블 값을 사용하면 전체 스테이트풀셋 파드 중 일부에만 서비스를 연결할 수도 있습니다.



.spec.updateStrategy.type 을 OnDelete 로 설정하면 데몬셋과 같이, 변경사항이 바로 적용되는게 아니라 삭제시에 다시 생성하면서 업데이트 됩니다.



### 6.6 잡

잡은 파드 실행 후 종료되어야 하는 성격의 작업에 사용하는 컨트롤러 입니다.



```yaml
apiVersion: batch/v1					#1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
        - name: pi
          image: perl
          command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]	#2
      restartPolicy: Never																							#3
  backoffLimit: 4																												#4
```



1. 잡이 포함된 apiVersion
2. 수행할 커맨드
3. 컨테이너의 재시작유무
    1. Never, OnFailure (실패시 재시작)
4. 잡 실행이 실패했을때 재시도 횟수
    1. 잡 실행이 실패했을때 재시도 할때마다 대기 시간이 늘어남. 처음은 10초, 그다음은 20초...이런식으로



#### 6.6.2 잡 병렬성 관리

.spec.parallelism 필드로 잡 하나가 몇개의 파드를 동시에 실행할지를 잡 병렬성이라고 합니다.

병렬성 옵션을 1 보다 큰 값으로 주더라도 여러가지 이유로 지정된 값보다 적게 파드가 실행될 수 있습니다.

* 정상 완료되어야 하는 잡의 개수가 총 10개이고, 현재 9개까지 완료된 상태라면 병렬성을 3으로 설정하더라도 실제로는 1개 파드만 실행됩니다.
* 워크 큐용 잡에서는 파드 하나가 정상적으로 완료될까지 새로운 파드를 실행하지 않습니다.
* 잡 컨트롤러가 정상적으로 실행 중이 아니거나, 자원부족, 권한부족 등의 문제로 파드를 실행하지 못할 수 있습니다.
* 잡의 파드들이 너무 많이 실패하면 제한될 수 있습니다.
* 파드가 그레이스풀하게 종료되었을때



#### 6.6.3 잡의 종류

* 단일잡
    * 파드 하나만 실행
    * 파드 생명 주기가  Succeeded 되면 잡 실행 완료
    * .spec.completions, .spec.parallelism 필드를 설정하지 않아야 함
* 완료된 잡 개수가 있는 병렬잡
    * .spec.completions 값을 양수로 설정
    * .spec.parallelism 값을 설정하지 않거나 1로 설정
* 워크큐가 있는 병렬잡
    * .spec.completions 필드를 설정하지 않고 .spec.parallelism 은 양수로 설정
    * .spec.completions 필드를 설정하지 않으면 .spec.parallelism 필드와 동일한 값으로 설정됨
    * 파드 각각은 정상적으로 실행 종료됐는지를 독립적으로 결정. 대기열의 모든 작업이 동시에 실행될 수도 있음
    * 파드 하나라도 정상적으로 실행 종료되면 새로운 파드가 실행되지 않습니다.
    * 최소한 파드 1개가 정상적으로 종료된 후 모든 파드가 실행 종료되면, 잡이 정상적으로 종료됩니다.
    * 일단 파드 1개가 정상적으로 실행 종료되면 다른 파드는 더 이상 동작하지 않거나 어떤 작업 처리 결과를 내지 않습니다. 다른 파드는 모두 종료 과정을 실행합니다.



#### 6.6.4 비정상적으로 실행 종료된 파드 관리하기

파드 안에 비정상적으로 실행 종료된 컨테이너가 있을 수 있습니다.

컨테이너의 재시작 정책은 .spec.template.spec.restartPolicy 필드로 지정할 수 있습니다.

OnFailure 로 설정시 파드는 원래 실행 중이던 노드에서 컨테이너를 재시작합니다.

Never 로 설정시 파드는 컨테이너를 재시작하지 않고, 잡에서 새로운 파드를 실행합니다.

이때 파드가 이전에 처리 중이던 상태를 인식해서 해당 상태부터 재시작하거나, 처음부터 다시 시작하게 할 수도 있는데, 동일한 프로그램이 2번 이상 실행될 수도 있기 때문에 애플리케이션은 이런 상황을 고려해서 설계되어야 합니다.



#### 6.6.5 잡 종료와 정리

잡은 정상적으로 실행 종료되면 잡과 파드는 남아있게 되고 이를 이용해 로그를 분석할 수도 있습니다.

특정 시작을 지정해 잡 실행을 종료하려면 .spec.activeDeadlineSeconds 필드에 시간을 설정합니다.

잡은 삭제하면 파드들도 모두 삭제됩니다.



#### 6.6.6 잡 패턴

잡을 병렬로 실행했을때 각 파드는 독립적으로 실행됩니다. 일반적인 사용 패턴은 다음과 같습니다.

* 작업마다 잡을 생성하는 것보다는, 작업이 많아질 수록 잡 하나가 여러개의 작업을 처리하는게 좋습니다.
* 작업 개수만큼의 파드를 생성하는 것보다 파드 하나가 여러개의 작업을 처리하는게 좋습니다.
* 워크 큐를 사용한다면 카프카, RabbitMQ 같은 큐 서비스로 워크 큐를 구현하도록 기존 프로그램이나 컨테이너를 수정해야 합니다.



### 6.7 크론잡

크론잡은 잡을 시간 기준으로 관리하도록 생성합니다.

지정한 시간에 한번만 잡을 실행하거나 지정한 시간동안 주기적으로 잡을 반복 실행할 수 있습니다.



```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"								#1
  jobTemplate:
    spec:
      template:
       spec:
         containers:
           - name: hello
             image: busybox
             args:
               - /bin/sh
               - -c
               - date; echo Hello from the kubernetes cluster
         restartPolicy: OnFailure
```



1. 일반적인 크론 명령과 같이 스케쥴을 지정



#### 6.7.2 크론잡 설정

* .spec.startingDeadlineSeconds
    * 지정된 시간에 크론잡이 실행되지 못했을때, 설정한 시간까지도 실행되지 못하면 크론잡이 더이상 실행되지 않습니다.
* .spec.concurrencyPolicy
    * 크론잡이 여러개 잡을 동시에 실행할 수 있습니다.
    * 만약 이전에 실행했던 잡이 아직 실행 중이거나, 정상적으로 종료되지 않고 실행 중 상태인데 다시 실행 시간이 돌아왔다면 어떻게 동작할지를 설정할 수 있습니다.
    * Allow, Forbid, Replace



