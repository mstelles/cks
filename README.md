

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



### CIS benchmark

List of security best practices for systems, which were addapeted to kubernetes by the `kube-bench` project. It contains tests that can be executed from different ways to check if the cluster is compliant with the guidelines.

- Running it from a docker container, which can be from a master or a worker node.

```bash
sudo docker run --pid=host -v /etc:/etc:ro -v /var:/var:ro -t aquasec/kube-bench:latest --version 1.21
```



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
kubectl exec -ti multi -- mount | grep serviceaccount
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



### mTLS / Pod to Pod communication

A mechanism used to mutually authenticate two ends in order to create a secure communication channel between them. By default the pod to pod communication happens in clear text and the goal of mTLS is to be able to encrypt this traffic by using certificates.

A way to make this happen is to use a proxy as a sidecar container, which would be responsible for the communication and certificate exchange, while the app container doesn't need to be touched. An initContainer would add IPTables routes to route the traffic to the app container and to do so, it needs the `NET_ADMIN` capability.

This functionality can be implemented by a ServiceMesh service, such as Istio or Linkerd.

The below example is a simple way to implement a pod with two containers, one acting as a simple app and the other one as a initContainer with `NET_ADMIN` capabilites and therefore, able to change network configurations in the pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: pod
  name: pod
spec:
  containers:
  - args:
    - sh
    - -c
    - ping www.google.com
    image: busybox
    name: pod
  - name: proxy
    image: debian
    args:
    - bash
    - -c
    - 'apt update && apt -y install iptables && iptables -L'
    securityContext:
      capabilities:
        add: ["NET_ADMIN"]
```



### Auditing in kubernetes

- Stages
  - RequestReceived: The API server audit handler receives the request;
  - ResponseStarted: The response header is sent and more data will follow;
  - ResponseComplete: The complete response is sent (header and body) and no more data is going to be sent;
  - Panic: Generated when a panic event happens.
- Levels
  - None: no message that matches a given rule will be logged;
  - Metadata: Logs the request metadata, which contains: requesting user, timestamp, resource, verb, etc. Doesn't log the request or body.
  - Request: Logs the metadata and the request body (not the response body). Doesn't apply to non-resource requests.
  - RequestResponse: Highest level, logs all the above. Also doesn't apply to non-resource requests.

Configuring audit in the cluster:

- In the master node, create a directory to hold the configuration (`/etc/kubernetes/audit`, for example);
- Create the policy file in this folder (`policy.yaml`, for example);

A policy that logs everything on `Metadata` level:

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
```

A more complex policy:

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
# Nothing from RequestReceived stage.
omitStages:
  - "RequestReceived"
rules:
  # nothing from verbs get, list and watch
  - level: None
    verbs: ["get","list","watch"]
  # Metadata level for secrets
  - level: Metadata
    resources:
    - group: ""
      resources: ["secrets"]
  # Everything from RequestResponse level
  - level: RequestResponse
```

- Enable audit in the api server manifest (`kube-apiserver.yaml`), adding the lines that can be found in the docs. The basic configuration involves:

Configuring the audit system itself;

```yaml
(...)
    - --audit-policy-file=/etc/kubernetes/audit/policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-log-maxsize=500
    - --audit-log-maxbackup=5
(...)
```

Making the directory with the policy file available to the kube-apiserver pod.

```yaml
(...)
    - mountPath: /etc/kubernetes/audit
      name: audit
      readOnly: true
    - mountPath: /var/log/kubernetes/
      name: audit-logs
(...)
  - hostPath:
      path: /etc/kubernetes/audit
      type: DirectoryOrCreate
    name: audit
  - hostPath:
      path: /var/log/kubernetes/
      type: DirectoryOrCreate
    name: audit-logs
(...)
```

#### Use case: investigate the access history of a secret

For this to work the audit policy needs to include RequestResponse from secrets. Below examples to test:

- Change the policy. The below example contains more than necessary configs.

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
# Nothing from RequestReceived stage.
omitStages:
  - "RequestReceived"
rules:
  # Removing some messages to make analysis easier
  # Nothing from some internal components
  - level: None
    users:
    - "system:kube-scheduler"
    - "system:kube-proxy"
    - "system:apiserver"
    - "system:kube-controller-manager"
    - "system:serviceaccount:gatekeeper-system:gatekeeper-admin"

  # Nothing from worker nodes
  - level: None
    userGroups: ["system:nodes"]

  # RequestResponse level for secrets
  - level: RequestResponse
    resources:
    - group: ""
      resources: ["secrets"]

  # Everything from RequestResponse level
  - level: RequestResponse
```

- Create new ServiceAccount and secret;

```bash
kubectl create sa my-super-sa
# a token should be automatically generated
kubectl get secret

NAME                      TYPE                                  DATA   AGE
default-token-54x59       kubernetes.io/service-account-token   3      40d
my-super-sa-token-qg7xh   kubernetes.io/service-account-token   3      12s
```

- Create some resource that leverages this ServiceAccount (a pod for example).

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: my-super-pod
  name: my-super-pod
spec:
  serviceAccountName: my-super-sa
  containers:
  - args:
    - sleep
    - "3600"
    image: busybox
    name: my-super-pod
```

- Grep the pod and ServiceAccount names in the log file. For example:

```bash
grep my-super-pod audit.log | grep my-super-sa | tail -1 | jq
```









## 
