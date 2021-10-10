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



