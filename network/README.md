

### Network policies

All files used here can be found on this repository: https://github.com/mstelles/cks/blob/main/network/policies

- Simple `deny-all` policy.

```bash
# run two simple nginx pods
$ kubectl run frontend --image=nginx
$ kubectl run backend --image=nginx
# expose the pods
$ kubectl expose pod frontend --port 80
$ kubectl expose pod backend --port 80
# test connectivity between the pods
$ kubectl exec frontend -- curl backend
$ kubectl exec backend -- curl frontend
# create the `default-deny` network policy
$ kubectl create -f https://raw.githubusercontent.com/mstelles/cks/main/network/policies/default-deny.yaml
$ kubectl get netpol
# test connectivity between the pods once again - it will fail
$ kubectl exec frontend -- curl backend
$ kubectl exec backend -- curl frontend
```

- Allow **any** TCP traffic from frontend to backend, using two separate policy files:
  - The pods will by defaul have the `run: <pod name>` label and this is what is going to be used as referrence;
  - frontend.yaml: allows all egress traffic from frontend pods;
  - backend.yaml: allows all infress traffic to backend pods.

```bash
# apply the policy to the backend pods
$ kubectl create -f https://raw.githubusercontent.com/mstelles/cks/main/network/policies/backend.yaml
# apply the policy to the frontend pods
$ kubectl create -f https://raw.githubusercontent.com/mstelles/cks/main/network/policies/frontend.yaml
# check the policies
$ kubectl get netpol
```

At this point, tests are possible **only** using IP addresses as DNS name resolution is not allowed.

```bash
$ kubectl get pods -o wide -l=run=backend
NAME      READY   STATUS    RESTARTS   AGE   IP          NODE          NOMINATED NODE   READINESS GATES
backend   1/1     Running   2          39h   10.44.0.2   k8sworker01   <none>           <none>
$ kubectl exec frontend -- curl -Is 10.44.0.2
HTTP/1.1 200 OK
Server: nginx/1.21.3
Date: Thu, 23 Sep 2021 14:06:50 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 07 Sep 2021 15:21:03 GMT
Connection: keep-alive
ETag: "6137835f-267"
Accept-Ranges: bytes
```

- Allow DNS name resolution: 
  - Create a separate policy to do so (my choice - dns.yaml) or add it to any other policy.

```bash
# apply the policy to allow DNS name resolution
$ kubectl apply -f https://raw.githubusercontent.com/mstelles/cks/main/network/policies/dns.yaml
```

- Test the connection using the DNS name

```bash
$ kubectl exec frontend -- curl -Is backend
HTTP/1.1 200 OK
Server: nginx/1.21.3
Date: Thu, 23 Sep 2021 14:08:12 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 07 Sep 2021 15:21:03 GMT
Connection: keep-alive
ETag: "6137835f-267"
Accept-Ranges: bytes
```



- Create a DB layer in another namespace and allow communication from the backend pod to the pod and service in the DB namespace.

```bash
# create the namespace and add a label to it
$ kubectl create ns db -o yaml --dry-run=client > db_ns.yaml
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: null
  name: db
  labels:
    ns: db
spec: {}
status: {}
```

- Create and expose a simple pod with the nginx image in the db label, for testing purposes.

```bash
$ kubectl run --image=nginx db --namespace db
$ kubectl -n db expose pod db --port=80
```

- Add a new entry to the `backend.yaml` file allowing outbound traffic to the db namespace. Full updated policy [here.](https://raw.githubusercontent.com/mstelles/cks/main/network/policies/backend_db.yaml "backend_db.yaml")

```yaml
(...)
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          ns: db
```

- To create a more secure environment, also create a default deny policy for the db namespace, along with the appropriate rules to allow the communication to the pod and DNS name resolution.

```bash
```

- Test connecting from `backend` pod to `db` pod in the `db namespace`

```bash
# testing using the db pod as destination
$ kubectl exec backend -- curl -Is 10.44.0.5
# testing using the FQDN (<pod>.<namespace>)
$ kubectl exec backend -- curl -Is db.db
```

- On both cases, the expected output would be similar to the one shown below.

```bash
HTTP/1.1 200 OK
Server: nginx/1.21.3
Date: Thu, 23 Sep 2021 13:55:50 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 07 Sep 2021 15:21:03 GMT
Connection: keep-alive
ETag: "6137835f-267"
Accept-Ranges: bytes
```



### Ingress

- A wrapper that will generate a config which will run in the pod responsible for the service. There's going to have a service as well, pointing to the ingress pods.
- Nginx ingress installation (search for nginx ingress installation and head to `bare-metal`)
  - OBS: testing with k8s v1.21, the ingress controller versions above `0.49.3` didn't work.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.49.3/deploy/static/provider/baremetal/deploy.yaml
```

- Checking:

```bash
kubectl -n ingress-nginx get pod,svc
NAME                                            READY   STATUS      RESTARTS   AGE
pod/ingress-nginx-admission-create-vhgbd        0/1     Completed   0          2m52s
pod/ingress-nginx-admission-patch-qm7rf         0/1     Completed   1          2m52s
pod/ingress-nginx-controller-5486956f45-zngsk   1/1     Running     0          2m52s

NAME                                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             NodePort    10.105.173.152   <none>        80:32319/TCP,443:30991/TCP   2m52s
service/ingress-nginx-controller-admission   ClusterIP   10.111.116.106   <none>        443/TCP                      2m52s
```

- Create the configuration (get from kubernetes.io an example).

```yaml
kubectl apply -f https://raw.githubusercontent.com/mstelles/cks/main/network/ingress/ingress.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /nginx
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80

      - path: /httpd
        pathType: Prefix
        backend:
          service:
            name: httpd=svc
            port:
              number: 80

```

- Create and expose two pods to act as backend, using the same name as specified on the `ingress` configuration.

```bash
kubectl run nginx --image=nginx
kubectl run httpd --image=httpd
kubectl expose pod httpd --name httpd-svc --port 80
kubectl expose pod nginx --name nginx-svc --port 80
```

- To test, I used another simple pod just to execute `curl` commands, using the node IP and the port from the service.

```bash
kubectl -n ingress-nginx get svc ingress-nginx-controller
NAME                       TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller   NodePort   10.104.226.221   <none>        80:32678/TCP,443:30802/TCP   58m
kubectl get nodes -o wide
NAME          STATUS   ROLES                  AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
k8smaster     Ready    control-plane,master   19d   v1.21.0   192.168.1.160   <none>        Ubuntu 18.04.5 LTS   4.15.0-159-generic   docker://20.10.7
k8sworker01   Ready    <none>                 19d   v1.21.0   192.168.1.161   <none>        Ubuntu 18.04.5 LTS   4.15.0-159-generic   docker://20.10.7
```



```bash
kubectl run multi --image=mstelles/multi
# testing the nginx service
kubectl exec -ti multi -- curl http://192.168.1.161:32678/nginx
# testing the httpd service
kubectl exec -ti multi -- curl http://192.168.1.161:32678/httpd
```

