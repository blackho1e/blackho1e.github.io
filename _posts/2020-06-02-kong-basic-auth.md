---
layout: post
title: Kong 인그레스 Basic Auth 적용
image: /assets/img/kong.png
tags: [kubernetes, kong, ingress]
---

## Kong 인그레스 Basic Auth 적용

## 샘플 서비스 설정
설명하기위해 클러스터에 httpbin 서비스를 설정하고 프록시합니다.
```bash
$ kubectl apply -f https://bit.ly/k8s-httpbin
service/httpbin created
deployment.apps/httpbin created
```
방금 만든 httpbin 서비스를 프록시하기 위해 인그레스 규칙을 만듭니다.
```bash
$ echo '
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: demo
  annotations:
    konghq.com/strip-path: "true"
    kubernetes.io/ingress.class: kong
spec:
  rules:
  - http:
      paths:
      - path: /foo
        backend:
          serviceName: httpbin
          servicePort: 80
' | kubectl apply -f -
ingress.extensions/demo created
```
아이피를 확인 하십시오.
```bash
$ kubectl get ingress demo
NAME   HOSTS   ADDRESS         PORTS   AGE
demo   *       35.XXX.XXX.XX   80      80s
```
인그레스 규칙을 테스트 하십시오.
```bash
$ curl -i 35.XXX.XXX.XX/foo/status/200
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 0
Connection: keep-alive
Server: gunicorn/19.9.0
Date: Tue, 02 Jun 2020 00:47:08 GMT
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
X-Kong-Upstream-Latency: 10
X-Kong-Proxy-Latency: 1
Via: kong/2.0.4
```

## 서비스에 인증 추가
Kong을 사용하면 API 앞에 인증을 추가하는 것이 플러그인을 활성화하는것만큼 간단합니다. API를 보호하기 위해 KongPlugin 리소스와 Secret를 추가하겠습니다 :
```bash
$ echo '
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: httpbin-auth
config:
  hide_credentials: true
plugin: basic-auth
' | kubectl apply -f -
kongplugin.configuration.konghq.com/httpbin-auth created
```
```bash
$ echo '
apiVersion: v1
kind: Secret
metadata:
  name: httpbin-secret
type: Opaque
data:
  kongCredType: YmFzaWMtYXV0aAo=
  username: ZGVtbwo=
  password: MTIzNDUK
' | kubectl apply -f -
secret/httpbin-secret created
```
또는 다음 명령어로 생성하십시오.
```bash
$ kubectl create secret generic httpbin-secret \
  --from-literal=kongCredType=basic-auth \
  --from-literal=username=demo --from-literal=password=12345
```

이제 이플러그인을 konghq.com/plugins annotation을 사용하여 생성 한 이전 Ingress 규칙과 연결하십시오.
```bash
$ kubectl patch ingress demo -p '{"metadata":{"annotations":{"konghq.com/plugins":"httpbin-auth"}}}'
ingress.extensions/demo patched
```
demo 인그레스에 정의 된 프록시 규칙과 일치하는 모든 요청에는 이제 유효한 API 키가 필요합니다.
```bash
$ curl -i 35.XXX.XXX.XX/foo/status/200
HTTP/1.1 401 Unauthorized
Date: Tue, 02 Jun 2020 01:03:33 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
WWW-Authenticate: Basic realm="kong"
Content-Length: 26
X-Kong-Response-Latency: 0
Server: kong/2.0.4

{"message":"Unauthorized"}
```

## 참고자료
[https://github.com/Kong/kubernetes-ingress-controller/blob/master/docs/guides/using-consumer-credential-resource.md](https://github.com/Kong/kubernetes-ingress-controller/blob/master/docs/guides/using-consumer-credential-resource.md)

