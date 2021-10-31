### Instance metadata (particular to cloud providers)

On cloud providers the instances may expose information via the internal metadata API service (normally http://169.254.169.254). To restrict access to such service, it's possible to leverage the network policies.

### Kubernetes and Operational system binaries

The OS where kubernetes is running will have other binaries and services, which can be used to gain access to the system. This means that it's important to keep track of any unwanted change on any file. This includes not only the binaries, but libraries and configuration files as well.

Kubernetes shares the sha512 hash for the files available to download, and this can be used as source of truth to check if the files are sane.

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
# The OS kernel
kubectl get nodes -o wide -l=kubernetes.io/hostname=k8sworker01
NAME          STATUS   ROLES    AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
k8sworker01   Ready    <none>   27d   v1.21.0   192.168.1.161   <none>        Ubuntu 18.04.5 LTS   4.15.0-156-generic   docker://20.10.7
```



### OS level security domains

- Security Contexts and pod security.

By default the pod will execute with `root` permissions and using this feature it is possible to change the privileges for pods or specific containers that are running in a pod. Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: multi
  name: multi
spec:
  containers:
  - image: mstelles/multi
    name: multi
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
```

```bash
kubectl exec -ti multi -- id
uid=1000 gid=3000 groups=3000,2000
```

It's also possible to use the `runAsNonRoot: true` option but this will only work with containers that were built to run as a non root user.

- Privileged containers: a way to map the root user from the OS to the root user from the container. One of the applications of this feature is to allow docker to run inside a docker container. Other examples can be changing kernel parameters or any other tasks that involve direct interaction with the kernel.

For example, in a pod that it's not running on the `privileged` mode the below output is expected when trying to change anh kernel parameter. Not that the user inside the pod is `root`, just not the "same" `root` user from the OS.

```bash
root@multi:/# id
uid=0(root) gid=0(root) groups=0(root)

root@multi:/# sysctl net.ipv4.ip_forward 
net.ipv4.ip_forward = 1

root@multi:/# sysctl net.ipv4.ip_forward=0
sysctl: setting key "net.ipv4.ip_forward": Read-only file system
```

However, applying the below configuration to the pod it's possible to execute the command and change any needed kernel parameter.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: multi
  name: multi
spec:
  containers:
  - image: mstelles/multi
    name: multi
  securityContext:
    privileged: true
```

```bash
root@multi:/# id
uid=0(root) gid=0(root) groups=0(root)

root@multi:/# sysctl net.ipv4.ip_forward=0
net.ipv4.ip_forward = 0
```

- Privilege escalation: capability that an application to gain more privilege during it's execution (sudo, for example). It's enabled by default and to disable, use the `allowPrivilegeEscalation: false` in the security context. This is leveraged by the `NoNewPrivs` kernel syscall. Feature also enabled by default.

```bash
kubectl exec -ti multi -- bash -c 'grep NoNewPrivs /proc/1/status'
NoNewPrivs:	0
```

Disabling it:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: multi
  name: multi
spec:
  containers:
  - image: mstelles/multi
    name: multi
  securityContext:
    allowPrivilegeEscalation: false
```

```bash
kubectl exec -ti multi -- bash -c 'grep NoNewPrivs /proc/1/status'
NoNewPrivs:	1
```

- Pod security policies: It's an addmission controller and when enabled (in the tube-apiserver) and configured, will affect all pods running in the cluster.

Steps to enable, configure and test it.

1. Enable in the `kube-apiserver`. Edit the manifest (`/etc/kubernetes/manifests/kube-apiserver.yaml`) and make the below change:

```bash
- --enable-admission-plugins=NodeRestriction
to
- --enable-admission-plugins=NodeRestriction,PodSecurityPolicy
```

2. Create the policy

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: testpsp
spec:
  # denying privileged pods and privilege escalation
  privileged: false
  allowProvilegeEscalation: false
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - '*'
```

From this moment onwards, manually running pods is still allowed as the pod is executed with the admin user rights.

```bash
kubectl run multi --image=mstelles/multi
pod/multi created

kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
multi   1/1     Running   0          14s
```

Running pods via deployments, however is not allowed as the replicaset won't have permissions to launch the pods. And this is because the default service account is not using the PSP.

```bash
kubectl create deploy multi --image=mstelles/multi
deployment.apps/multi created

kubectl get deploy multi
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
multi   0/1     0            0           2m14s
```

3. To allow the deployments (and other objects) to use the PSP, create a role and a role binding.

```bash
kubectl create role psp-access --verb=use --resource=podsecuritypolicies
role.rbac.authorization.k8s.io/psp-access created

kubectl create rolebinding psp-access --role=psp-access --serviceaccount=default:default
rolebinding.rbac.authorization.k8s.io/psp-access created
```

4. Recreate the deployment and the pods should start running.

```bash
kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
multi                   1/1     Running   0          10m
multi-5dd8d6c89-xjrph   1/1     Running   0          4s

kubectl get deploy multi
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
multi   1/1     1            1           17s
```

5. Another test is trying to run a pod in `privileged` mode or with `allowPrivilegeEscalation` set to `true`. By doing so, a message similar to the one below should appear.

```bash
Error from server (Forbidden): error when creating "multi.yaml": pods "multi" is forbidden: PodSecurityPolicy: unable to admit pod: [spec.containers[0].securityContext.allowPrivilegeEscalation: Invalid value: true: Allowing privilege escalation for containers is not allowed]
```



#### Controlling access to syscalls and resources on OS level

- AppArmor: a Linux kernel module that intercepts syscalls made from applications on user space before it reaches the kernel. It works with profiles, which can operate on three modes:
  - Enforce: the most restrictive mode, where the syscalls will be intercepted and some action may be taken, depending on the config.
  - Complain: the syscalls made by the application will be registered but no action will be taken.
  - Unconfined: process have full access to whatever syscalls.

First instal `apparmor-utils` to manipulate the configurations. With this package in place, it's possible to generate a profile for the application and after it, update this profile with the needed details to allow the app to run, when it's the case. This is automatically done by apparmor by reading the log files from syslog. Note that AppArmor has to be configured on all worker nodes.

Example: 

```bash
# type `F` when asked
aa-gen-prof curl
# check the profile
cat /etc/apparmor.d/usr.bin.curl
# test the curl command
curl google.com
curl: (6) Could not resolve host: google.com
```

After the creation of the profile and execution of the `curl` command, the access to the URL was denied and some entries were added to `/var/log/syslog`. This allows to check other legit files needed by the application and in the future if the behaviour changes, the access to other resources would be denied.

```bash
aa-logprof

Reading log entries from /var/log/syslog.
Updating AppArmor profiles in /etc/apparmor.d.
Enforce-mode changes:

Profile:  /usr/bin/curl
Path:     /etc/ssl/openssl.cnf
New Mode: owner r
Severity: 2

 [1 - #include <abstractions/lxc/container-base>]
  2 - #include <abstractions/lxc/start-container>
  3 - #include <abstractions/openssl>
  4 - #include <abstractions/ssl_keys>
  5 - owner /etc/ssl/openssl.cnf r,
(A)llow / [(D)eny] / (I)gnore / (G)lob / Glob with (E)xtension / (N)ew / Audi(t) / (O)wner permissions off / Abo(r)t / (F)inish
Adding #include <abstractions/lxc/container-base> to profile.
Deleted 2 previous matching profile entries.

= Changed Local Profiles =

The following local profiles were changed. Would you like to save them?

 [1 - /usr/bin/curl]
(S)ave Changes / Save Selec(t)ed Profile / [(V)iew Changes] / View Changes b/w (C)lean profiles / Abo(r)t
Writing updated profile for /usr/bin/curl.
```

Extending this to docker, it's possible to create a profile and invoke it when running the container. This will cause the container's execution to be controlled by AppArmor.

To do so, first get the contents of the [docker-generic](https://raw.githubusercontent.com/mstelles/cks/main/apparmor-generic_profile) file, add it to `/etc/apparmor.d/docker-generic` and run:

```bash
apparmor_parser /etc/apparmor.d/docker-generic
```

WIth this configuration active, now it's possible to execute the pods using this profile.

```bash
docker run -dt --name=busybox --security-opt apparmor=docker-generic busybox
```

Now, execing a shell inside the container, some operations won't be allowed, such as invoking a shell, creating/changing files in some directories and so on.

```bash
sh
sh: sh: Permission denied
echo asdasd >> /etc/shadow
sh: can't create /etc/shadow: Permission denied
```

To use AppArmor with kubernetes, the pods have to be deployed with a certain annotation, following the below example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: busybox
  annotations:
    container.apparmor.security.beta.kubernetes.io/busybox: localhost/docker-generic
  name: busybox
(...)
```



- Seccomp: in a similar way when comparing to AppArmor, it controls the access from applications to kernel syscalls.

In Kubernetes it should first be enabled via the `kubelet` daemon, indicating where the profiles are going to be stored (`/var/lib/kubelet/seccomp` by default) and then can be used from the pods.

On this example, using the contents of a [generic](https://raw.githubusercontent.com/mstelles/cks/main/seccomp-generic_profile) Seccomp profile. This configuration lies in the worker nodes as well (similar to AppArmor).

Use a simple pod manifest to test and one difference from AppArmor is that for seccomp the definition is as a `securityContext`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: busybox
  name: busybox
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: audit.json
(...)
```

