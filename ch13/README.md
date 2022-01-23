# 13장 인증과 권한 관리



## 13.1 인증

쿠버네티스는 kube-apiserver 를 실행하여 쿠버네티스의 접근을 API 호출의 형태로 허용하고 있습니다.

외부에서 API 서버에 접근할 수 있는 기본 포트는 6443이고 TLS 인증이 적용되어 있습니다.

쿠버네티스의 계정은

* 사용자 계정: 구글 계정, 오픈스택의 키스톤, LDAP 등 별도의 외부 인증 시스템에 있는 사용자 정보를 연결
    * 별도의 외부 인증 시스템이 없다면 단순하게 파일에 사용자 이름과 비밀번호를 저장해 사용할 수 있음
* 서비스 계정: 쿠버네티스가 직접 관리하는 사용자 계정. 해당 서비스 계정에는 시크릿이 할당되어 비밀번호 역할을 함



> https://kubernetes.io/ko/docs/reference/access-authn-authz/authorization/



### 13.1.1 kubectl의 config 파일에 있는 TLS 인증 정보 구조 확인하기

kube-apiserver 와 통신할때 기본 인증 방법으로 TLS 를 사용하기 때문에 클라이언트 인증서가 필요합니다.

우리가 kubectl 명령을 수행하면 쿠버네티스 클러스터에 연결되었던 이유는 .kube/config 파일에 인증 정보가 포함되어 있었기 때문입니다.

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /Users/user/.minikube/ca.crt
    extensions:
    - extension:
        last-update: Thu, 18 Nov 2021 15:57:37 KST
        provider: minikube.sigs.k8s.io
        version: v1.21.0
      name: cluster_info
    server: https://192.168.99.115:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    extensions:
    - extension:
        last-update: Thu, 18 Nov 2021 15:57:37 KST
        provider: minikube.sigs.k8s.io
        version: v1.21.0
      name: context_info
    namespace: default
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    client-certificate: /Users/user/.minikube/profiles/minikube/client.crt
    client-key: /Users/user/.minikube/profiles/minikube/client.key
```

* cluesters
    * cluster.certificate-authority-data: 클러스터 인증에 필요한 해시값
    * cluster.insecure-skip-tls-verify: 인증서가 사설 인증서, private 인증서이기 때문에 true 로 설정하여 검증 과정을 생략
    * cluster.server: 외부에서 kube-apiserver 에 접속할 주소
    * name: 클러스터 이름
* contexts
    * context.cluster: 접근할 클러스터 이름
    * context.user: 클러스터에 접근할 사용자 그룹
    * context.namespace: 접근할 네임스페이스
    * name: 컨텍스트 이름
* users
    * name: 사용자 그룹 이름
    * user.client-certificate-data: 클라이언트 인증에 필요한 해시값
    * user.client-key-data: 클라이언트의 키 해시값



### 13.1.2 서비스 계정 토큰을 이용해 인증하기

```bash
JWT_TOKEN=$(kubectl get secret default-token-5jltq -ojson | jq -r .data.token | base64 -d)
kubectl api-versions --token $JWT_TOKEN
kubectl get po --token $JWT_TOKEN
```



## 13.2 권한 관리

쿠버네티스 클러스터 인증 후에는 접근하려는 API를 사용할 권한이 있는지 확인해야 합니다.

권한 관리 방법

* ABAC. Attribute-based access control
    * 속성 기반 권한 관리
    * 속성: 사용바, 그룹, 요청 경로, 요청 동사 등
    * 권한 설정 내용을 파일로 관리하기 때문에 변경시에는 마스터에 접속해서 파일을 변경한 후 kube-apiserver 재시작
* RBAC. Role-based access control
    * 역할 기반 권한 관리
    * 사용자에게 역할을 Binding 해서 권한을 부여
    * 최근 많이 사용하는 방법 (ex. AWS)



### 13.2.1 롤

특정 API 나 자원 사용 권한들을 명시해둔 규칙의 집합

* 일반 롤
    * 해당 롤이 속한 네임스페이스에만 적용
    * 네임스페이스에 한정되지 않은 자원과 API들의 사용권한을 설정할 수 있음
        * 노드 사용 권한
        * 헬스 체크용 URL "/healthz"

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: read-role
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
```

* verbs
    * Create, Get, List, Update, Patch, Delete, deletecollection
* resourceNames
    * 리소스의 이름을 넣어서, 리소스 하위 특정 자원에만 접근할 수 있도록 설정할 수 있음



* 클러스터 롤
    * 특정 네임스페이스 사용 권한이 아닌 클러스터 전체 사용 권한

```yaml
Kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-clusterrole
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
```

* aggregationRole
    * 클러스터롤에 설정된 레이블을 통해 다수의 Role 을 조합해서 사용할 수 있음

```yaml
Kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: admin-aggregation
aggregationRole:
  clusterRoleSelectors:
    - matchLabels:
        kubernetes.io/bootstrapping: rbac-defaults
rules: []
```

* 클러스터롤의 특징
    * 지금까지 규칙은 대부분 자원을 대상으로 설정했는데, 클러스터롤은 자원이 아니라 URL 형식으로 규칙을 설정

```yaml
rules:
- nonResourceURLs: ["/healthcheck", "/metrics/*"]
  verbs: ["get", "post"]
```



### 13.2.3 롤바인딩

롤바인딩은 사용자와 롤을 묶는 역할입니다.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myuser
  namespace: default
```

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-rolebinding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: myuser
    apiGroup: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: read-role
```



### 13.2.4 클러스터롤바인딩

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-clusterrolebinding
subjects:
  - kind: ServiceAccount
    name: myuser
    namespace: default
    apiGroup: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  name: read-clusterrole
```



### 13.2.5 다양한 롤의 권한 관리 확인하기

롤바인딩은 사용자와 롤을 묶어서 특정 네임스페이스에 권한을 할당하고,

클러스터롤바인딩은 사용자와 클러스터롤을 묶어서 쿠버네티스 클러스터에 권한을 할당합니다.



롤바인딩 실습 과정

```bash

kubectl apply -f ServiceAccount.yaml
kubectl apply -f Role.yaml
kubectl apply -f RoleBinding.yaml

kubectl get secret myuser-token-8glt9 -ojson | jq -r .data.token | base64 -d |pbcopy
kubectl config set-credentials myuser --token={{ TOKEN }}

kubectl config get-clusters
kubectl config set-context myuser-context --cluster=minikube --user=myuser
kubectl config get-contexts
kubectl config use-context myuser-context

kubectl get pods
```
