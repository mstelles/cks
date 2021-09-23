

### Network policies

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
$ kubectl create -f https://raw.githubusercontent.com/mstelles/cks/main/default-deny.yaml
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
$ kubectl create -f https://raw.githubusercontent.com/mstelles/cks/main/backend.yaml
# apply the policy to the frontend pods
$ kubectl create -f https://raw.githubusercontent.com/mstelles/cks/main/frontend.yaml
# check the policies
$ kubectl get netpol
```

At this point, tests are possible **only** using IP addresses as DNS name resolution is not allowed.

```bash
$ kubectl get pods -o wide -l=run=backend
NAME      READY   STATUS    RESTARTS   AGE   IP          NODE          NOMINATED NODE   READINESS GATES
backend   1/1     Running   2          39h   10.44.0.2   k8sworker01   <none>           <none>
$ kubectl exec frontend -- curl 10.44.0.2
```

- Allow DNS name resolution: 
  - Create a separate policy to do so (my choice - dns.yaml) or add it to any other policy.

```bash
# apply the policy to allow DNS name resolution
$ kubectl apply -f https://raw.githubusercontent.com/mstelles/cks/main/dns.yaml
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

- Add a new entry to the `backend.yaml` file allowing outbound traffic to the db namespace. Full updated policy [here.](https://raw.githubusercontent.com/mstelles/cks/main/backend_db.yaml "backend_db.yaml")

```yaml
(...)
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          ns: db
```

