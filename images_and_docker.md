### CIS benchmark

List of security best practices for systems, which were addapeted to kubernetes by the `kube-bench` project. It contains tests that can be executed from different ways to check if the cluster is compliant with the guidelines.

- Running it from a docker container, which can be from a master or a worker node.

```bash
sudo docker run --pid=host -v /etc:/etc:ro -v /var:/var:ro -t aquasec/kube-bench:latest --version 1.21
```



### Docker security related aspects

- Reduce the final image size by using multi stage builds. Building smaller images means:
  - Easier to deploy;
  - Less binaries, lower probability of exploitable applications.

- Other good practices, to be applied when possible:
  - Don't build the image using another image with the 'latest' tag. Try to select the specific version;
  - Avoid running the application as root;
  - Remove filesystem permissions;
  - Remove the shell.

```dockerfile
FROM ubuntu:20.04
ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y golang-go
COPY app.go .
RUN CGO_ENABLED=0 go build app.go

FROM alpine:3.14.2
RUN addgroup -S group && adduser -S lameuser -G group -h /home/lameuser
COPY --from=0 /app /home/lameuser
RUN chmod u-w /etc
RUN rm /bin/*
USER lameuser
CMD ["/home/user/app"]
```



### Use credentials to log into private registries

When using private image registries is necessary to inform the credentials apiserver so it can pull the images from the repository. To do so, first a secret should be created:

```bash
kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
```

Then a service account which will use the secret and be used by the pods that uses the image.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-reg
spec:
  containers:
  - name: private-reg-container
    image: <your-private-image>
  imagePullSecrets:
  - name: regcred
```



### Use the image digest (hash) instead the tag

To be 100% sure that the image being used by some pods is legit, instead of using tags, which can be forged, the image digest can be used. A way to get the image hash is to get the information from a pod with a container executing the wanted image. For example:

```bash
kubectl get pods -o json | jq '.items[0].status.containerStatuses[0].imageID'
"docker-pullable://mstelles/multi@sha256:55c162aaaa83f6561d142d411c5f92140dc0a7c810ed1fa71e451e580eef2c07"
```

This same hash can be found on `hub.docker.com`, searching for the image and selecting the desired release.

Get this hash and use it as label.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: multi
  name: multi
spec:
  containers:
  - image: mstelles/multi@sha256:55c162aaaa83f6561d142d411c5f92140dc0a7c810ed1fa71e451e580eef2c07
    name: multi
```



### Security analysis

- Kubesec: can be used as a docker container to test a given pod manifest file. It will only perform an analysis on the configuration, not image used itself.

```bash
kubectl run multi --image=mstelles/multi -o yaml --dry-run=client
docker run -i kubesec/kubesec:512c5e0 scan /dev/stdin < pod.yaml
```

Based on the output it's possible to make adjustments to enhance pod security.

- Trivy: can be used to check vulnerabilities on docker images.

```bash
docker run ghcr.io/aquasecurity/trivy:latest image <image name>
```

- Falco: A service which runs in the OS monitoring kernel syscalls that might be suspicious, according to previous definitions. It could for example detect whether a shell was executed inside a pod, if a privileged container was started or any other aspect that is added to the configuration.

The main configuration starting point will be `/etc/falco` and by default it will forward the messages via `syslog`.

Example output:

```bash
# executing a shell from the container
06:17:14.397803113: Notice A shell was spawned in a container with an attached terminal (user=root user_loginuid=-1 k8s_alpine_alpine_default_8c3d9a15-f285-4c2b-8b8a-a8772e78ca6f_0 (id=ea7014d98c55) shell=sh parent=runc cmdline=sh terminal=34816 container_id=ea7014d98c55 image=alpine)
# change anything in /etc
06:17:29.054628382: Error File below /etc opened for writing (user=root user_loginuid=-1 command=sh parent=<NA> pcmdline=<NA> file=/etc/anyfile program=sh gparent=<NA> ggparent=<NA> gggparent=<NA> container_id=ea7014d98c55 image=alpine)
# run an app related command (apt, yum, apk)
06:26:57.516232567: Error Package management process launched in container (user=root user_loginuid=-1 command=apk add vim container_id=ea7014d98c55 container_name=k8s_alpine_alpine_default_8c3d9a15-f285-4c2b-8b8a-a8772e78ca6f_0 image=alpine:latest)
```



### Immutability

The act of not changing the container/pod/VM while it's running, as the application should be already deployed as it should be. Some mechanisms can be used to enforce it and on extreme scenarios init pods or even `startupProbe` can be used to make last minute changes to the pods.

Example 1: Setting the fliesystem as `RO` at container level, but using an `emptyDir` to allow a service to run.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: httpd
  name: httpd
spec:
  containers:
  - image: httpd
    name: httpd
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - mountPath: /usr/local/apache2/logs
      name: apache-logs
  volumes:
  - name: apache-logs
```

Example 2: use a `startupProbe` to remove all shells from the pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: httpd
  name: httpd
spec:
  containers:
  - image: httpd
    name: httpd
    startupProbe:
      exec:
        command:
        - rm
        - /bin/*sh
      initialDelaySeconds: 1
      periodSeconds: 5
```



