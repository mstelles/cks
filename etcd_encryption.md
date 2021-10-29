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


