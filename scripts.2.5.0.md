# Note. the last version of operator have problem.please check out v2.4.0 version！

## drop rcu
cd /root/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-rcu-schema

./drop-rcu-schema.sh -s SOA1 \
  -d oracle-db.soans.svc.cluster.local:1521/devpdb.k8s \
  -t osb 
  

 
## stop db
cd /root/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-soa-domain/domain-home-on-pv/create-database
kubectl -n soans delete -f db-with-pv.yaml

## clean 
cd /root/soa/scripts/weblogic-domains/soainfra
kubectl apply -f delete-domain-job.yaml -n soans
kubectl -n soans delete -f domain.yaml
kubectl -n soans delete -f create-domain-job.yaml
kubectl -n soans delete -f delete-domain-job.yaml 
kubectl delete -f delete-domain-job.yaml 
kubectl delete configmap soainfra-create-fmw-infra-sample-domain-job-cm -n soans
rm -rf /root/soa/domainstorage/*
rm -rf /root/soa/scripts/weblogic-domains



## check status
kubectl -n soans get all
kubectl get all
kubectl -n soans get secret
kubectl get secret


## start db
#cd /root/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-oracle-db-service/
#./start-db-service.sh -n soans
cd /root/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-soa-domain/domain-home-on-pv/create-database
kubectl -n soans create -f db-with-pv.yaml



cd /root/weblogic-kubernetes-operator/kubernetes
### modify weblogic-kubernetes-operator/kubernetes/samples/scripts/create-rcu-schema/common/rcu.yaml
###  change imagePullPolicy: Always--->imagePullPolicy: IfNotPresent 
### change namespace in common/rcu.yaml to soans


cd /root/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-rcu-schema

./create-rcu-schema.sh \
  -s SOA1 \
  -t osb \
  -d oracle-db.soans.svc.cluster.local:1521/devpdb.k8s \
  -i container-registry.oracle.com/middleware/soasuite:12.2.1.3

### Modify the domain.input.yaml to use [oracle-db.soans.svc.cluster.local:1521/devpdb.k8s] as rcuDatabaseURL and [SOA1] as rcuSchemaPrefix 
### Modify the domain.input.yaml to use [oracle-db.default.svc.cluster.local:1521/devpdb.k8s] as rcuDatabaseURL and [SOA1] as rcuSchemaPrefix 


kubectl -n soans delete secret soainfra-rcu-credentials
cd /root/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-rcu-credentials
./create-rcu-credentials.sh \
  -u SOA1_SOAINFRA \
  -p Oradoc_db1 \
  -a sys \
  -q Oradoc_db1 \
  -d soainfra \
  -n soans \
  -s soainfra-rcu-credentials

  
./create-pv-pvc.sh \
  -i create-pv-pvc-inputs.yaml \
  -o /root/soa/domainstorage

kubectl -n soans delete secret soainfra-domain-credentials
cd /root/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-weblogic-domain-credentials
./create-weblogic-credentials.sh -u weblogic -p Welcome1 -n soans -d soainfra -s soainfra-domain-credentials

cd /root/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-soa-domain/domain-home-on-pv

./create-domain.sh \
  -i create-domain-inputs.yaml \
  -o /root/soa/scripts
  
cd /root/soa/scripts/weblogic-domains/soainfra
kubectl apply -f domain.yaml -n soans
  
  
  
