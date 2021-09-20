## CKS study files

### Namespaces

```bash
# Running a simple docker container
$ docker container run --name c1 -d ubuntu sh -c 'sleep 1d'
$ docker container exec c1 ps auxfw
$ ps auxfe
# Running another docker container to check isolagion of processess
$ docker container run --name c2 -d ubuntu sh -c 'sleep 100d'
$ ps auxfe
# Remove c2 and recreate using the same PID namespace from c1
$ docker container rm c2 --force
$ docker container run --name c2 --pid=container:c1 -d ubuntu sh -c 'sleep 100d'
$ docker container exec c1 ps auxfw
$ docker container exec c2 ps auxfw
# For the kernel, the PIDs will allways be different
ps auxfe
# Remove the containers
docker container rm c1 --force
docker container rm c2 --force
docker container ps
```



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
$ kubectl create -f https://github.com/cks/default-deny.yaml
$ kubectl get netpol
# test connectivity between the pods once again - it will fail
$ kubectl exec frontend -- curl backend
$ kubectl exec backend -- curl frontend
```


