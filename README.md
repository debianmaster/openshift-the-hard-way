# openshift-the-hard-way
```sh
yum install centos-release-openshift-origin
yum install origin-clients
yum install origin
yum install docker
#edit /etc/sysconfig/docker file and add --insecure-registry 172.30.0.0/16 to the OPTIONS parameter.
sed -i '/OPTIONS=.*/c\OPTIONS="--selinux-enabled --insecure-registry 172.30.0.0/16"' \
/etc/sysconfig/docker
systemctl is-active docker
systemctl enable docker
systemctl start docker
openshift start
export KUBECONFIG="$(pwd)"/openshift.local.config/master/admin.kubeconfig
export CURL_CA_BUNDLE="$(pwd)"/openshift.local.config/master/ca.crt
chmod +r "$(pwd)"/openshift.local.config/master/admin.kubeconfig
oc get nodes
oc adm create-node-config \
    --node-dir=openshift.local.config/node-test \
    --node=test \
    --hostnames=test,10.128.0.2 \
    --certificate-authority="openshift.local.config/master/ca.crt" \
    --signer-cert="openshift.local.config/master/ca.crt" \
    --signer-key="openshift.local.config/master/ca.key" \
    --signer-serial="openshift.local.config/master/ca.serial.txt" \
    --node-client-certificate-authority="openshift.local.config/master/ca.crt"
```
