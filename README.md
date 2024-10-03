# How to reproduce the idpbuilder gitea issue

## Pre-requisites

- podman >= 5.x

## Instructions

- Create a cluster using the `idpbuilder` and install the `Tekton` package

```bash
idpbuilder create -p idp/tekton
```

- When, idp is up and running, check that you can access to 
:
- https://gitea.cnoe.localtest.me:8443
- https://tekton-ui.cnoe.localtest.me:8443


- Expose the Podman API as TCP Service on the port `2375`
```bash
podman system service tcp:0.0.0.0:2375 --log-level=debug --time=0 &
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

- Deploy the Tekton pipeline able to build/test a Quarkus application and pushing the image
  to `gitea.cnoe.localtest.me:8443/giteaadmin/my-quarkus-app`

- Open the Tekton UI: https://tekton-ui.cnoe.localtest.me:8443 and follow the execution of the following job: `https://tekton-ui.cnoe.localtest.me:8443/#/namespaces/demo/pipelineruns/build-push-image`

