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
   
   
