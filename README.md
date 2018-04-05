# kubernetes

## kops install
```
export VERSION=$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)
curl -LO https://github.com/kubernetes/kops/releases/download/${VERSION}/kops-linux-amd64
chmod +x kops-linux-amd64 && sudo mv kops-linux-amd64 /usr/local/bin/kops
```

## kops usage
```
export KOPS_CLUSTER_NAME=kube.nalbam.com
export KOPS_STATE_STORE=s3://clusters.kube.nalbam.com

aws route53 create-hosted-zone --name ${KOPS_CLUSTER_NAME} --caller-reference ${KOPS_CLUSTER_NAME}

aws s3 mb ${KOPS_STATE_STORE}

kops create cluster \
    --name ${KOPS_CLUSTER_NAME} \
    --state ${KOPS_STATE_STORE} \
    --master-size t2.small \
    --node-size t2.medium \
    --node-count 1 \
    --zones ap-northeast-2a,ap-northeast-2c \
    --dns-zone nalbam.com \
    --network-cidr 10.20.0.0/16 \
    --networking calico

kops get cluster

kops edit cluster ${KOPS_CLUSTER_NAME}

kops update cluster ${KOPS_CLUSTER_NAME} --yes

kops validate cluster

watch kubectl -n kube-system get node,pod,svc

kops delete cluster ${KOPS_CLUSTER_NAME} --yes
```
 * https://github.com/kubernetes/kops
 * https://kubernetes.io/docs/getting-started-guides/kops/
 * http://woowabros.github.io/experience/2018/03/13/k8s-test.html

## sample
```
git clone https://github.com/nalbam/kubernetes.git
cd kubernetes

kubectl apply -f sample/sample-node.yml
kubectl apply -f sample/sample-spring.yml
kubectl apply -f sample/sample-web.yml

watch kubectl -n default get node,pod,svc
```

## ingress-nginx
```
ADDON=addons/.temp.yml
cp -rf addons/ingress-nginx.yml ${ADDON}

SSL_CERT_ARN=$(aws acm list-certificates | jq '.CertificateSummaryList[] | select(.DomainName=="nalbam.com")' | grep CertificateArn | cut -d '"' -f 4)

sed -i -e "s@{{SSL_CERT_ARN}}@${SSL_CERT_ARN}@g" "${ADDON}"

kubectl apply -f ${ADDON}

# ingress-nginx 에서 ELB Name 을 획득
ELB_NAME=$(kubectl get svc -n kube-ingress -owide | grep ingress-nginx | awk -F' ' '{print $4}' | cut -d '-' -f 1)

# ELB 에서 Hosted Zone ID, DNS Name 을 획득
ELB_ZONE_ID=$(aws elb describe-load-balancers --load-balancer-name ${ELB_NAME} | grep CanonicalHostedZoneNameID | cut -d '"' -f 4)
ELB_DNS_NAME=$(aws elb describe-load-balancers --load-balancer-name ${ELB_NAME} | grep '"DNSName"' | cut -d '"' -f 4)

# Route53 에서 해당 도메인의 Hosted Zone ID 를 획득
ZONE_ID=$(aws route53 list-hosted-zones | jq '.HostedZones[] | select(.Name=="nalbam.com.")' | grep '"Id"' | cut -d '"' -f 4 | cut -d '/' -f 3)

DOMAIN="sample-web.nalbam.com."

# temp file
RECORD=sample/.temp.json
cp -rf sample/record-sets.json ${RECORD}

# replace
sed -i -e "s@{{DOMAIN}}@${DOMAIN}@g" "${RECORD}"
sed -i -e "s@{{ELB_ZONE_ID}}@${ELB_ZONE_ID}@g" "${RECORD}"
sed -i -e "s@{{ELB_DNS_NAME}}@${ELB_DNS_NAME}@g" "${RECORD}"

# Route53 의 Record Set 에 입력/수정
aws route53 change-resource-record-sets --hosted-zone-id ${ZONE_ID} --change-batch file://./${RECORD}
```
 * https://github.com/kubernetes/ingress-nginx/
 * https://github.com/kubernetes/kops/tree/master/addons/ingress-nginx

## heapster
```
ADDON=addons/.temp.yml
cp -rf addons/heapster.yml ${ADDON}

kubectl apply -f ${ADDON}

watch kubectl -n kube-system top pod
```
 * https://github.com/kubernetes/heapster/
 * https://github.com/kubernetes/kops/blob/master/docs/addons.md

## dashboard
```
ADDON=addons/.temp.yml
cp -rf addons/dashboard.yml ${ADDON}

SSL_CERT_ARN=$(aws acm list-certificates | jq '.CertificateSummaryList[] | select(.DomainName=="nalbam.com")' | grep CertificateArn | cut -d '"' -f 4)

sed -i -e "s@{{SSL_CERT_ARN}}@${SSL_CERT_ARN}@g" "${ADDON}"

kubectl apply -f ${ADDON}

kubectl -n kube-system get secret | grep dashboard
kubectl -n kube-system describe secret kubernetes-dashboard-token-xxxxx

kubectl proxy

http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
```
 * https://github.com/kubernetes/dashboard/
 * https://github.com/kubernetes/kops/blob/master/docs/addons.md

## cluster-autoscaler
```
ADDON=addons/.temp.yml
cp -rf addons/cluster-autoscaler.yml ${ADDON}

CLOUD_PROVIDER=aws
IMAGE=k8s.gcr.io/cluster-autoscaler:v1.1.2
MIN_NODES=2
MAX_NODES=5
AWS_REGION=ap-northeast-2
GROUP_NAME="nodes.kube.nalbam.com"
SSL_CERT_PATH="/etc/ssl/certs/ca-certificates.crt"

sed -i -e "s@{{CLOUD_PROVIDER}}@${CLOUD_PROVIDER}@g" "${ADDON}"
sed -i -e "s@{{IMAGE}}@${IMAGE}@g" "${ADDON}"
sed -i -e "s@{{MIN_NODES}}@${MIN_NODES}@g" "${ADDON}"
sed -i -e "s@{{MAX_NODES}}@${MAX_NODES}@g" "${ADDON}"
sed -i -e "s@{{GROUP_NAME}}@${GROUP_NAME}@g" "${ADDON}"
sed -i -e "s@{{AWS_REGION}}@${AWS_REGION}@g" "${ADDON}"
sed -i -e "s@{{SSL_CERT_PATH}}@${SSL_CERT_PATH}@g" "${ADDON}"

kubectl apply -f ${ADDON}

```
 * https://github.com/kubernetes/autoscaler/
 * https://github.com/kubernetes/kops/tree/master/addons/cluster-autoscaler
