---
layout: post
title: 쿠버네티스(Kubernetes) cert-manager로 Let's Encrypt 인증서 발급
image: /assets/img/cert-manager.png
---

## Kong 인그레스 설치
```bash
$ kubectl create -f https://bit.ly/k4k8s
namespace/kong created
customresourcedefinition.apiextensions.k8s.io/kongplugins.configuration.konghq.com created
customresourcedefinition.apiextensions.k8s.io/kongconsumers.configuration.konghq.com created
customresourcedefinition.apiextensions.k8s.io/kongcredentials.configuration.konghq.com created
customresourcedefinition.apiextensions.k8s.io/kongingresses.configuration.konghq.com created
serviceaccount/kong-serviceaccount created
clusterrole.rbac.authorization.k8s.io/kong-ingress-clusterrole created
clusterrolebinding.rbac.authorization.k8s.io/kong-ingress-clusterrole-nisa-binding created
configmap/kong-server-blocks created
service/kong-proxy created
service/kong-validation-webhook created
deployment.extensions/kong created
```

## cert-manager 설치
클러스터에 cert-manager를 설치하는 방법에 대해서는 cert-manager의 [설명서](https://docs.cert-manager.io/en/latest/getting-started/install/kubernetes.html)를 참조하십시오. 

```bash
$ kubectl get all -n cert-manager
NAME                                           READY   STATUS    RESTARTS   AGE
pod/cert-manager-79dcdb95bb-kl9rk              1/1     Running   0          23d
pod/cert-manager-cainjector-757548979c-cqlwd   1/1     Running   0          23d

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/cert-manager   ClusterIP   10.35.242.140   <none>        9402/TCP   67d

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cert-manager              1/1     1            1           67d
deployment.apps/cert-manager-cainjector   1/1     1            1           67d

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/cert-manager-79dcdb95bb              1         1         1       67d
replicaset.apps/cert-manager-cainjector-757548979c   1         1         1       67d
```

## 데모용 echo 서비스 설치
```bash
$ kubectl apply -f https://bit.ly/echo-service
service/echo created
deployment.apps/echo created
```

## 무료 도메인 생성
다음 사이트에서 무료 도메인을 생성 합니다. 

https://xn--220b31d95hq8o.xn--3e0b707e/

## DNS 설정
Kong 로드 밸런서 IP 주소를 가져옵니다.
```bash
$ kubectl get service -n kong kong-proxy
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                      AGE
kong-proxy   LoadBalancer   10.35.251.209   35.221.109.88   80:32118/TCP,443:31462/TCP   3d16h
```

IP 주소만 얻으려면:
```bash
$ kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].ip}" service -n kong kong-proxy
35.221.109.88
```

위에 사으트에서 생성한 도메인을 Kong 로드 밸런서 IP (35.221.109.88) 로 설정 합니다.

DNS IP가 잘 설정되었는지 확인 방법은:
```bash
$ dig +short demo.infose.kro.kr
35.221.109.88
```

## Let's Encrypt 인증서 생성
cert-manager를 위해 ClusterIssuer를 설정하십시오.
```bash
echo "apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: demo.infose.kro.kr
  namespace: cert-manager
spec:
  acme:
    email: blackdole@naver.com
    privateKeySecretRef:
      name: demo.infose.kro.kr
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
    - http01:
        ingress: {}" | kubectl apply -f -
clusterissuer.cert-manager.io/demo.infose.kro.kr configured
```

## 응용 프로그램 인터넷에 노출
```bash
$ echo "apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: demo-ingress
  annotations:
    kubernetes.io/tls-acme: "true"
    cert-manager.io/cluster-issuer: demo.infose.kro.kr
spec:
  tls:
  - hosts:
    - demo.infose.kro.kr
    secretName: demo.infose.kro.kr-tls
  rules:
    - host: demo.infose.kro.kr
      http:
        paths:
        - path: /
          backend:
            serviceName: echo
            servicePort: 80" | kubectl apply -f -
ingress.extensions/demo-ingress configured
```

## 테스트 HTTPS
```bash
$ curl -v https://demo.infose.kro.kr
*   Trying 35.221.109.88:443...
* Connected to demo.infose.kro.kr (35.221.109.88) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/certs/ca-certificates.crt
  CApath: none
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=demo.infose.kro.kr
*  start date: May  1 03:00:44 2020 GMT
*  expire date: Jul 30 03:00:44 2020 GMT
*  subjectAltName: host "demo.infose.kro.kr" matched cert's "demo.infose.kro.kr"
*  issuer: C=US; O=Let's Encrypt; CN=Let's Encrypt Authority X3
*  SSL certificate verify ok.
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x560e462478b0)
> GET / HTTP/2
> Host: demo.infose.kro.kr
> user-agent: curl/7.69.1
> accept: */*
>
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* old SSL session ID is stale, removing
* Connection state changed (MAX_CONCURRENT_STREAMS == 128)!
< HTTP/2 200
< content-type: text/plain; charset=UTF-8
< date: Fri, 01 May 2020 04:22:28 GMT
< server: echoserver
< x-kong-upstream-latency: 2
< x-kong-proxy-latency: 1
< via: kong/2.0.3
<


Hostname: echo-758859bbfb-c6446

Pod Information:
        node name:      gke-crawler-1-default-pool-917bc6f9-r1qh
        pod name:       echo-758859bbfb-c6446
        pod namespace:  default
        pod IP: 10.32.2.95

Server values:
        server_version=nginx: 1.12.2 - lua: 10010

Request Information:
        client_address=10.32.1.74
        method=GET
        real path=/
        query=
        request_version=1.1
        request_scheme=http
        request_uri=http://demo.infose.kro.kr:8080/

Request Headers:
        accept=*/*
        connection=keep-alive
        host=demo.infose.kro.kr
        user-agent=curl/7.69.1
        x-forwarded-for=106.247.255.90
        x-forwarded-host=demo.infose.kro.kr
        x-forwarded-port=8443
        x-forwarded-proto=https
        x-real-ip=106.247.255.90

Request Body:
        -no body in request-

* Connection #0 to host demo.infose.kro.kr left intact
```
