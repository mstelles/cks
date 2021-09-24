

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

