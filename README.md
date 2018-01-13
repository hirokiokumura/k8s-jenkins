# 参考サイト
* [Kubernetes Engineでの Jenkins の設定  |  ソリューション  |  Google Cloud Platform](https://cloud.google.com/solutions/jenkins-on-container-engine-tutorial?hl=ja)

# 構築
## 環境の準備
```
git clone https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes.git
cd continuous-deployment-on-kubernetes
```

## Kubernetes クラスタの作成
* Kubernetes Engineクラスタが接続して使用する Compute Engine ネットワークを作成します。
```
gcloud compute networks create jenkins --mode auto
```

* Kubernetes Engineを使用して Kubernetes クラスタをプロビジョニングします。
```
gcloud container clusters create jenkins-cd \
--machine-type=n1-standard-1 \
--zone=asia-northeast1-a \
--num-nodes=3 \
--network=jenkins \
--disk-size=100 \
--cluster-version=1.8.5-gke.0 \
--username=admin \
--password=1234567890123456 \
--scopes "https://www.googleapis.com/auth/projecthosting,storage-rw"
```

## Jenkins ホーム ボリュームの作成
```
gcloud compute images create jenkins-home-image --source-uri https://storage.googleapis.com/solutions-public-assets/jenkins-cd/jenkins-home-v2.tar.gz
gcloud compute disks create jenkins-home --image jenkins-home-image --zone asia-northeast1-a
```

## Jenkins 認証情報の設定
* デフォルトの Jenkins ユーザー向けのパスワードを設定します。
```
PASSWORD=`openssl rand -base64 15`; echo "Your password is $PASSWORD"; sed -i.bak s#CHANGE_ME#$PASSWORD# jenkins/k8s/options
Your password is lToM+xmEyU5/tjvYGVv2
```

* Jenkins 向けの Kubernetes 名前空間
```
$kubectl create ns jenkins
```

* Kubernetes シークレットを作成します。Kubernetes はこのオブジェクトを使用して、Jenkins が起動したときに、Jenkins にデフォルトのユーザー名とパスワードを提供します。
```
$kubectl create secret generic jenkins --from-file=jenkins/k8s/options --namespace=jenkins
```

## Jenkins デプロイメントとサービスの作成
```
$kubectl apply -f jenkins/k8s/
```


## podがPendingのままで調べたらCPUが足りなかった。原因は1ノードで作成していたので3ノードでクラスタ作成し直した
```
$kubectl describe nodes gke-jenkins-cd-default-pool-7891e3af-dm0j

Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  1m (x57 over 17m)  default-scheduler  No nodes are available that match all of the predicates: Insufficient cpu (1).
```

## 外部負荷分散の設定
### 暗号化の設定
```
$openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /tmp/tls.key -out /tmp/tls.crt -subj "/CN=jenkins/O=jenkins"
$kubectl create secret generic tls --from-file=/tmp/tls.crt --from-file=/tmp/tls.key --namespace jenkins
```

### ロードバランサの作成
```
$kubectl apply -f jenkins/k8s/lb/ingress.yaml
$kubectl describe ingress jenkins --namespace jenkins
```
