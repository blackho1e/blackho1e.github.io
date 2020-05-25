---
layout: post
title: Helm 차트 github private 리포지토리에 배포 방법
---

Helm은 Kubernetes의 훌륭한 패키지 관리자입니다. 복잡한 설정에서 특별히 k8s를 사용하는 경우 helm은 멋진 release 프로세스를 만드는데 도움이 될수 있습니다.

## Github에 Helm 차트 리포지토리 만들기
저장소를 작성하고 패키지를 추가하려면 다음 명령을 따르십시오. (helm 설치하는 방법에 대해서는 [Installing Helm](https://helm.sh/docs/intro/install/)를 참조하십시오. )
```
$ helm package $YOUR_CHART_PATH/  # 해당 경로에 yaml 파일들을 압축한 $HELM_NAME.tgz 파일이 생성됩니다.

# 리포지토리에 index.yaml을 작성하거나 아니면 다음 명령어로 업데이트를 합니다.
$ helm repo index .

# private 리포지토리에선 다음 명령어로 업데이트를 합니다.
$ $ helm repo index $YOUR_CHART_PATH/ --url https://raw.githubusercontent.com/blackho1e/blackho1e.github.io/master/charts/stable/

# 생성된 index.yaml와 $HELM_NAME.tgz를 github에 배포 하세요.
$ git add .
$ git commit -m 'New chart version'
$ git push
```

## Github에 배포된 차트를 엑세스 사용하는 방법
차트 리포지토리 추가
```
$ helm repo add blackho1e 'https://blackhole.github.io/charts/stable/'
$ helm repo update
$ helm search $HELM_NAME
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
blackho1e/$HELM_NAME    0.1.2           1.0             A Helm chart for Kubernetes
```
차트 private 리포지토리 추가 방법
```
$ helm repo add --username $USERNAME --password $GITHUB_TOKEN blackho1e 'https://raw.githubusercontent.com/blackho1e/blackho1e.github.io/master/charts/stable/'
$ helm repo update
$ helm search $HELM_NAME
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
blackho1e/$HELM_NAME    0.1.2           1.0             A Helm chart for Kubernetes
```


## Helm에 배포된 차트를 이용해서 리소스를 설치하는 방법
helm 차트 리포지토리를 추가후 다음 명령어를 이용해서 Kubernetes에 설치 하세요.
```
$ helm install --name blackho1e/$HELM_NAME -f values.yaml
```
