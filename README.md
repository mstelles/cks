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



### 