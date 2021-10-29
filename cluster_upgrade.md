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


