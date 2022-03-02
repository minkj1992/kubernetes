# Resource
> [Best practice resource requests and limits](https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-resource-requests-and-limits)

- 언제나 request <= limit 해야함.

## CPU

CPU 리소스는 밀리 코르 단위로 정의됩니다. 

- 컨테이너를 실행하는 데 2 ​​개의 전체 코어가 필요한 경우 "2000m"값을 입력합니다. 컨테이너에 코어의 ¼ 만 필요한 경우 "250m"값을 입력.
- CPU 요청에 대해 명심해야 할 한 가지는 가장 큰 노드의 코어 수보다 큰 값을 입력하면 포드가 예약되지 않음.
- 앱이 다중 코어 (과학적 컴퓨팅 및 일부 데이터베이스가 떠오르는 경우)를 활용하도록 특별히 설계되지 않은 경우 일반적으로 CPU 요청을 '1'이하로 유지하고 더 많은 rs을 사용해 확장하는 것이 좋다.
- CPU는 `압축 가능한`리소스로 간주됨. 
  - 앱이 CPU 제한에 도달하기 시작하면 Kubernetes가 컨테이너 조절
  - 즉, CPU가 인위적으로 제한되어 앱 성능이 저하는 가능하지만, 종료되거나 제거되지는 않음

## Memory

메모리 리소스는 바이트 단위로 정의됩니다. 

- 일반적으로 메모리에 메비 바이트 값 (기본적으로 메가 바이트와 동일)을 제공하지만 바이트에서 페타 바이트까지 무엇이든 제공 가능
- CPU와 마찬가지로 노드의 메모리 양보다 큰 메모리 요청을 넣으면 포드가 예약되지 않습니다.
- 압축 불가(cpu와 달리)
- 메모리 값 초과 시 컨테이너 종료하지만, pod가 deployment, statefulSet, DaemonSet 같은 컨트롤러에서 관리되면 파드 교체 시도(restart)
  -  > 이 이유로 locust 부하 테스트에서 장고 pod가 죽는 현상이 발생한 것으로 생각됨

## 네임스페이스
> namespace level에서 리밋 설정 가능


## 자동 검사

- [VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
  - 수직 형 Pod 자동 확장 처리는 클러스터에 설치하고 Pod에 대한 올바른 요청 및 한도를 추정하는 구성 요소
  - 즉, 제한과 요청을 추정하기 위해 알고리즘을 만들 필요가 없음
- Goldilocks (helm)

## VPA

```bash
git clone https://github.com/kubernetes/autoscaler.git

```