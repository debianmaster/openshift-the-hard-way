#  CAUTION ------WORK IN PROGRES------     

# openshift-the-hard-way

## On all nodes/master instances
```sh
yum install -y centos-release-openshift-origin
yum install -y origin-clients
yum install -y origin
yum install -y docker
#edit /etc/sysconfig/docker file and add --insecure-registry 172.30.0.0/16 to the OPTIONS parameter.
sed -i '/OPTIONS=.*/c\OPTIONS="--selinux-enabled --insecure-registry 172.30.0.0/16"' \
/etc/sysconfig/docker
systemctl is-active docker
systemctl enable docker
systemctl restart docker
```

## On Master let's say 10.128.0.2

```sh
mkdir -p ocp/master

openshift start master --dns='tcp://0.0.0.0:8053' --public-master='https://10.128.0.2:8443' \
--listen='https://0.0.0.0:8443' \
--master='https://10.128.0.2:8443' \
--write-config='ocp/master'
openshift start master --config=ocp/master/master-config.yaml
export KUBECONFIG=$(pwd)/ocp/master/admin.kubeconfig
oc get nodes
scp -r ocp node1:/tmp
```

## On nodes  let's say 10.128.0.3
```sh
oc adm create-node-config \
    --master='https://10.128.0.2:8443' \
    --node-dir=ocp/node1 \
    --node=node1 \
    --hostnames=node1,10.128.0.3 \
    --certificate-authority="ocp/master/ca.crt" \
    --signer-cert="ocp/master/ca.crt" \
    --signer-key="ocp/master/ca.key" \
    --signer-serial="ocp/master/ca.serial.txt" \
    --node-client-certificate-authority="ocp/master/ca.crt"

openshift start node --config=node-config.yaml
export KUBECONFIG=$(pwd)/ocp/master/admin.kubeconfig
oc get nodes 
oc adm policy add-scc-to-user hostnetwork -z router
oc adm router
oc new-app debianmaster/go-welcome
oc expose svc go-welcome --hostname=go-welcome.tmp.xfc.io
```




# Gcloud
```sh
gcloud compute instances list
gcloud config set compute/region asia-east1
gcloud config set compute/zone asia-east1-a
gcloud compute networks create openshift --mode custom
gcloud compute networks subnets create openshift-subnet \
  --network openshift \
  --range 10.240.0.0/24

gcloud compute firewall-rules create allow-internal \
  --allow tcp,udp,icmp \
  --network openshift \
  --source-ranges 10.240.0.0/24,10.200.0.0/16
 
gcloud compute firewall-rules create allow-external \
  --allow tcp:22,tcp:3389,tcp:6443,tcp:8443,icmp \
  --network openshift \
  --source-ranges 0.0.0.0/0  
  
gcloud compute firewall-rules create allow-healthz \
  --allow tcp:8080 \
  --network openshift \
  --source-ranges 130.211.0.0/22 

gcloud compute firewall-rules list --filter "network=openshift"
gcloud compute addresses create openshift --region=asia-east1
gcloud compute addresses list openshift
``` 
