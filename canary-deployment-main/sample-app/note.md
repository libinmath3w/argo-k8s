https://awswithatiq.com/how-to-set-up-prometheus-and-grafana-in-eks/


oidc_id=$(aws eks describe-cluster --name prometheusk8 --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
eksctl utils associate-iam-oidc-provider --cluster prometheusk8 --approve


eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster prometheusk8 \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-only \
  --role-name AmazonEKS_EBS_CSI_DriverRole


eksctl create addon --name aws-ebs-csi-driver --cluster prometheusk8 --service-account-role-arn arn:aws:iam::918252278260:role/AmazonEKS_EBS_CSI_DriverRole --force

# add prometheus Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# add grafana Helm repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update


helm install prometheus prometheus-community/prometheus \
    --namespace monitoring \
    --set alertmanager.persistentVolume.storageClass="gp2" \
    --set server.persistentVolume.storageClass="gp2"



# note for future use

prometheus-server.monitoring.svc.cluster.local

export POD_NAME=$(kubectl get pods --namespace monitoring -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace monitoring port-forward $POD_NAME 9090


kubectl get pods -n monitoring

kubectl get all -n monitoring


# loadbalancer

kubectl patch svc prometheus-server -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'

http://a8efc2cd34bc841af917182ebb10b659-372634069.ap-south-1.elb.amazonaws.com/

# k8s metrics server

https://jhooq.com/prometheous-k8s-aws-setup/

kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

kubectl get deployment metrics-server -n kube-system


# grafana

mkdir -p ${HOME}/environment/grafana

cat << EoF > ${HOME}/environment/grafana/grafana.yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server.grafana.svc.cluster.local
      access: proxy
      isDefault: true
EoF

kubectl create namespace grafana

helm install grafana grafana/grafana \
    --namespace grafana \
    --set persistence.storageClassName="gp2" \
    --set persistence.enabled=true \
    --set adminPassword='Med@12345' \
    --values ${HOME}/environment/grafana/grafana.yaml \
    --set service.type=LoadBalancer


grafana.grafana.svc.cluster.local


kubectl get pods -n grafana

kubectl get all -n grafana


a111757aa59474bdc837957f56633e48-1091271305.ap-south-1.elb.amazonaws.com

admin:Med@12345


# argocd
-------------------
abcca0743faae454988e4de0648f05ff-1778306526.ap-south-1.elb.amazonaws.com

PLoavdFy2kWDGJJK


# argo Rollouts
------------------------------

kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml



curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x ./kubectl-argo-rollouts-linux-amd64
sudo mv ./kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts


# nginx ingress

helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace



kubectl create ns monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl apply -f storageclass.yaml -n monitoring
helm install prometheus prometheus-community/prometheus -n monitoring --set server.persistentVolume.existingClaim="efs-sca"
export POD_NAME=$(kubectl get pods --namespace monitoring -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace monitoring port-forward $POD_NAME 9090
kubectl get pods -n monitoring
kubectl get pvc -n monitoring



helm install -n monitoring -f values.yaml prometheus .




issues
------------------

prometheus server not installing.

create this storage class
https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/examples/kubernetes/dynamic_provisioning/specs/storageclass.yaml

https://stackoverflow.com/questions/75781947/not-able-to-attach-the-persistent-volume-to-the-prometheus-pod-installed-using-h

solution : https://stackoverflow.com/questions/47235014/why-prometheus-pod-pending-after-setup-it-by-helm-in-kubernetes-cluster-on-ranch


# projects

https://github.com/jhandguy/canary-deployment

https://code.egym.de/canary-deployment-in-kubernetes-part-3-smart-canary-deployment-using-argo-rollouts-and-47992d72222c