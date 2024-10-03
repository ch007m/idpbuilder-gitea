# How to reproduce the idpbuilder gitea issue

## Pre-requisites

- podman >= 5.x

## Instructions

- Create a Kind cluster using the [idpbuilder](https://github.com/cnoe-io/idpbuilder/) CLI (>= 0.7) and install the `Tekton` package

```bash
export DOCKER_HOST="unix:///var/run/docker.sock"
idpbuilder create --color -p idp/packages/tekton
```

- When, idp is up and running, check that you can access the following urls:

  - https://gitea.cnoe.localtest.me:8443
  - https://tekton-ui.cnoe.localtest.me:8443


- Expose the Podman API as TCP Service on the port `2375`.
**Note**: If podman runs in a VM on your laptop, ssh first to it: `podman machine ssh`
```bash
podman system service tcp:0.0.0.0:2375 --log-level=debug --time=0 &
...
DEBU[0000] Called service.PersistentPreRunE(podman system service tcp:0.0.0.0:2375 --log-level=debug --time=0)
DEBU[0000] Using conmon: "/usr/bin/conmon"
INFO[0000] Using sqlite as database backend
DEBU[0000] Using graph driver overlay
DEBU[0000] Using graph root /var/lib/containers/storage
DEBU[0000] Using run root /run/containers/storage
DEBU[0000] Using static dir /var/lib/containers/storage/libpod
DEBU[0000] Using tmp dir /run/libpod
DEBU[0000] Using volume path /var/lib/containers/storage/volumes
DEBU[0000] Using transient store: false
DEBU[0000] [graphdriver] trying provided driver "overlay"
DEBU[0000] overlay: imagestore=/usr/lib/containers/storage
DEBU[0000] Cached value indicated that overlay is supported
DEBU[0000] Cached value indicated that overlay is supported
DEBU[0000] Cached value indicated that metacopy is being used
DEBU[0000] NewControl(/var/lib/containers/storage/overlay): nextProjectID = 2420729484
DEBU[0000] Cached value indicated that native-diff is not being used
INFO[0000] Not using native diff for overlay, this may cause degraded performance for building images: kernel has CONFIG_OVERLAY_FS_REDIRECT_DIR enabled
DEBU[0000] backingFs=xfs, projectQuotaSupported=true, useNativeDiff=false, usingMetacopy=true
DEBU[0000] Initializing event backend journald
DEBU[0000] Configured OCI runtime crun-vm initialization failed: no valid executable found for OCI runtime crun-vm: invalid argument
DEBU[0000] Configured OCI runtime runc initialization failed: no valid executable found for OCI runtime runc: invalid argument
DEBU[0000] Configured OCI runtime runsc initialization failed: no valid executable found for OCI runtime runsc: invalid argument
DEBU[0000] Configured OCI runtime krun initialization failed: no valid executable found for OCI runtime krun: invalid argument
DEBU[0000] Configured OCI runtime runj initialization failed: no valid executable found for OCI runtime runj: invalid argument
DEBU[0000] Configured OCI runtime kata initialization failed: no valid executable found for OCI runtime kata: invalid argument
DEBU[0000] Configured OCI runtime youki initialization failed: no valid executable found for OCI runtime youki: invalid argument
DEBU[0000] Configured OCI runtime ocijail initialization failed: no valid executable found for OCI runtime ocijail: invalid argument
DEBU[0000] Using OCI runtime "/usr/bin/crun"
INFO[0000] Setting parallel job count to 19
WARN[0000] Using the Podman API service with TCP sockets is not recommended, please see `podman system service` manpage for details
...
```
- Grab the ip address of the `kind` network
```bash
podman network inspect kind | jq -r '.[].subnets.[1].gateway' 
```
- Update the parameter: `podman-host` within the file `kubernetes/02-pipelinerun.yml` 

- Create a `demo` namespace and kubernetes secret with the registry credentials of your `dockerhub` account and the one of gitea
```bash
podman login docker.io -u xxxx -p xxxx
podman login --tls-verify=false gitea.cnoe.localtest.me:8443 -u giteaAdmin -p $(idpbuilder get secrets -o json -p gitea | jq -r '.[].data.password')
kubectl create secret generic dockerconfig-secret --from-file=config.json=$HOME/.config/containers/auth.json -n demo
```

- Deploy the Tekton pipeline able to build/test a Quarkus application and pushing the image
  to `gitea.cnoe.localtest.me:8443/giteaadmin/my-quarkus-app`
```bash
kubectl -n demo apply -f pipelines
tkn -n demo pr logs -f
```
**Note**: To replay, delete the resources `kubectl -n demo delete -f pipelines` and re-deploy them

- Open the Tekton UI: https://tekton-ui.cnoe.localtest.me:8443 and follow the execution of the following job: `https://tekton-ui.cnoe.localtest.me:8443/#/namespaces/demo/pipelineruns/build-push-image`

- Observe at the step `build-and-push` what it is happening
```text
...
I got another issue now even using the hack 
[buildah-image : build-and-push] time="2024-10-03T17:23:15Z" level=warning msg="Failed, retrying in 4s ... (3/3). Error: trying to reuse blob sha256:dd5e77a90e609b328f2e49aa60e50bd8837e505c157060c337725413ccf449f1 at destination: pinging container registry gitea.cnoe.localtest.me:8443: Get \"https://gitea.cnoe.localtest.me:8443/v2/\": dial tcp 127.0.0.1:8443: connect: connection refused"
[buildah-image : build-and-push] Getting image source signatures
[buildah-image : build-and-push] Copying blob sha256:72fa8206770d7c12f6e10be169329790d09af8fce623d03a6e48fdc3d6a36436
[buildah-image : build-and-push] Copying blob sha256:dd5e77a90e609b328f2e49aa60e50bd8837e505c157060c337725413ccf449f1
[buildah-image : build-and-push] Copying blob sha256:8b30b41a0b038bf660c4538ae04bb77b7cdbfcfcb0f5a378129ddf82b91542e7
[buildah-image : build-and-push] Error: pushing image "gitea.cnoe.localtest.me:8443/giteaadmin/my-quarkus-app" to "docker://gitea.cnoe.localtest.me:8443/giteaadmin/my-quarkus-app": trying to reuse blob sha256:dd5e77a90e609b328f2e49aa60e50bd8837e505c157060c337725413ccf449f1 at destination: pinging container registry gitea.cnoe.localtest.me:8443: Get "https://gitea.cnoe.localtest.me:8443/v2/": dial tcp 127.0.0.1:8443: connect: connection refused

```

## Step validating that a pod can be created using a gitea image

You can verify that we can pull/tag and push an image like also to run a pod using the image pushed on gitea
```bash
podman tag docker.io/library/ubuntu:24.04 gitea.cnoe.localtest.me:8443/giteaadmin/ubuntu:24.04
podman push gitea.cnoe.localtest.me:8443/giteaadmin/ubuntu:24.04 --tls-verify=false
kubectl apply -n demo -f kubernetes/simple.pod.yaml
...
kubectl get -n demo pod/debug-pod
NAME        READY   STATUS    RESTARTS   AGE
debug-pod   1/1     Running   0          2m9s
```

## Step demonstrating that we can ping the gitea server if we add localhost to /etc/hosts

```bash
kubectl -n demo apply -f kubernetes/simple-task.yaml
...
tkn -n demo taskrun logs gitea-server-check

[check-ping] # Kubernetes-managed hosts file.
[check-ping] 127.0.0.1  localhost
[check-ping] ::1        localhost ip6-localhost ip6-loopback
[check-ping] fe00::0    ip6-localnet
[check-ping] fe00::0    ip6-mcastprefix
[check-ping] fe00::1    ip6-allnodes
[check-ping] fe00::2    ip6-allrouters
[check-ping] 10.244.0.57        gitea-server-check-pod
[check-ping] search demo.svc.cluster.local svc.cluster.local cluster.local dns.podman
[check-ping] nameserver 10.96.0.10
[check-ping] options ndots:5
[check-ping] ## Hack to allow to access: gitea.cnoe.localtest.me
[check-ping] ## Pinging gitea.cnoe.localtest.me ...
[check-ping] PING gitea.cnoe.localtest.me (127.0.0.1): 56 data bytes
[check-ping] 64 bytes from 127.0.0.1: seq=0 ttl=64 time=0.251 ms
[check-ping] 64 bytes from 127.0.0.1: seq=1 ttl=64 time=0.111 ms
[check-ping] 64 bytes from 127.0.0.1: seq=2 ttl=64 time=0.141 ms
[check-ping] 64 bytes from 127.0.0.1: seq=3 ttl=64 time=0.163 ms
[check-ping] 
[check-ping] --- gitea.cnoe.localtest.me ping statistics ---
[check-ping] 4 packets transmitted, 4 packets received, 0% packet loss
[check-ping] round-trip min/avg/max = 0.111/0.166/0.251 ms

container step-check-ping has failed  : [{"key":"StartedAt","value":"2024-10-03T17:05:39.973Z","type":3}]
```


