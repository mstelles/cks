

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
