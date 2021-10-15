

## CKS study files

### Namespaces

```bash
# Running a simple docker container
docker container run --name c1 -d ubuntu sh -c 'sleep 1d'
docker container exec c1 ps auxfw
ps auxfe
# Running another docker container to check isolagion of processess
docker container run --name c2 -d ubuntu sh -c 'sleep 100d'
ps auxfe
# Remove c2 and recreate using the same PID namespace from c1
docker container rm c2 --force
docker container run --name c2 --pid=container:c1 -d ubuntu sh -c 'sleep 100d'
docker container exec c1 ps auxfw
docker container exec c2 ps auxfw
# For the kernel, the PIDs will allways be different
ps auxfe
# Remove the containers
docker container rm c1 --force
docker container rm c2 --force
docker container ps
```



### GUI elements

- `kubectl proxy` creates a proxy server between the local machine and the kubernetes API endpoint, using the credentials from the `kubeconfig` file.
- `kubectl port-forward` is also used to forward traffic from the local machine to the kubernetes cluster but in a more generic way. It can for example forward any TCP traffic (not only HTTP/HTTPS).
- Install the dashboard:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
```

- Allow **insecure** external access to the dashboard. Note that this shouldn't ever be done in a production environment as it's going to expose the cluster **without restrictions**.
- - Edit the deployment and change some lines below the `spec` section as follows:

```bash
kubectl -n kubernetes-dashboard edit deployment kubernetes-dashboard 
```

- - Change some lines below the `spec, containers` section as follows:

```yaml
(...)
      - args:
        - --namespace=kubernetes-dashboard
        - --insecure-port=9090
        image: kubernetesui/dashboard:v2.3.1
        imagePullPolicy: Always
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /
            port: 9090
            scheme: HTTP
(...)
```

- - Edit the service to change it from `ClusterIP` to `NodePort`, so it's accessible from the outside world.

```bash
kubectl get svc -n kubernetes-dashboard
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
dashboard-metrics-scraper   ClusterIP   10.98.101.226   <none>        8000/TCP   22m
kubernetes-dashboard        ClusterIP   10.99.171.243   <none>        443/TCP    22m
kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard
```

- - Change the `spec` session as follows:

```yaml
(...)
  ports:
  - port: 9090
    protocol: TCP
    targetPort: 9090
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: NodePort
(...)
```

- - Check again and the service should be now `NodePort`:

```bash
kubectl get svc -n kubernetes-dashboard
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
dashboard-metrics-scraper   ClusterIP   10.98.101.226   <none>        8000/TCP         29m
kubernetes-dashboard        NodePort    10.99.171.243   <none>        9090:32325/TCP   29m
```

- - To access it, grab the IP of the node where the pod is running and from the browser access using this IP and the NodePort. For example: http://192.168.1.161:32325
- The dashboard won't show much as the default service account (`kubernetes-dashboard:kubernetes-dashboard`) doesn't have permissions. To allow it, we can create a rolebinding (or cluster role binding, to allow access on all namespaces) between the service account and a default `view` cluster role, which allows read access to resources.

```bash
kubectl -n kubernetes-dashboard get sa
NAME                   SECRETS   AGE
default                1         2d21h
kubernetes-dashboard   1         2d21h
```

```bash
kubectl get clusterrole view
NAME   CREATED AT
view   2021-09-18T12:54:11Z
```

- - Role binding:

```bash
kubectl -n kubernetes-dashboard create rolebinding insecure --serviceaccount kubernetes-dashboard:kubernetes-dashboard --clusterrole view
```

- - Cluster role binding:

```bash
kubectl -n kubernetes-dashboard create clusterrolebinding insecure --serviceaccount kubernetes-dashboard:kubernetes-dashboard --clusterrole view
```

### Instance metadata (particular to cloud providers)

On cloud providers the instances may expose information via the internal metadata API service (normally http://169.254.169.254). To restrict access to such service, it's possible to leverage the network policies.

### CIS benchmark

List of security best practices for systems, which were addapeted to kubernetes by the `kube-bench` project. It contains tests that can be executed from different ways to check if the cluster is compliant with the guidelines.

- Running it from a docker container, which can be from a master or a worker node.

```bash
sudo docker run --pid=host -v /etc:/etc:ro -v /var:/var:ro -t aquasec/kube-bench:latest --version 1.21
```

### Kubernetes and Operational system binaries

The OS where kubernetes is running will have other binaries and services, which can be used to gain access to the system. This means that it's important to keep track of any unwanted change on any file. This includes not only the binaries, but libraries and configuration files as well.

Kubernetes shares the sha512 hash for the files available to download, and this can be used as source of truth to check if the files are sane.

### Role and rolebinding

Relative to namespaces.

- Example creating two namespaces and giving different permissions to the same user in each of the namespaces.
  - User: flanders
    - Namespace `finance`: get pods
    - Namespace `development`: get and list pods

```bash
kubectl create ns finance
kubectl create role finance --verb=get --resource=pods
kubectl create rolebinding finance --role=finance --user=flanders
```

```bash
kubectl create ns development
kubectl create role development --verb=get --verb=list --resource=pods
kubectl create rolebinding development --role=development --user=flanders
```

Tests:

```bash
kubectl -n finance auth can-i get pods --as flanders
yes
kubectl -n development auth can-i get secrets --as flanders
no
kubectl -n development auth can-i get pods --as flanders
yes
```



### ClusterRole and ClusterRoleBinding

Relative to the whole cluster.

- Example: Create a ClusterRole allowing:
  - User Jim: can delete deployments in all namespaces
  - User Todd can delete deploymentes only in namespace `development`

```bash
kubectl create clusterole deploy-control --verb=delete --resource=deployment
kubectl create clusterrolebinding deploy-control --clusterrole=deploy-control --user=jim
```

```bash
kubectl -n development create rolebinding --clusterrole=deploy-control --user=todd
```

Tests:

```bash
kubectl auth can-i delete deployment --as jim
yes
kubectl -n finance auth can-i delete deployment --as jim
yes
kubectl -n kube-system auth can-i delete deployment --as jim
yes
kubectl -n development auth can-i delete deploy --as todd
yes
kubectl -n development auth can-i create deploy --as todd
no
kubectl -n finance auth can-i delete deploy --as todd
no
```



### User

There's no `user` resource in kubernetes but it's possible to give access to users via certificates and keys. To do so, the certificate needs to be signed by the cluster's CA and the user name must be present as a CN (CN=myuser, for example).

Once emitted, a certificate cannot be invalidated and teh user name can't also be used until the cert is expired.  The alternatives are:

- Remove all access using RBAC;
- Create a new CA and re-issue all certs.

To create a certificate, the steps are:

- Create a key;

```bash
openssl genrsa -out lobo.key 2048
```

- Create the CSR;

```bash
openssl req -new -key lobo.key -out lobo.csr -subj='/CN=lobo'
```

- Create the  `CertificateSigningRequest` resource. This involves some steps which are getting a template from the docs, converting the certificate contents to base64 to add it to the `request` field and create the resource.

```bash
cat lobo.csr | base64 -w 0
```

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: lobo
spec:
  request: <insert here the output>
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
```

```bash
kubectl create -f https://raw.githubusercontent.com/mstelles/cks/main/rbac/csr.yaml
```

- The CSR will be on pending state.

```bash
kubectl get csr
NAME   AGE   SIGNERNAME                            REQUESTOR          CONDITION
lobo   21s   kubernetes.io/kube-apiserver-client   kubernetes-admin   Pending
```

- Approve the certificate. Once this is executed, the certificate will be present on the user cert.

```bash
kubectl certificate approve lobo
certificatesigningrequest.certificates.k8s.io/lobo approved
kubectl get csr
NAME   AGE     SIGNERNAME                            REQUESTOR          CONDITION
lobo   4m14s   kubernetes.io/kube-apiserver-client   kubernetes-admin   Approved,Issued
```

- Get it from the CSR and decode.

```bash
kubectl get csr lobo -o yaml | grep certificate: | awk '{print$2}' | base64 -d > lobo.crt
```

- Add the information to the kubeconfig file.

```bash
kubectl config set-credentials lobo --client-key=./lobo.key --client-certificate=lobo.crt
# or adding the contents of the key and cert to kubeconfig
kubectl config set-credentials lobo --client-ley=./lobo.key --client-certificate=lobo.crt --embed-certs
```

- Connect the user to the cluster

```bash
kubectl config set-context lobo --user=lobo --cluster=kubernetes
```

- Checking

```bash
kubectl config view
kubectl config get-users
kubectl config get-contexts
```

- Testing

```bash
kubectl config use-context lobo
# User has no permissions, as none were given to it
kubectl get pods
Error from server (Forbidden): pods is forbidden: User "lobo" cannot list resource "pods" in API group "" in the namespace "default"
kubectl run pod1 --image=ubuntu -- sleep 3600
Error from server (Forbidden): pods is forbidden: User "lobo" cannot create resource "pods" in API group "" in the namespace "default"
```

### Service Account

Used to give permissions to resources. All pods that are deployed will use a ServiceAccount and if none is specified, the default SA is chosen.

```bash
k exec -ti multi -- mount | grep serviceaccount
tmpfs on /run/secrets/kubernetes.io/serviceaccount type tmpfs (ro,relatime)
```

With the ServiceAccount comes a `token` that can be used to access the cluster API. in a default configuration this `token` won't give many permissions but in most of the cases pods won't need to have such kind of access and disabling it might be needed.

The below example shows the `multi` pod using the token from the `supersa` service account to communicate with the cluster's API.

```bash
root@multi:/# curl -k https://kubernetes -H "Authorization: Bearer $(cat /run/secrets/kubernetes.io/serviceaccount/token)"
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "forbidden: User \"system:serviceaccount:default:supersa\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {

  },
  "code": 403
}
```

It's possible to disable the mount point for the pod by changing the pod or service account. Below an example changing the service account.

```yaml
kubectl edit sa supersa
(...)
automountServiceAccountToken: false
metadata:
(...)
```

Redeploy the pod and check that it won't have the service account mounted as a volume.

```bash
kubectl -f multi.yaml replace --force
kubectl exec -ti multi -- mount | grep serviceaccount
```

When creating a service account it by default won't have any permissions. The same goes for the de fault service account. But it's possible to give permissions by binding it to a `cluster role` or `role`.

Below an example where a new role is created allowing to get pods in the default namespace.

```bash
kubectl create role superrole --verb=get --resource=pods
kubectl create rolebinding superrole --role=superrole --serviceaccount=default:supersa
kubectl auth can-i get pods --as system:serviceaccount:default:supersa
yes
kubectl auth can-i list pods --as system:serviceaccount:default:supersa
no
```

### Anonymous access to the API service

By default the access to the API service is allowed to the anonymous user, and this happens because internally the cluster needs this kind of access as some health probes keep on reaching the API service. However, it's possible to disable it if needed, in a specific situation.

- Test before disabling

```bash
curl -k https://192.168.1.160:6443
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {

  },
  "code": 403
}
```

- Edit the cluster API service manifest, adding the `--anonymous-auth=false` line to it, under the `command` section
- Test again and the access will be unauthorised.

```bash
curl -k https://192.168.1.160:6443
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}
```

It's important to re-enable the access so the cluster internals work properly.

### Creating manual requests to the API server

It is possible to get the credentials being used and stored in the `kubeconfig` file and use them to create the calls to the API server.

- Get the CA:

```bash
kubectl config view --raw | grep certificate-authority-data: | awk '{print$2}' | base64 -d > ca
```

- Get the certificate:

```bash
kubectl config view --raw | grep client-certificate-data: | awk '{print$2}' | base64 -d  > crt
```

- Get the client key:

```bash
kubectl config view --raw | grep client-key-data: | awk '{print$2}' | base64 -d  > key
```

- Using the three generated files with the `curl` command to make the API call and get a list of the cluster's resources, for example:

```bash
curl https://192.168.1.160:6443 --cacert ca --cert crt --key key
{
  "paths": [
    "/.well-known/openid-configuration",
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/admissionregistration.k8s.io",
(...)
```

### Admission Controllers

Piece of code that runs inside the `kube-apiserver` and processes requests after authentication and authorization stages. To change which admission controllers are in use by the cluster, change the `--enable-admission-plugins` option in the `kube-apiserver` manifest. There is an extense list of admission controllers and on this example, the focus is the `NodeRestriction` controller as it is normally enabled by default and widely used to restrict pods and nodes to change labels, add/remove taints and so on.

### Cluster upgrade

Order to upgrade the parts of the cluster:

1. Master node components (apiserver, controller-manager, scheduler);

   1. apiserver
   2. scheduler
   3. controller manager

2. Worker node components (kubelet, kube-proxy). They should be on the same minor version as the apiserver or at least one below but it can be up to two minor versions below.

   1. kubelet and kube-proxy should follow the master node components;
   2. kubectl can be up to two versions different (up or below).
   3. To safely upgrade the worker node, it is necessary to drain the node, upgrade and uncordon it.

   ```bash
   kubectl drain k8sworker01 --ignore-daemonsets
   node/k8sworker01 already cordoned
   WARNING: ignoring DaemonSet-managed Pods: kube-system/kube-proxy-qszhc, kube-system/weave-net-dkbx8
   evicting pod kube-system/coredns-558bd4d5db-xvhbl
   evicting pod kube-system/coredns-558bd4d5db-bghs5
   pod/coredns-558bd4d5db-xvhbl evicted
   pod/coredns-558bd4d5db-bghs5 evicted
   node/k8sworker01 evicted
   # upgrade
   kubectl uncordon k8sworker01
   node/k8sworker01 uncordoned
   ```

### ConfigMaps and Secrets

A way to share information with pods in a decoupled way. For example, the applications may need credentials or other data and instead adding it directly to the code. it's possible to share it using `secrets` or `configMaps`. The basic difference between these two is that ConfigMaps will hold information that doesn't need to be encrypted while `secrets` will be used when handling sensitive information.

Creatiing two simple secrets and using them on a pod. 

```bash
kubectl create secret generic secret1 --from-literal=key1="value for key1, secret1"
kubectl create secret generic secret1 --from-literal=key2="value for key2, secret2"
```

The below manifest is an example to use the first as a mount point and the second as environment variable.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: multi
  name: multi
spec:
  containers:
  - args:
    - sleep
    - "3600"
    image: mstelles/multi
    env:
      - name: SECRET2_VAR
        valueFrom:
          secretKeyRef:
            name: secret2
            key: key2
    name: multi
    volumeMounts:
    - name: secret1
      mountPath: "/etc/secret1"
  volumes:
  - name: secret1
    secret:
      secretName: secret1
```

Check:

```bash
kubectl exec -ti multi -- bash -c "cat /etc/secret1/key1"
value for key1, secret1
kubectl exec -ti multi -- bash -c "env | grep SECRET2_VAR"
SECRET2_VAR=value for key2, secret2
```

Since this information is available to the container, it can also be accessed via the filesystem from the node where the container is running.

- Via `/proc` filesystem:

```bash
docker container ps # get the ID of the container where the secret was mounted
docker container inspect <id> | jq .[0] | jq '.State.Pid' # get the pid of the container
cat /proc/<pid>/root/etc/secret1/key1
value for key1, secret1
```

- Via the container filesystem in the OS:

```bash
docker container ps # get the ID of the container where the secret was mounted
secret=$(docker container inspect 2c8edf1334843  | grep Source.*secret1 | cut -d\" -f4 #secret1 as secret name)
cat $secret/key1 #key1 as the key name for the secret1 secret
value for key1, secret1
```

For environment variables, just check the output of the `docker container inspect` command, as it's already exposed there.

```bash
# The last number is for the `Env` list and may be different
docker container inspect 2c8edf1334843  | jq .[0] | jq '.Config.Env' | jq .[0] 
"SECRET2_VAR=value for key2, secret2"
```

Another last option would be to check it in `etcd`, which should be running as a pod in the master node. To make this happen, use the `etcdctl` command, always informing the API version and the certficate/key files.

Example:

```bash
ETCDCTL_API=3 etcdctl endpoint health --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/apiserver-etcd-client.crt --key /etc/kubernetes/pki/apiserver-etcd-client.key
127.0.0.1:2379 is healthy: successfully committed proposal: took = 679.12µs
```

Hint: to get the certs, grep `etcd` in the manifest file for the service.

```bash
grep etcd /etc/kubernetes/manifests/kube-apiserver.yaml
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
```

Now getting the secret via `etcd`, The output below shows on the last line the key, the value and the type of the secret.

```bash
ETCDCTL_API=3 etcdctl --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/apiserver-etcd-client.crt --key /etc/kubernetes/pki/apiserver-etcd-client.key get /registry/secrets/default/secret2
/registry/secrets/default/secret2
k8s


v1Secret�
�
secret2default"*$9a3ed911-a813-42de-8c7e-1caf746769de2����z�_
kubectl-createUpdatev����FieldsV1:-
+{"f:data":{".":{},"f:key2":{}},"f:type":{}}
key2value for key2, secret2Opaque"
```

### Enabling encryption on `etcd`

It is possible to encrypt secrets that are stored in the cluster. Since this service is running as a pod in the master node, necessary to follow the below steps:

1. Create an `EncryptionConfiguration` resource file in the k8s master node. This file can be anywhere but `/etc/kubernetes/etcd/ec.yaml` is a good starting point.

   The below example uses two providers and it is important to understand that they are processed in order. Meaning, with this configuration, the secrets that will be created from this point onwards will be encrypted with `aescbc` algorithm but plain text secrets will be readable (`identity: {}` provider). Removing the last line will cause the plain text secrets to be unreadable.

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          # echo -n abcdefhg12345678 | base64
          secret: YWJjZGVmZ2gxMjM0NTY3OA==
    - identity: {}
```

OBS.: The secret has to be 16, 24 or 32 byts in size. To generate a random secret, try:

```bash
head -c 32 /dev/urandom | base64
```

To use an already know key as secret, try:

```bash
echo -n <your 16, 24 or 32 bytes key> | base64
```

2. Add this file with the `--encryption-provider-config` parameter to the pod manifest. This can be done either editing the pod or it's manifest (`/etc/kubernetes/manifests/kube-apiserver.yaml`). On this same file, mount the `/etc/kubernetes/etcd/` directory in the pod. Full example [here](https://raw.githubusercontent.com/mstelles/cks/main/kube-apiserver.yaml).

```yaml
(...)
spec:
  containers:
  - command:
    - kube-apiserver
    - --encryption-provider-config=/etc/kubernetes/etcd/ec.yaml
(...)
    volumeMounts:
    - mountPath: /etc/kubernetes/pki
      name: k8s-certs
      readOnly: true
(...)
   - mountPath: /etc/kubernetes/etcd
      name: etcd
      readOnly: true
(...)
```

From this moment onwards, all new secrets will be encrypted and it won't be possible to read them via the etcd. Via the api server still possible.

For example, creating `secret3` to test.

```bash
kubectl create secret generic secret3 --from-literal=key3=topsecret
# getting it via API
kubectl get secret secret3 -o yaml  | grep ^\ \ key3: | awk '{print$2}' | base64 -d; echo
topsecret
# getting via etcd
ETCDCTL_API=3 etcdctl --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/apiserver-etcd-client.crt --key /etc/kubernetes/pki/apiserver-etcd-client.key get /registry/secrets/default/secret3
/registry/secrets/default/secret3
��g��D�^ϯl��-���޶�TP�ќ^m��N�]�ML�n�[4TJ����Y�Ȍ>W邪S�4�Ĥ*���`tZ�$��H�c*g�S��l�:�5�@Th��U�l�\����5���4�EeP�\>�{��DfH~�Ii�>΢�w�˝}�FZ��(ܳW�2K���+泻�cA,�B�}��#G��9�>
```

If the final outcome is to encrypt all the secrets that are already in place, first run the below command and then remove the `identity` line from the `EncryptionConfiguration` manifest.

```bash
kubectl get secret --all-namespaces -o yaml | kubectl replace -f -
```

### Sandbox

When related to containers, sandbox is an additional layer between the container application and the OS kernel syscalls. And due to it's natural architrecture, it will inflict more load on the OS and might not be applicable to all situations. Below some summary topics when using sandbox containers:

- Resource usage from OS perspective will increase;
- Not optimal for large or containers with heavy syscall workloads;
- Won't allow direct access to hardware.

The `kubelet` daemon uses CRI (container runtime interface) to choose which container runtime is going to be used and in a cluster, only one runtime is allowed. Meaning, it's not possible to use for example `docker` and one sandbox engine at the same time.

Two solutions for sandbox are:

- gVisor: it implements a "fake" kernel on top of the OS kernel which acts as a proxy. It doesn't supports all system calls though.
- Katacontainers: It wraps the container in a virtual machine, creating a more strict isolation layer but also creating some restrictions for it's usage.

Pratical example using gvisor:

1. Install gVisor

```bash
ARCH=$(uname -m)
URL=https://storage.googleapis.com/gvisor/releases/release/20210806/${ARCH}
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common
mkdir ./tmp_install
cd ./tmp_install
wget ${URL}/runsc ${URL}/runsc.sha512 \
  ${URL}/containerd-shim-runsc-v1 ${URL}/containerd-shim-runsc-v1.sha512
sha512sum -c runsc.sha512 -c containerd-shim-runsc-v1.sha512
chmod a+rx runsc containerd-shim-runsc-v1
sudo mv runsc containerd-shim-runsc-v1 /usr/local/bin
```

Create the `/etc/containerd/config.toml` file (contents [here](https://raw.githubusercontent.com/mstelles/cks/main/config.toml))

Restart `kubelet` and  `containerd` services.

```bash
systemctl restart containerd
systemctl restart kubelet
```

OBS.: If even following these steps the service pod isn't running and with the `Failed to create pod sandbox: rpc error: code = Unknown desc = RuntimeHandler "runsc" not supported` error message, check the FAQ below.

Ref:

- Install: https://gvisor.dev/docs/user_guide/install/
- Configure: https://gvisor.dev/docs/user_guide/containerd/quick_start/
- FAQ: https://gvisor.dev/docs/user_guide/faq/#runtime-handler

2. Create the `RuntimeClass` k8s object

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
```

3. Run a pod using the created `RuntimeClass`

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: multi
  name: multi
spec:
  runtimeClassName: gvisor
  containers:
  - image: mstelles/multi
    name: multi
```

4. Check the differences. If everything is working fine, executing the `uname` command from whithin the pod should show a different output when comparing to the OS.

```bash
# from the pod
kubectl exec -ti multi -- bash -c "uname -a"
Linux multi 4.4.0 #1 SMP Sun Jan 10 15:06:54 PST 2016 x86_64 GNU/Linux
# from the OS
uname -a
Linux centos8 4.18.0-338.el8.x86_64 #1 SMP Fri Aug 27 17:32:14 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
```

