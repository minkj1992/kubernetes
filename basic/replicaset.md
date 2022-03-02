# 레플리카셋(Replicaset)
> 파드 복제 집합을 관리하는 논리적인 entity(개체)

파드는 본질적으로 일회성 singleton으로써, 동적으로 늘어나거나 줄어들 목적으로 만들어졌으며 이러한 관리를 제공해주기위해 파드들의 집합을 묶어 준 것이 레플리카 셋입니다. 이 때문에 *일반적으로 레플리카셋은 stateless 서비스를 위해 설계됐다 할 수 있습니다. 레플리카셋에 의해 생성된 요소는 상호 교환이 가능하며, scale down시 임의의 파드가 삭제되도록 선택됩니다. (p.169 쿠버네티스 시작하기 2/e)*  

- 약어로 `rs`로 사용하기도 합니다.

```bash
$ kubectl get rs
NAME                            DESIRED   CURRENT   READY   AGE
celery-worker-7b66b64           1         1         1       2d2h
django-backend-7d49cb6bdb       1         1         1       2d2h
flower-7787b4f984               1         1         1       2d2h
kube-event-watcher-5c474dd45    0         0         0       16d
kube-event-watcher-5ff7b979d4   0         0         0       16d
kube-event-watcher-6b9d8bdbb8   1         1         1       16d
react-frontend-856dcd657f       1         1         1       2d2h
```

## 주요 특징

- 레플리카셋(`rs`)은 클러스터 전반에 걸쳐 파드를 관리하는 역할을 수행해, 항상 올바른 타입과 개수의 파드가 실행되는 것을 보장해줍니다. (쿠버네티스 self healing의 근간)
- rs에 의해 관리되는 파드는 다음과 같은 상황에서 자동으로 리스케줄링 됩니다.
  - 노드 장애
  - `네트워크 파티션(network partition)`: 클러스터 일부 노드가 master api를 통해 다른 노드와 통신할 수 없는 상황
- 쿠버네티스 내부적으로 `조정 루프(reconciliation loop)`를 통해, rs가 pod들을 관리하도록 합니다.

## 조정 루프
> 쿠버네티스의 선언적 구성과 자동화(declarative configuration and automation)의 핵심

쿠버네티스는 선언적 구성(yaml)을 통해서 `원하는 상태`를 지정하고, 이를 지속적인 모니터링을 통해 `현재 상태`와 비교 하며 서로의 상태가 일치할 수 있도록 동작합니다.(자가 치유 시스템)

레플리카셋에서 pod를 관리할 때 사용되는 조정 루프는 간단한 단일 루프이지만, 노드가 사라지거나 장애가 발생했을 경우 이를다시 클러스터에 참여시키며, 사용자 요청에 따라 스케일 업/다운 처리를 합니다.

## 파드와 rs의 관계
쿠버네티스가 추구하는 핵심 주제 중 하나는 decoupling이듯, 파드와 레플리카셋의 관계는 느슨한 결합(loosely coupled)관계입니다.

즉 rs는 pods를 생성하고 관리하지만 소유하지는 않습니다.

- loosely coupled in Replicaset
  - 레플리카셋은 `라벨 쿼리(label query)`를 사용해 관리해야 할 파드들 식별
  - 레플리카셋은 파드 api를 활용해 파드 생성
  - **파드들을 로드밸런싱하는 서비스 api와 레플리카셋이 파드 생성 때 사용하는 api 서비스는 완전히 분리된 API 객체**

## 컨테이너 격리

실 서비스에서 오류를 발생하는 pod를 발견하였을 때, 간단한 해결 방법은 파들르 중지하는 것이지만 이는 개발자가 발생한 문제를 확인하기 위해 로그를 확인할 떄 문제가 생길 수 있다. **이 경우 문제가 발생한 파드의 label을 변경하면 rs은 파드가 삭제된 것으로 판단해 새로운 파드를 생성할 것이며, 문제가 발생한 파드 또한 격리되어 실행되게 된다.**

```bash
# Update pod 'foo' with the label 'status' and the value 'unhealthy', overwriting any existing value.
$ kubectl label --overwrite pods foo status=unhealthy
```
## 레플리카셋 명세(manifest)

- hello-rs.yaml
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: koin
spec: # rs 스펙
  replicas: 3 # want pod 3개로 명시
  selector:
    matchLabels: # select할 label (pod 라벨과 맞추어야 함)
      pod: django-backend
  template: # pod template 명시( pod 생성때 사용할 .yaml과 비슷)
    metadata:
      labels:
        pod: django-backend
        version: "1"
    spec: # 파드 스펙
      containers:
        - name: django-backend # 파드 
          image: idock.daumkakao.io/con/con-item-backend
          imagePullPolicy: Always
          ports:
            - containerPort: 8000
              protocol: TCP
          workingDir: /con-item/
```
## 레플리카셋 생성
레플리카셋은 레플리카셋 객체를 kubernetes API에 제출해 생성된다.

```bash
k apply -f <.레플리카셋yaml>
```

## 레플리카셋 세부 정보 확인
```
k describe rs <레플리카셋이름>
```

## 파드에서 레플리카셋 찾기
떄로는 파드가 레플리카셋에 의해 관리되고 있는지 여부와, 어떤 레플리카셋이 owner인지 궁금할 수 있다. 이런 검색을 가능하게 하기 위해 레플리카셋 컨트롤러는 자신이 생성하는 모든 파드에 annotation을 추가합니다.

**애노테이션 섹션에서 `kubernetes.io/created-by` 항목을 확인하면 됩니다.**

```
k get pods <파드이름> -o yaml # k8s 마스터가 덧붙인 yaml 파일 cat
```

## 레플리카셋에 대한 파드 집합 찾기

```bash
k get pods -l <라벨 셀렉터> # yaml에서 지정해준 셀렉터
```

**이 커맨드는 현재 파드의 수를 확인하기 위해 레플리카셋이 (조정루프에서) 실행하는 쿼리와 동일합니다.** 

## 레플리카 셋 확장

- HPA(heapster)
