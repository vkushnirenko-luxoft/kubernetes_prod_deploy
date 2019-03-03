Repo with deploy files. Here is short manual how to proceed

Requirements to proceed

* linux/macos terminal
* git
* terraform
* gcloud
* kubectl (version from gcloud is ok)
* helm
* Please follow "Before you begin" part of https://cloud.google.com/kubernetes-engine/docs/how-to/iam


### Deploy GKE
* Visit [https://console.cloud.google.com/apis/api/container.googleapis.com/overview](https://console.cloud.google.com/apis/api/container.googleapis.com/overview) end enable Kubernetes engine API and wait it to become available
* Edit and then run following command
```
gcloud container clusters create new-staging --create-subnetwork name=new-staging-0 --num-nodes 2 --enable-autoscaling --max-nodes=3 --min-nodes=2 --cluster-version latest --enable-network-policy --enable-autorepair --enable-ip-alias --enable-private-nodes --master-ipv4-cidr 172.16.0.0/28 --enable-master-authorized-networks --master-authorized-networks 46.4.94.145/32,68.183.74.205/32,159.224.49.180/32 --no-enable-basic-auth --issue-client-certificate --enable-legacy-authorization --zone=europe-west3-b --node-locations=europe-west3-b --project=kuna-stage
```

Adjust cluster-name `new-staging`

Adjust zone and node-locations.
 
Adjust --master-ipv4-cidr `172.16.1.0/28`, for example to `172.16.2.0/28`, you need to use gitlab IP and all your IPs.
 
Adjust project=`kuna-stage`

Adjust --master-authorized-networks `46.4.94.145/32,68.183.74.205/32,159.224.49.180/32`

command will take some time to finish.
You may adjust IP control whitelist later if any with following command

```
gcloud container clusters update real-staging --enable-master-authorized-networks --master-authorized-networks Full-IPs-List
```
* Enable RBAC
Adjust email

```
ADMIN_USER=av@arilot.com
PROJECT=$(gcloud config get-value project)
gcloud projects add-iam-policy-binding $PROJECT --member=user:$ADMIN_USER --role=roles/container.admin
kubectl create clusterrolebinding cluster-admin-binding --clusterrole cluster-admin --user $ADMIN_USER
```

### Deploy NAT gateway
* Adjust `CLUSTER_NAME`, `REGION`, `ZONE`

```
git clone https://github.com/GoogleCloudPlatform/terraform-google-nat-gateway
cd terraform-google-nat-gateway/examples/gke-nat-gateway

CLUSTER_NAME=new-staging
REGION=europe-west3
ZONE=europe-west3-b
echo "region = \"${REGION}\"" > terraform.tfvars
echo "zone = \"${ZONE}\"" >> terraform.tfvars
echo "gke_master_ip = \"$(gcloud container clusters describe ${CLUSTER_NAME} --zone ${ZONE} --format='get(endpoint)')\"" >> terraform.tfvars
echo "gke_node_tag = \"$(gcloud compute instance-templates describe $(gcloud compute instance-templates list --filter=name~gke-${CLUSTER_NAME} --limit=1 --uri) --format='get(properties.tags.items[0])')\"" >> terraform.tfvars
export GOOGLE_PROJECT=$(gcloud config get-value project)

gcloud auth application-default login

terraform init
terraform apply
```
Answer `yes` when prompted
You should get output like following

```
Apply complete! Resources: 8 added, 0 changed, 0 destroyed.

Outputs:

ip-nat-gateway = 35.246.244.250
```
* Here is how to check NAT gateway works

```
kubectl run example -i -t --rm --restart=Never --image centos:7 -- curl -s http://ipinfo.io/ip

```

### Configure datadog monitoring
* Init helm

```
helm init
```
* Provide permissions to helm:

Get tiller-rbac.yaml from repo [https://g.anuk.tech/kuna/infra/deploy/blob/master/common/tiller-rbac.yaml](https://g.anuk.tech/kuna/infra/deploy/blob/master/common/tiller-rbac.yaml) and then run

```
kubectl create -f tiller-rbac.yaml
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```
* go to datadog and generate API key
* Install datadog agent, adjust `apiKey`

```
helm install --namespace datadog --name datadog --set datadog.apiKey=XXXX,datadog.logLevel=DEBUG,datadog.resources.requests.cpu=50m,datadog.logsEnabled=true,datadog.logsConfigContainerCollectAll=true stable/datadog
```
### Configure CI/CD
* Create GCP service account with limited permissions to access GKE

```
ACCOUNT=gitlab-ci
PROJECT=$(gcloud config get-value project)
gcloud --project $PROJECT iam roles create gke_ci --permissions container.deployments.list,container.deployments.get,container.deployments.getScale,container.deployments.getStatus,container.deployments.rollback,container.deployments.update,container.deployments.updateScale,container.deployments.updateStatus,container.clusters.get
gcloud --project $PROJECT iam service-accounts create $ACCOUNT
gcloud --project $PROJECT iam service-accounts keys create $PROJECT-$ACCOUNT.json --iam-account=${ACCOUNT}@${PROJECT}.iam.gserviceaccount.com
gcloud projects add-iam-policy-binding $PROJECT --member serviceAccount:${ACCOUNT}@${PROJECT}.iam.gserviceaccount.com --role projects/${PROJECT}/roles/gke_ci
gcloud projects add-iam-policy-binding $PROJECT --member serviceAccount:${ACCOUNT}@${PROJECT}.iam.gserviceaccount.com --role roles/storage.admin
echo GCE_SA_PROD=${ACCOUNT}@${PROJECT}.iam.gserviceaccount.com
echo GCE_KEY_PROD=$(cat ./$PROJECT-$ACCOUNT.json)
```
* Add variables `GCE_SA_PROD` and `GCE_KEY_PROD` to Gitlab CI to **each** project we need to deploy to cluster: `Project Settings -> CI/CI -> Variables, expand`

### Deploy api and fron inside k8s
```
#create namespace
kubectl create ns kuna

#clone repo with manifests
git clone https://g.anuk.tech/kuna/infra/deploy.git

#apply front
cd deploy/staging/main-front/k8s/
kubectl -n kuna apply -f .
cd ../..

#apply api
cd api/etc
./apply.sh 
cd ../k8s
kubectl -n kuna apply -f .

#apply ingress to both api and front
cd ../..
cd ingress/etc
./apply.sh 

cd ../k8s
kubectl -n kuna apply -f .
```
You shoud get output with ingress IP. You may point DNS or proxy to this IP
