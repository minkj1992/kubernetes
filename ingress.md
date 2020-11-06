# 인그레스
> 인그레스를 통한 HTTP 로드밸런싱

- 서비스 객체는 OSI 4계층(TCP, UDP)
- 인그레스는 7계층으로 동작함

## 기본

```yaml
apiVersion: extensions/v1beta1
kind: ingress
metadata:
  name: simple-ingress
spec:
  backend:
    serviceName: simple-svc
    servicePort: 8080
```

모든 요청은 simple-svc로 들어가게 된다.

## spec.rules.host 사용
> HTTP 호스트 헤더를 보고 해당 헤더를 기반으로 트래픽을 전달

```yaml
apiVersion: extensions/v1beta1
kind: ingress
metadata:
  name: host-ingress
spec:
  rules:
  - host: host.example.com
    http:
      paths:
      - backend:
        serviceName: host-svc
        servicePort: 8080
```

- `default-htto-backend`
  - 다른 방식으로 처리되지 않는 요청을 처리하기 위해 일부 인그레스 컨트롤러에서만 사용하는 규칙입니다. 즉 처리되지 않을(요청한 path와 맞는 패턴이 없는 경우) 요청에 대해 kube-system 네임스페이스의 default-http-backend 서비스로 요청을 보내게 됩니다.

## spec.rules.http.paths.path 사용
> 호스트 이름뿐만 아니라, HTTP 요청의 경로를 기반으로 트래픽 전달

```yaml
apiVersion: extensions/v1beta1
kind: ingress
metadata:
  name: path-ingress
spec:
  rules:
  - host: host.example.com
    http:
      paths:
      - path: "/"
        backend:
          serviceName: front-svc
          servicePort: 3000
      - path: "/api/"
        backend:
          serviceName: api-svc
          servicePort: 8080
```

- 인그레스 시스템에 나열된 동일한 호스트에 여러 경로가 있을 경우, 가장 구체화된 주소부터 규칙이 적용됩니다.
- **요청이 업스트림 서비스로 프록시 될떄 경로는 수정되지 않은 상태로 유지됩니다.** 
  - 예를 들어 host.example.com/api/user/10" 인 host가 호출될 떄 api 업스트림에는 /api/user/10 이라는 path가 전달됩니다.
- 만약 경로가 수정되기 원한다면 path rewriting 기능을 사용해야 합니다. (nginx controller 사용의 경우 rewrite-target)

## 다중-인그레스-컨트롤러 사용
> 단일 클러스터에서 여러 개의 인그레스-컨트롤러를 실행할 수 있다.

이 경우, `kubernetes.io/ingress.class(k8s 1.18 이상 부터는 ingressClassName)` 어노테이션을 사용하여 특정 인그레스 객체가 어떤 인그레스-컨트롤러를 사용해야 하는지를 지정할 수 있다. 

## 다중-인그레스 사용(다중 인그레스 객체)

여러 개의 인그레스 객체를 지정하면 인그레스 컨트롤러가 해당 객체들을 읽고, 일관된 configuration(설정)으로 병합해야 합니다.

- 중복되고 충돌하는 설정을 사용할 경우 동작이 지정되지 않는다.

## 인그레스와 네임스페이스

- **인그레스는 동일한 네임스페이스에 있는 업스트림 서비스만 참조할 수 있습니다.**
  - 즉 인그레스 객체 사용 시 다른 네임스페이스에 있는 서비스에 대해 path 로 참조할 수 없습니다.
  - **하지만 예외적으로 헤드리스 서비스를 사용한다면, 다른 네임스페이스에 존재하는 서비스를 현재 ingress의 하위 서비스로 사용 가능합니다.** [참고링크](https://coffeewhale.com/kubernetes/service/2020/01/22/headless-svc/)
- 다른 네임스페이스에 위치한 여러 인그레스 객체는 동일한 호스트에 대하여 하위 경로(path)로 지정 가능합니다.
  - 즉 동일한 example.com에 대해 다른 네임스페이스에 존재하는 인그레스들이 접근 가능합니다.
  - 이러한 특성 때문에 네임스페이스 간 동작은 클러스터 전체에서 고려하여 적용되어야 합니다. (한 네임스페이스의 인그레스 객체가 다른 네임스페이스에서 문제를 유발 할 수 있기 때문)

## TLS( `secret` )

TLS 인증서와 키를 사용해 시크릿`secret`을 정의할 수 있습니다.

`kubectl create secret tls <시크릿 이름>  --cert <인증서 pem 파일> -key <private키 pem 파일 >`


- tls-secret.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  creationTimestamp: null
  name: tls-secret-name
type: kubernetes.io/tls
data:
  tls.crt: <base64 encoded 인증서>
  tls.key: <base64 encoded 개인 키>
```

- tls-ingress.yaml
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - a.example.com
    secretName: tls-secret-name
  rules:
  - host: a.example.com
    http:
      paths:
      - backend:
        serviceName: a-svc
        servicePort: 8080
```

- 인증서가 업로드되면 인그레스 객체에서 인증서를 참조할 수 있습니다.
- **주의할 점은 여러 인그레스 객체가 동일한 호스트 이름에 대한 인증서를 지정하면 동작이 정의되지 않습니다.**


## nginx ingress controller

동작은 기본적으로 NGINX-INGRESS-CONTROLLER가 인그레스 객체를 읽고 NGINX conf로 병합합니다. 이후 NGINX 프로세스에 신호를 보내 새로운 conf로 재시작을 합니다. (사용중인 기존 연결을 그대로 서비스함)

