# Shows how to install OSB in k8s by using weblogic-kubernetes-operator
### Detail information please refer to https://oracle.github.io/weblogic-kubernetes-operator/samples/simple/domains/soa-domain/
### please make sure the kubernetes version is 1.15, the weblogic-kubernetes-operator v2.4.0 and v2.5.0 didn't support kubernetes 1.16+. and also please make sure helm installed
### all the demo related configuration file, for example 'create-domain-inputs.yaml','soainfra-domain-pv.yaml' and 'soainfra-domain-pv' are in this repository

# Steps
- Checkout sourcecode( for example clone to directory '/root/soa' )
```
git clone https://github.com/oracle/weblogic-kubernetes-operator.git
cd weblogic-kubernetes-operator
git checkout v2.4.0
```
- get operator
```
docker pull oracle/weblogic-kubernetes-operator:2.5.0
```
- get Traefik
```
docker pull traefik:1.7.12
```
- get SOA suite image
```
docker pull container-registry.oracle.com/middleware/soasuite:12.2.1.3
```
- Grant the Helm service account the cluster-admin role
```
  get imagecat <<EOF | kubectl apply -f -
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: helm-user-cluster-admin-role
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: cluster-admin
  subjects:
  - kind: ServiceAccount
    name: default
    namespace: kube-system
  EOF
```
- Use Helm to install the operator and Traefik load balancer
```
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
```
- Create a Traefik (ingress-based) load balancer
```
kubectl create namespace traefik

 helm install traefik-operator stable/traefik \
     --namespace traefik \
     --values kubernetes/samples/charts/traefik/values.yaml \
     --set "kubernetes.namespaces={traefik}" \
     --wait
```
- install operator
```
kubectl create namespace sample-weblogic-operator-ns

kubectl create serviceaccount -n sample-weblogic-operator-ns sample-weblogic-operator-sa

helm install sample-weblogic-operator kubernetes/charts/weblogic-operator \
  --namespace sample-weblogic-operator-ns \
  --set image=oracle/weblogic-kubernetes-operator:2.5.0 \
  --set serviceAccount=sample-weblogic-operator-sa \
  --set "domainNamespaces={}" \
  --wait
```
- create namespace
```
kubectl create namespace soans
```
- Use helm to configure the operator to manage domains in this namespace soans
```
helm upgrade sample-weblogic-operator  kubernetes/charts/weblogic-operator \
    --namespace soans \
    --reuse-values \
    --set "domainNamespaces={soans}" \
    --wait
```
- Configure Traefik to manage ingresses created in this namespace
```
helm upgrade traefik-operator stable/traefik \
    --namespace traefik \
    --reuse-values \
    --set "kubernetes.namespaces={traefik,soans}" \
    --wait
```    
- create pv and pvc for domain
```
kubectl -n soans -f soainfra-domain-pv.yaml
kubectl -n soans -f soainfra-domain-pvc.yaml
```
- create db
```
cd /root/soa/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-rcu-schema
./start-db-service.sh
```
- create rcu
```
cd /root/soa/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-rcu-schema
./create-rcu-schema.sh \
  -s SOA1 \
  -t soaessosb \
  -d oracle-db.default.svc.cluster.local:1521/devpdb.k8s \
  -i container-registry.oracle.com/middleware/soasuite:12.2.1.3
```
- create rcu credentials
```
cd /root/soa/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-rcu-credentials
./create-rcu-credentials.sh \
  -u SOA1_SOAINFRA \
  -p Oradoc_db1 \
  -a sys \
  -q Oradoc_db1 \
  -d soainfra \
  -n soans \
  -s soainfra-rcu-credentials
```
- create weblogic domain credentials
```
cd /root/soa/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-weblogic-domain-credentials
./create-weblogic-credentials.sh -u weblogic -p Welcome1 -n soans -d soainfra -s soainfra-domain-credentials
```
- create domain
```
cd /root/soa/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-soa-domain/domain-home-on-pv
./create-domain.sh \
  -i create-domain-inputs.yaml \
  -o /root/soa/scripts
```  
- init domain
```
cd /root/soa/scripts/weblogic-domains/soainfra
kubectl apply -f domain.yaml -n soans
```

# Clean all
- clean domain
```
cd /root/soa/scripts/weblogic-domains/soainfra  
kubectl apply -f delete-domain-job.yaml
kubectl  delete -f domain.yaml
kubectl delete -f create-domain-job.yaml
kubectl delete -f delete-domain-job.yaml 
kubectl delete configmap soainfra-create-fmw-infra-sample-domain-job-cm -n soans
rm -rf /root/soa/domainstorage/*
rm -rf /root/soa/scripts/weblogic-domains
```
- clean rcu
```
cd /root/soa/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-rcu-schema
./drop-rcu-schema.sh \
  -s SOA1 \
  -d oracle-db.default.svc.cluster.local:1521/devpdb.k8s \
  -t osb 
```
- clean db
```
cd /root/soa/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-rcu-schema
./stop-db-service.sh
```
- delete credentials
```
kubectl -n soans delete secret soainfra-rcu-credentials
kubectl -n soans delete secret soainfra-domain-credentials
```
