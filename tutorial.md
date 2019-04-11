# 在 Google Cloud Shell 上建置 Istio 
## Task 1 環境建置 - 安裝 GKE

接下來的動作，將會協助您安裝與設定 Google Cloud Platform。您會建立一個新的專案，並安裝後續 Task 所需的 Google Kubernetes Engine(GKE)

## 1. 建立新的專案

### 將專案ID及名稱存為環境參數

```bash
PROJECT_ID=systex-lab-$(cat /proc/sys/kernel/random/uuid | cut -b -6) && \
echo "PROJECT_ID=$PROJECT_ID" | tee -a ~/.profile
```

### 使用 gcloud 指令建立新的專案

```bash
gcloud projects create $PROJECT_ID
```

在 Lab 完成後，您可以直接移除此專案，以省約費用


## 2. 將 Cloud Shell 連接至專案

```bash
gcloud config set project $PROJECT_ID
```

## 3. 設定預設的 region

預設建議為 asia-east1，所在地點為台灣彰濱。

```bash
DEFAULT_REGION=asia-east1 && \
echo "DEFAULT_REGION=$DEFAULT_REGION" | tee -a ~/.profile && \
gcloud config set compute/region $DEFAULT_REGION
```

## 4. 設定專案連結帳戶

您必需要為新建的專案連結付費帳戶設定，您可以執行以下指令完成，或至web page[Link a billing account](https://console.developers.google.com/billing/linkedaccount)

### 找尋已設定可用的 Billing account

```bash
BILLING_ACCOUNT=$(gcloud beta billing accounts list | grep True | awk -F" " '{print $1}')
```

### 設定專案連結付費帳戶

```bash
gcloud beta billing projects link $PROJECT_ID --billing-account $BILLING_ACCOUNT
```

## 安裝 Google 準備的 K8S 叢集

### 啟用 Kubernetes Engin API  

以下指令會啟用所有 Kubernetes Engine 所需要的 API 使用權限，大約需要 2min
；或是至Web page [啟用 Kubernetes Engin API](https://console.cloud.google.com/apis/library/container.googleapis.com)

```bash
gcloud services enable container.googleapis.com
```

### 安裝GKE

以下指令，會協助您建立 K8S 叢集，只有一個 worknodes，完成約需 3min

```bash
gcloud container clusters create ${PROJECT_ID}-k8s \
    --project=$PROJECT_ID \
    --machine-type=n1-standard-2 \
    --region=${DEFAULT_REGION} \
    --cluster-version=1.11.7-gke.12 \
    --num-nodes=1
```

請上網查看[相容的K8S版本](https://cloud.google.com/istio/docs/istio-on-gke/installing#supported_gke_cluster_versions)，設定到底下的 `--cluster-version`。 

_參數 --num-nodes 設定太大時，可能會請求不到資源_

##  安裝GKE後的設定
1. 授權使用者權限為 GKE的cluster-admin
```bash
kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value core/account)
```

2. 取得 K8S 叢集認證
```bash
gcloud container clusters get-credentials $PROJECT_ID-k8s --region=asia-east1
````

3. 查看 K8S 叢集
```bash
gcloud container clusters list
```

## Task 2 在GKE上部屬Istio
後續的功能演示需要部屬Istio，下列步驟將在K8S叢集中建立Istio的各項服務，並更改K8s部分原有架構，來達到對APP非侵入式控管與監控，中其中Istio架構如下：

![istio.png](imgs/istio.png)

我們將使用[Helm](https://helm.sh/ "Helm")來快速部屬Istio，Helm為K8s上常用的部屬工具

![helm.png](imgs/helm.png)

## 安裝Heml

```bash
wget https://storage.googleapis.com/kubernetes-helm/helm-v2.11.0-linux-amd64.tar.gz
```
```bash
tar xf helm-v2.11.0-linux-amd64.tar.gz
```
```bash
sudo cp linux-amd64/helm linux-amd64/tiller /usr/local/bin/
```
```bash
export PATH=$(pwd)/istio-$ISTIO_VERSION/bin:$(pwd)/linux-amd64/:$PATH
```
```bash
helm init --client-only
```

## 下載Istio 1.05
```bash
ISTIO_VERSION=1.0.5
```
```bash
echo "ISTIO_VERSION=1.0.5" | tee -a ~/.profile
```
```bash
wget https://github.com/istio/istio/releases/download/$ISTIO_VERSION/istio-$ISTIO_VERSION-linux.tar.gz
```
```bash
tar xvzf istio-$ISTIO_VERSION-linux.tar.gz
```

## 安裝Istio 1.05
```bash
echo "PATH=$(pwd)/GKE-Istio/istio-$ISTIO_VERSION/bin:$(pwd)/linux-amd64/:$PATH" | tee -a ~/.profile
```
```bash
kubectl create ns istio-system
```
```bash
helm template istio-$ISTIO_VERSION/install/kubernetes/helm/istio --name istio --namespace istio-system \
   --set servicegraph.enabled=true \
   --set tracing.enabled=true \
   --set sidecarInjectorWebhook.enabled=true \
   --set gateways.istio-ilbgateway.enabled=true \
   --set kiali.enabled=true \
   --set global.mtls.enabled=false  > istio.yaml
```
```bash
kubectl apply -f istio.yaml
```

### 補充說明
Istio可針對安全性需求，將微服務之間通信設為強制 mutual TLS 版本(本workshop使用非強制設定)
若需使用強制 mutual TLS 版本，於產生istio.yaml範本時可增加下列設定來達成

```
helm template istio-$ISTIO_VERSION/install/kubernetes/helm/istio --name istio --namespace istio-system \
   ...
   --set global.mtls.enabled=true  > istio.yaml
   ...
```
在需執行下列指令，完成設定
```
kubectl apply -f istio-$ISTIO_VERSION/install/kubernetes/helm/istio/templates/crds.yaml
kubectl apply -f istio-$ISTIO_VERSION/install/kubernetes/helm/istio/charts/certmanager/templates/crds.yaml
```

## 驗證 Istio 安裝結果
3. 驗證 Istio 安裝在命名空間 istio-system 的結果
```bash
kubectl get svc -n istio-system
```
   應該看到類似的結果
```
NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                                                                                   AGE
istio-citadel              ClusterIP      10.47.245.92    <none>        8060/TCP,9093/TCP                                                                                                         12s
istio-egressgateway        ClusterIP      10.47.248.129   <none>        80/TCP,443/TCP                                                                                                            12s
istio-galley               ClusterIP      10.47.248.109   <none>        443/TCP,9093/TCP                                                                                                          12s
istio-ingressgateway       LoadBalancer   10.47.248.117   <pending>     80:31380/TCP,443:31390/TCP,31400:31400/TCP,15011:30221/TCP,8060:32445/TCP,853:30663/TCP,15030:32010/TCP,15031:32633/TCP   12s
istio-pilot                ClusterIP      10.47.251.133   <none>        15010/TCP,15011/TCP,8080/TCP,9093/TCP                                                                                     12s
istio-policy               ClusterIP      10.47.255.244   <none>        9091/TCP,15004/TCP,9093/TCP                                                                                               12s
istio-sidecar-injector     ClusterIP      10.47.240.36    <none>        443/TCP                                                                                                                   12s
istio-statsd-prom-bridge   ClusterIP      10.47.247.135   <none>        9102/TCP,9125/UDP                                                                                                         12s
istio-telemetry            ClusterIP      10.47.242.73    <none>        9091/TCP,15004/TCP,9093/TCP,42422/TCP                                                                                     12s
promsd                     ClusterIP      10.47.241.188   <none>        9090/TCP                                          
```

## 驗證 Istio 安裝結果
4. 驗證 K8S Pods 已在運行
```bash
kubectl get pods -n istio-system
```
   請重複執行直到所有狀態都是Runing 或 Completed 
   應該看到類似的結果
```
NAME                                        READY   STATUS      RESTARTS   AGE
istio-citadel-555d845b65-xfdmj              1/1     Running     0          2d
istio-cleanup-secrets-8x2pl                 0/1     Completed   0          2d
istio-egressgateway-667d854c49-9q5dl        1/1     Running     0          2d
istio-galley-6c9cd5b8bb-4j4jk               1/1     Running     0          2d
istio-ingressgateway-6c796c5594-f972p       1/1     Running     0          2d
istio-pilot-77f74fc6f-rpbfj                 2/2     Running     0          2d
istio-policy-655b87fff-4wbwq                2/2     Running     0          2d
istio-security-post-install-tm2rm           0/1     Completed   1          2d
istio-sidecar-injector-668c9fb4db-p6lwt     1/1     Running     0          2d
istio-statsd-prom-bridge-5b645f6f4d-6pbgf   1/1     Running     0          2d
istio-telemetry-d9848f498-wf6kh             2/2     Running     0          2d
promsd-6b989699d8-l7jxt                 1/1     Running     0          2d
```

## 設定 Istio 使用範圍

將k8s的default namespace下的Pod，設為自動注入Sidecar，以加入Istio網格(service mesh)服務中

```bash
kubectl label namespace default istio-injection=enabled
```
![istio-k8s-namespace.png](imgs/istio-k8s-namespace.png)

而此workshop中k8s中會有3個namespaces
![k8s-namespaces.png](imgs/k8s-namespaces.png)


## Task 3 安裝 bookinfo 並簡易演示藍綠部屬
## 安裝 Istio 範例 bookinfo (1/3)
1. 設定 istio1.05 版本之參數
```bash
export ISTIO_LAST=istio-$ISTIO_VERSION
```
```bash
cd ~/GKE-Istio/istio-1.0.5
```
2. 設定 istioctl 路徑
```bash
echo "ISTIO_LAST=istio-$ISTIO_VERSION" | tee -a ~/.profile
```
3. 以下列指令打開設定檔 ~/GKE-Istio/bookinfo-only-have-veviews-v1.yaml。可以看到所有安裝的微服務版本都只有v1，後續將會增加佈署的版本。

```bash
less ~/GKE-Istio/bookinfo-only-have-veviews-v1.yaml
```


## 安裝 Istio 範例 bookinfo (2/3)

4. 安裝 bookinfo-only-have-veviews-v1.yaml
```bash
kubectl apply -f <(~/GKE-Istio/istio-1.0.5/bin/istioctl kube-inject -f ~/GKE-Istio/bookinfo-only-have-veviews-v1.yaml)
```
```bash
kubectl apply -f  ~/GKE-Istio/istio-1.0.5/samples/bookinfo/networking/bookinfo-gateway.yaml
```
5. 驗證 bookinfo 安裝
```bash
kubectl get services
```
   應該看到如下結果：
```
NAME                       CLUSTER-IP   EXTERNAL-IP   PORT(S)              AGE
details                    10.0.0.31    <none>        9080/TCP             6m
kubernetes                 10.0.0.1     <none>        443/TCP              7d
productpage                10.0.0.120   <none>        9080/TCP             6m
ratings                    10.0.0.15    <none>        9080/TCP             6m
reviews                    10.0.0.170   <none>        9080/TCP             6m
```
6. 更進一步驗證
```bash
kubectl get deployments,ing
```
   應該看到類似的結果
```
NAME                                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/details-v1       1         1         1            1           6m
deployment.extensions/productpage-v1   1         1         1            1           6m
deployment.extensions/ratings-v1       1         1         1            1           6m
deployment.extensions/reviews-v1       1         1         1            1           6m
```
7. 驗證 K8S Pods 已在運行
```bash
kubectl get pods
```
   請重複執行直到所有狀態都是Runing 或 Completed 
   應該看到類似的結果
   
8. 取得bookinfo網址 http://$GATEWAY_URL/productpage 並驗證
```bash
INGRESSGATEWAY=istio-ingressgateway \
&& INGRESSGATEWAY_LABEL=istio
```
```bash
export INGRESS_IP=$(kubectl -n istio-system get service istio-ingressgateway  \
-o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```
```bash
echo "INGRESS_IP=$INGRESS_IP" | tee -a ~/.profile
```
```bash
echo http://$INGRESS_IP/productpage
```

## 安裝 Istio 範例 bookinfo (3/3)

開啟 kaili 工具的Graph 比對
### 注意: 請開啟新的cloud shell 將kiali的port-forword指令貼在新開啟的shell執行
請在跟CloudShell同一個brower開啟新分頁連線到GCP 網站上 Kubernetes Engine --> 服務 --> kiali (點選)--> 通訊埠轉送 (點選) 
注意此時請copy 指令碼在頁面右上方開啟新的cloudshell 貼上指令碼執行
--> 在網頁預覽中開啟

![Image 2.jpg](imgs/Image%202.jpg)

![Image 3.jpg](imgs/Image%203.jpg)

![Image 4.jpg](imgs/Image%204.jpg)

![Image 4-1.jpg](imgs/Image%204-1.jpg)

![Image 4-1.jpg](imgs/Image%204-2.jpg)

![Image 5.jpg](imgs/Image%205.jpg)

帳密: admin / admin

Graph

![Image 6.jpg](imgs/Image%206.jpg)
![set-per-sec.jpg](imgs/set-per-sec.jpg)

##  bookinfo 藍綠部屬 (1/2)
1. 佈屬bookinfo含有bookinfo veviews-v1, veviews-v2 和veviews-v3
```bash
kubectl apply -f <(~/GKE-Istio/istio-1.0.5/bin/istioctl kube-inject -f  ~/GKE-Istio/$ISTIO_LAST/samples/bookinfo/platform/kube/bookinfo.yaml)
```
2. 驗證 是否新增新的微服務
```bash
kubectl get deployments,ing
```
3. 應該看到類似的結果
```
NAME                                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/details-v1       1         1         1            1           42m
deployment.extensions/productpage-v1   1         1         1            1           42m
deployment.extensions/ratings-v1       1         1         1            1           42m
deployment.extensions/reviews-v1       1         1         1            1           42m
deployment.extensions/reviews-v2       1         1         1            1           2m
deployment.extensions/reviews-v3       1         1         1            1           2m
```
4. 驗證 K8S Pods 已在運行
```bash
kubectl get pods
```
   請重複執行直到所有狀態都是Runing 或 Completed 
   應該看到類似的結果
   
5. 驗證 會看到有三個不同的星星顯示方式
```bash
echo http://$INGRESS_IP/productpage
```
6. kaili 畫面

![Image 7.jpg](imgs/Image%207.jpg)

##  bookinfo 藍綠部屬 (2/2)
### 假設V1版本出問題
7. 將疑似有問題的v1版本下線
```bash
kubectl delete -f <(~/GKE-Istio/istio-1.0.5/bin/istioctl kube-inject -f  ~/GKE-Istio/reviews-v1.yaml)
```
會看到v1 狀態為 如下Terminating
```
kubectl get po
NAME                              READY     STATUS        RESTARTS   AGE
details-v1-75754887d9-rdbrx       2/2       Running       0          5m
productpage-v1-7b96bbf89f-jssgp   2/2       Running       0          5m
ratings-v1-5b89496689-bgwkq       2/2       Running       0          5m
reviews-v1-5fb6b66dfb-wwfw8       2/2       Terminating   0          5m
reviews-v2-ff6679549-vx5qc        2/2       Running       0          5m
reviews-v3-67499898bb-j9xcw       2/2       Running       0          5m
```
8. 驗證 沒有星號的V1 版本消失了
```bash
echo http://$INGRESS_IP/productpage
```
9. kaili 畫面

![no-v1.jpg](imgs/no-v1.jpg)

10. 恢復 原本veviews有三個版本以利後續演示
```bash
kubectl apply -f <(~/GKE-Istio/istio-1.0.5/bin/istioctl kube-inject -f  ~/GKE-Istio/$ISTIO_LAST/samples/bookinfo/platform/kube/bookinfo.yaml)
```

## Task  DelayFault Injection
##  bookinfo 延遲故障注入(Delay Injection) (1/4)

延遲故障注入，可以模擬當網路出現延遲時，微服務的運作是否正常

本段將模擬 reviews v3 到 ratings v1 的過程中出現網路延遲，健康的 bookinfo 服務時間如下圖

![delay-inject-before.png](imgs/delay-inject-before.png)

##  bookinfo 延遲故障注入(Delay Injection) (2/4)
預設rule

```bash
kubectl apply -f ~/GKE-Istio/istio-1.0.5/samples/bookinfo/networking/destination-rule-all.yaml
```
```bash
kubectl apply -f ~/GKE-Istio/istio-1.0.5/samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml
```
執行以下指令，將注入 1秒的延遲

```bash
kubectl apply -f ~/GKE-Istio/fault-inject-delay.yaml
```

設定細節說明
```
* __spec.http.fault.delay.fixedDelay__ : 注入延遲的時間
* __spec.http.fault.delay.percent__ : 注入錯誤的流量百分比 (0 ~ 100)
* __spec.http.match.sourceLabels.version__ : 流量來源的版號

apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        fixedDelay: 1s
        percent: 100
    match:
    - sourceLabels:
        version: v3
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
```

##  bookinfo 延遲故障注入(Delay Injection) (3/4)

延遲故障注入後，可以明顯看到，通過 `reviews v3` 的流量，回應時間明顯增加，如下圖

![delay-inject-after.png](imgs/delay-inject-after.png)

##  bookinfo 延遲故障注入 (Delay Injection)(4/4)

接著要展示， __中斷故障注入__ ，在開始之前，請執行以下指令，取消原本注入的錯誤

```bash
kubectl delete -f ~/GKE-Istio/fault-inject-delay.yaml
```

##  bookinfo 中斷故障注入(Fault Injection) (1/4)

中斷故障注入，可以模擬當其它服務異常時，微服務的運作是否正常

本段將模擬 reviews v3 到 ratings v1 的過程中出現中斷異常，健康的 bookinfo 服務時間如下圖

![inject-before.png](imgs/inject-before.png)

##  bookinfo 延遲故障注入(Fault Injection) (2/4)

執行以下指令，將注入 1秒的延遲

```bash
kubectl apply -f ~/GKE-Istio/fault-inject.yaml
```

設定細節說明
```
* __spec.http.fault.delay.fixedDelay__ : 注入延遲的時間
* __spec.http.fault.delay.percent__ : 注入錯誤的流量百分比 (0 ~ 100)
* __spec.http.match.sourceLabels.version__ : 流量來源的版號
```

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      abort:
        httpStatus: 500
        percent: 100
    match:
    - sourceLabels:
        version: v3
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
```

##  bookinfo 中斷故障注入(Fault Injection) (3/4)

中斷故障注入後，可以明顯看到， `reviews v3` 往 `ratings v1` 的流量，全部都收到 `http: 500` 的中斷

![inject-after.png](imgs/inject-after.png)

##  bookinfo 中斷故障注入(Fault Injection) (4/4)

Cleanup

```bash
kubectl delete -f ~/GKE-Istio/fault-inject.yaml
```

## 刪除專案 
### 注意執行後專案就不見了也就不會再計費
### 但也請您確認已完成整個workshop內容
```bash
gcloud projects delete $PROJECT_ID
```
