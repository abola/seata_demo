# seata_demo

## 參考資料及原始碼來源

* https://github.com/seata/seata
* https://github.com/seata/seata-k8s/
* https://github.com/seata/seata-docker
* https://github.com/seata/seata-samples
* https://github.com/seata/seata-samples/tree/docker/springboot-dubbo-fescar 

---

## 測試環境條件建議需求

### BUILD

* Docker 1.18
* OpenJDK 1.8

### DEPLOY

測試過程主要於 GCP 中完成，會使用 GKE(Google Kubernetes Engine) 及 GCR(Google Container Registry)，但這不是必要的，只要環境符合以下需求即可

* Helm 
* Kubernetes 1.11+ with 2vCPU + 4GRAM
* Seata server container: 1x vCPU + 2G RAM 以上

---

## 建立 GCP 環境 (optional)

以下代碼必需於 Google Cloud Shell 中執行。最終目的，只是要取得一座 kubernetes 來運行測試展示。 

```bash
PROJECT_NAME=seata-demo-$(cat /proc/sys/kernel/random/uuid | cut -b -6)

# Create seata demo project
gcloud projects create $PROJECT_NAME
gcloud config set project $PROJECT_NAME

# Require billing-account 
gcloud beta billing projects link $PROJECT_NAME --billing-account $(gcloud beta billing accounts list | grep True | awk -F" " '{print $1}')

# Enable Container API
gcloud services enable container.googleapis.com --project=$PROJECT_NAME

# Create GKE (3-5mins needed)
gcloud container clusters create seata-demo \
        --project=$PROJECT_NAME \
        --machine-type=n1-standard-2 \
        --zone=asia-east1-c \
        --num-nodes=3 \
        --disk-type "pd-standard" \
        --disk-size "100"

kubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole=cluster-admin \
    --user=$(gcloud config get-value core/account)
```

---

## Build Docker images

### 原始碼取得

下載測試展示的完整資料

```bash
git clone https://github.com/abola/seata_demo.git
cd seata_demo/
```

###  建立 Server docker image (Seata+Nacos+MySQL)

(Optional) 如果您是使用 GCP 環境，請先指定 GCR 位置。若是使用其它 Docker registry ，請自行調整。
```bash
DOCKER_REGISTRY="gcr.io/${PROJECT_NAME}/"
```

Server image

```bash
# build image
docker build -t ${DOCKER_REGISTRY}seata:0.9.0 build/docker/seata-server/

# push image to GCR
docker push ${DOCKER_REGISTRY}seata:0.9.0 
```

###  建立 Samples client docker image

```bash
# Set samples version
SAMPLES_VERSION=1.0.0
SAMPLES_PATH=build/docker/samples-client/springboot-dubbo-seata

# build java source (10-15mins needed, first time only )
mvn -f ${SAMPLES_PATH}/pom.xml clean package

# build image
docker build -t ${DOCKER_REGISTRY}samples-account:${SAMPLES_VERSION} ${SAMPLES_PATH}/samples-account/
docker build -t ${DOCKER_REGISTRY}samples-business:${SAMPLES_VERSION} ${SAMPLES_PATH}/samples-business/
docker build -t ${DOCKER_REGISTRY}samples-order:${SAMPLES_VERSION} ${SAMPLES_PATH}/samples-order/

# push image to GCR
docker push ${DOCKER_REGISTRY}samples-account:${SAMPLES_VERSION}
docker push ${DOCKER_REGISTRY}samples-business:${SAMPLES_VERSION}
docker push ${DOCKER_REGISTRY}samples-order:${SAMPLES_VERSION}
```

---

## Deploy Server & Samples

### 安裝 Helm 

```bash
curl -L https://git.io/get_helm.sh | bash
```

### 先啟動 Seata server

```bash
helm template deploy/seata-server \
    --set server.fescar.image.registry=$DOCKER_REGISTRY \
    | kubectl apply -f -
```

### 確認 Server 啟動

### 匯入Samples 資料表進入 MySQL 

```bash
cat ${SAMPLES_PATH}/sql/db_seata.sql | kubectl exec -i $(kubectl get pods --selector app=seata --output=jsonpath={.items..metadata.name}) -c mysql -- mysql -uroot --password=root db_gts_fescar
```

### 啟動 Samples client

```bash
helm template deploy/samples-client | kubectl apply -f -
```


```
kubectl get pods --selector app=seata-business --output=jsonpath={.items..metadata.name}
```

## 測試驗證 

扣 10個水杯 共100元

```bash
kubectl exec -i sleep-56869bd5f5-4bkt9 -- curl -H "Content-Type: application/json" -X POST --data "{\"userId\":\"1\",\"commodityCode\":\"C201901140001\",\"count\":10,\"amount\":100}"   seata-business-service:8104/business/dubbo/buy
```

查看資料庫

```
echo 'select * from t_account;'| kubectl exec -i $(kubectl get pods --selector app=seata --output=jsonpath={.items..metadata.name}) -c mysql -- mysql -uroot --password=root db_gts_fescar && \
echo 'select * from t_order;'| kubectl exec -i $(kubectl get pods --selector app=seata --output=jsonpath={.items..metadata.name}) -c mysql -- mysql -uroot --password=root db_gts_fescar && \
echo 'select * from t_storage;'| kubectl exec -i $(kubectl get pods --selector app=seata --output=jsonpath={.items..metadata.name}) -c mysql -- mysql -uroot --password=root db_gts_fescar && \
echo 'select * from undo_log;'| kubectl exec -i $(kubectl get pods --selector app=seata --output=jsonpath={.items..metadata.name}) -c mysql -- mysql -uroot --password=root db_gts_fescar 
```