## 쿠버네티스 환경 취득

구글 계정이 있으면, 기본 1년치 계정을 얻을 실 수 있습니다 - [Google Cloud Platform account](https://cloud.google.com/), 

프로젝트를 하나 생성하신 후 [Project](https://cloud.google.com/resource-manager/docs/creating-managing-projects), GCloud SDK 를 설정하세요 - [gcloud SDK](https://cloud.google.com/sdk/).

아래의 gcloud 명령으로 쿠버네티스 클러스터를 생성할 수 있습니다:
```
$ export ISTIO_PROJECT_ID=$(gcloud config get-value core/project)
$ gcloud --project=$ISTIO_PROJECT_ID alpha container clusters create istio-cluster \
  --zone=us-central1-c --num-nodes=4 --machine-type=n1-standard-4 \
  --cluster-version=1.9.2-gke.1
```

클러스터가 생성된 후에 내 gcloud 계정에 쿠버네티스의 어드민 권한을 부여해주어야 이후 작업들을 수행할 수 있습니다:
```
$ kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value core/account)
```

## Install Istio

이스티오를 설치합니다. [Istio 0.5.1 release](https://github.com/istio/istio/releases/tag/0.5.1).
패키지를 Unpack 하고 istioctl 명령을 PATH에 잡아줍니다:

```
curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.0.0 sh -
cd ./istio-*
export PATH=$PWD/bin:$PATH  # istioctl 명령을 어디서나 쓸 수 있게

```

CRD와 Auth 모듈을 제외한 Istio 플러그인 들을 설치해줍니다. (현재 Auth 가 liveness 및 readiness check 에 문제가 좀 있다고 합니다)
```
# install Custom Resource Definitions for istio 
kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml
sleep 5
kubectl apply -f install/kubernetes/istio-demo-auth.yaml
```

namespace 로 istio-by-java 를 만들어주고, 해당 namespace 가 자동으로 istio 의 영향이 적용되도록 해줍니다.
```
kubectl create namespace istio-by-java
kubectl label namespace istio-by-java istio-injection=enabled
```

이스티오가 제대로 설치되었는지 확인한 후:
```
$ kubectl get pods -n istio-system
```

istio-ingress gateway 에 대해 방화벽을 열어줍니다.:
```
$ gcloud --project=$ISTIO_PROJECT_ID compute firewall-rules create allow-istio-ingress \ 
  --allow tcp:$(kubectl get svc istio-ingress -n istio-system -o jsonpath='{.spec.ports[0].nodePort}')
```
