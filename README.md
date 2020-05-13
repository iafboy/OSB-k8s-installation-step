# Shows how to install OSB in k8s by using weblogic-kubernetes-operator
### Detail information please refer to https://oracle.github.io/weblogic-kubernetes-operator/samples/simple/domains/soa-domain/
### please make sure the k8s version is 1.15, the weblogic-kubernetes-operator v2.4.0 and v2.5.0 didn't support k8s 1.16+

# Steps
- Checkout sourcecode( for example clone to directory '/root/soa' )
```
git clone https://github.com/oracle/weblogic-kubernetes-operator.git
cd weblogic-kubernetes-operator
git checkout v2.4.0
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
  -t osb \
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
kubectl apply -f domain.yaml
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
