# Scenario using idpbuilder and gitea

As the issue identified part of this test case project has been [fixed](#issue-fixed), then the documentation has been updated to demonstrate
some use cases about gitea as repository of container's images with idpbuilder.

## Pre-requisites

- podman >= 5.x
- idpbuilder >= 0.8

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

kubectl create ns demo
kubectl create secret generic dockerconfig-secret --from-file=config.json=$HOME/.config/containers/auth.json -n demo
```

## Tekton pipeline building an Quarkus application and using podman remote like gitea as image repository

- Deploy the Tekton pipeline able to build/test a Quarkus application and pushing the image
  to `gitea.cnoe.localtest.me:8443/giteaadmin/my-quarkus-app`
```bash
kubectl -n demo apply -f pipelines
tkn -n demo pr logs -f
```
**Note**: To replay, delete the resources `kubectl -n demo delete -f pipelines` and re-deploy them

- Open the Tekton UI: https://tekton-ui.cnoe.localtest.me:8443 and follow the execution of the following job: `https://tekton-ui.cnoe.localtest.me:8443/#/namespaces/demo/pipelineruns/build-push-image`

- Observe at the step `build-and-push` that buildah can build and push the image
```text
...
[buildah-image : build-and-push] ## Buildah version
[buildah-image : build-and-push] buildah version 1.37.3 (image-spec 1.1.0, runtime-spec 1.2.0)
[buildah-image : build-and-push] ## Build the project ...
[buildah-image : build-and-push] STEP 1/11: FROM registry.access.redhat.com/ubi8/openjdk-21:1.20
[buildah-image : build-and-push] Trying to pull registry.access.redhat.com/ubi8/openjdk-21:1.20...
[buildah-image : build-and-push] Getting image source signatures
[buildah-image : build-and-push] Checking if image destination supports signatures
[buildah-image : build-and-push] Copying blob sha256:8dc97931d0a29118b7e8dd695ac355f7b569223a972f3eb87f2ff07fc9fc190a
[buildah-image : build-and-push] Copying blob sha256:b46e4e7892d6177335aee5445f59105231c351f2fb68a24f25ee7b2656e29674
[buildah-image : build-and-push] Copying config sha256:b8d81704f56858c6859ce949133635bb716162ddd3c8d012ec77572449403153
[buildah-image : build-and-push] Writing manifest to image destination
[buildah-image : build-and-push] Storing signatures
[buildah-image : build-and-push] STEP 2/11: ENV LANGUAGE='en_US:en'
[buildah-image : build-and-push] STEP 3/11: COPY --chown=185 target/quarkus-app/lib/ /deployments/lib/
[buildah-image : build-and-push] STEP 4/11: COPY --chown=185 target/quarkus-app/*.jar /deployments/
[buildah-image : build-and-push] STEP 5/11: COPY --chown=185 target/quarkus-app/app/ /deployments/app/
[buildah-image : build-and-push] STEP 6/11: COPY --chown=185 target/quarkus-app/quarkus/ /deployments/quarkus/
[buildah-image : build-and-push] STEP 7/11: EXPOSE 8080
[buildah-image : build-and-push] STEP 8/11: USER 185
[buildah-image : build-and-push] STEP 9/11: ENV JAVA_OPTS_APPEND="-Dquarkus.http.host=0.0.0.0 -Djava.util.logging.manager=org.jboss.logmanager.LogManager"
[buildah-image : build-and-push] STEP 10/11: ENV JAVA_APP_JAR="/deployments/quarkus-run.jar"
[buildah-image : build-and-push] STEP 11/11: ENTRYPOINT [ "/opt/jboss/container/java/run/run-java.sh" ]
[buildah-image : build-and-push] COMMIT gitea.cnoe.localtest.me:8443/giteaadmin/my-quarkus-app
[buildah-image : build-and-push] Getting image source signatures
[buildah-image : build-and-push] Copying blob sha256:dd5e77a90e609b328f2e49aa60e50bd8837e505c157060c337725413ccf449f1
[buildah-image : build-and-push] Copying blob sha256:8b30b41a0b038bf660c4538ae04bb77b7cdbfcfcb0f5a378129ddf82b91542e7
[buildah-image : build-and-push] Copying blob sha256:20389ed4aeda74d541449deffb02c642dac8a466318e5659eac7444bfe4343cc
[buildah-image : build-and-push] Copying config sha256:74674d0d1472ecfb8bb2e42cf9a0a96b7d76d4820172aee9d903dae87dc3fd90
[buildah-image : build-and-push] Writing manifest to image destination
[buildah-image : build-and-push] --> 74674d0d1472
[buildah-image : build-and-push] Successfully tagged gitea.cnoe.localtest.me:8443/giteaadmin/my-quarkus-app:latest
[buildah-image : build-and-push] 74674d0d1472ecfb8bb2e42cf9a0a96b7d76d4820172aee9d903dae87dc3fd90
[buildah-image : build-and-push] + buildah --storage-driver=overlay push --tls-verify=false --digestfile /tmp/image-digest gitea.cnoe.localtest.me:8443/giteaadmin/my-quarkus-app docker://gitea.cnoe.localtest.me:8443/giteaadmin/my-quarkus-app
[buildah-image : build-and-push] Getting image source signatures
[buildah-image : build-and-push] Copying blob sha256:20389ed4aeda74d541449deffb02c642dac8a466318e5659eac7444bfe4343cc
[buildah-image : build-and-push] Copying blob sha256:dd5e77a90e609b328f2e49aa60e50bd8837e505c157060c337725413ccf449f1
[buildah-image : build-and-push] Copying blob sha256:8b30b41a0b038bf660c4538ae04bb77b7cdbfcfcb0f5a378129ddf82b91542e7
[buildah-image : build-and-push] Copying config sha256:74674d0d1472ecfb8bb2e42cf9a0a96b7d76d4820172aee9d903dae87dc3fd90
[buildah-image : build-and-push] Writing manifest to image destination
[buildah-image : build-and-push] + set +x
[buildah-image : build-and-push] sha256:4e236a2bce1f0102f505d35a2661d2e9c214f5a84373ca2942a388b3b566768bgitea.cnoe.localtest.me:8443/giteaadmin/my-quarkus-app
```

## Pod created from an image hosted on gitea

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

## Tekton task accessing the gitea server and podman

```bash
kubectl -n demo apply -f kubernetes/simple-task.yaml
...
❯ tkn -n demo taskrun logs curl-gitea -f

[curl-gitea] # Kubernetes-managed hosts file.
[curl-gitea] 127.0.0.1	localhost
[curl-gitea] ::1	localhost ip6-localhost ip6-loopback
[curl-gitea] fe00::0	ip6-localnet
[curl-gitea] fe00::0	ip6-mcastprefix
[curl-gitea] fe00::1	ip6-allnodes
[curl-gitea] fe00::2	ip6-allrouters
[curl-gitea] 10.244.0.48	curl-gitea-pod
[curl-gitea] search demo.svc.cluster.local svc.cluster.local cluster.local dns.podman
[curl-gitea] nameserver 10.96.0.10
[curl-gitea] options ndots:5
[curl-gitea] ## Curl to the registry host: https://gite-a.cnoe.localtest.me:8443/api/swagger ...
[curl-gitea]   % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
[curl-gitea]                                  Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Host gite-a.cnoe.localtest.me:8443 was resolved.
[curl-gitea] * IPv6: (none)
[curl-gitea] * IPv4: 10.96.14.241
[curl-gitea] *   Trying 10.96.14.241:8443...
[curl-gitea] * ALPN: curl offers h2,http/1.1
[curl-gitea] } [5 bytes data]
[curl-gitea] * TLSv1.3 (OUT), TLS handshake, Client hello (1):
[curl-gitea] } [512 bytes data]
[curl-gitea] * TLSv1.3 (IN), TLS handshake, Server hello (2):
[curl-gitea] { [122 bytes data]
[curl-gitea] * TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
[curl-gitea] { [19 bytes data]
[curl-gitea] * TLSv1.3 (IN), TLS handshake, Certificate (11):
[curl-gitea] { [447 bytes data]
[curl-gitea] * TLSv1.3 (IN), TLS handshake, CERT verify (15):
[curl-gitea] { [78 bytes data]
[curl-gitea] * TLSv1.3 (IN), TLS handshake, Finished (20):
[curl-gitea] { [52 bytes data]
[curl-gitea] * TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
[curl-gitea] } [1 bytes data]
[curl-gitea] * TLSv1.3 (OUT), TLS handshake, Finished (20):
[curl-gitea] } [52 bytes data]
[curl-gitea] * SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384 / x25519 / id-ecPublicKey
[curl-gitea] * ALPN: server accepted h2
[curl-gitea] * Server certificate:
[curl-gitea] *  subject: O=cnoe.io
[curl-gitea] *  start date: Oct  7 10:18:58 2024 GMT
[curl-gitea] *  expire date: Oct  7 16:18:58 2025 GMT
[curl-gitea] *  issuer: O=cnoe.io
[curl-gitea] *  SSL certificate verify result: self-signed certificate (18), continuing anyway.
[curl-gitea] *   Certificate level 0: Public key type EC/prime256v1 (256/128 Bits/secBits), signed using ecdsa-with-SHA256
[curl-gitea] } [5 bytes data]
[curl-gitea] * Connected to gite-a.cnoe.localtest.me (10.96.14.241) port 8443
[curl-gitea] * using HTTP/2
[curl-gitea] * [HTTP/2] [1] OPENED stream for https://gite-a.cnoe.localtest.me:8443/api/swagger
[curl-gitea] * [HTTP/2] [1] [:method: GET]
[curl-gitea] * [HTTP/2] [1] [:scheme: https]
[curl-gitea] * [HTTP/2] [1] [:authority: gite-a.cnoe.localtest.me:8443]
[curl-gitea] * [HTTP/2] [1] [:path: /api/swagger]
[curl-gitea] * [HTTP/2] [1] [user-agent: curl/8.10.1]
[curl-gitea] * [HTTP/2] [1] [accept: */*]
[curl-gitea] } [5 bytes data]
[curl-gitea] > GET /api/swagger HTTP/2
[curl-gitea] > Host: gite-a.cnoe.localtest.me:8443
[curl-gitea] > User-Agent: curl/8.10.1
[curl-gitea] > Accept: */*
[curl-gitea] >
[curl-gitea] { [5 bytes data]
[curl-gitea] * TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
[curl-gitea] { [57 bytes data]
[curl-gitea] * TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
[curl-gitea] { [57 bytes data]
[curl-gitea] * Request completely sent off
[curl-gitea] { [5 bytes data]
[curl-gitea] < HTTP/2 404
[curl-gitea] < date: Wed, 09 Oct 2024 08:46:51 GMT
[curl-gitea] < content-type: application/json
[curl-gitea] < content-length: 206
[curl-gitea] < audit-id: 3ace8654-7826-44ba-bb2d-f45c8d3877c2
[curl-gitea] < cache-control: no-cache, private
[curl-gitea] < x-kubernetes-pf-flowschema-uid: 8e5306b9-167c-4bf6-9046-5abe670f4af8
[curl-gitea] < x-kubernetes-pf-prioritylevel-uid: 897c97f1-b1c9-42aa-8ff1-c5a8dc035122
[curl-gitea] < strict-transport-security: max-age=31536000; includeSubDomains
[curl-gitea] <
[curl-gitea] {
[curl-gitea]   "kind": "Status",
[curl-gitea]   "apiVersion": "v1",
[curl-gitea]   "metadata": {},
[curl-gitea]   "status": "Failure",
[curl-gitea]   "message": "the server could not find the requested resource",
[curl-gitea]   "reason": "NotFound",
[curl-gitea]   "details": {},
[curl-gitea]   "code": 404
[curl-gitea] { [206 bytes data]
100   206  100   206    0     0  44802      0 --:--:-- --:--:-- --:--:-- 51500
[curl-gitea] * Connection #0 to host gite-a.cnoe.localtest.me left intact
[curl-gitea] }

[podman-check] Creating a container against daemon host: 10.89.0.1.
[podman-check] host:
[podman-check]   arch: arm64
[podman-check]   buildahVersion: 1.35.4
[podman-check]   cgroupControllers:
[podman-check]   - cpuset
[podman-check]   - cpu
[podman-check]   - io
[podman-check]   - memory
[podman-check]   - pids
[podman-check]   - rdma
[podman-check]   - misc
[podman-check]   cgroupManager: systemd
[podman-check]   cgroupVersion: v2
[podman-check]   conmon:
[podman-check]     package: conmon-2.1.10-1.fc40.aarch64
[podman-check]     path: /usr/bin/conmon
[podman-check]     version: 'conmon version 2.1.10, commit: '
[podman-check]   cpuUtilization:
[podman-check]     idlePercent: 95.92
[podman-check]     systemPercent: 1.43
[podman-check]     userPercent: 2.65
[podman-check]   cpus: 6
[podman-check]   databaseBackend: sqlite
[podman-check]   distribution:
[podman-check]     distribution: fedora
[podman-check]     variant: coreos
[podman-check]     version: "40"
[podman-check]   eventLogger: journald
[podman-check]   freeLocks: 1944
[podman-check]   hostname: localhost.localdomain
[podman-check]   idMappings:
[podman-check]     gidmap: null
[podman-check]     uidmap: null
[podman-check]   kernel: 6.8.8-300.fc40.aarch64
[podman-check]   linkmode: dynamic
[podman-check]   logDriver: journald
[podman-check]   memFree: 289091584
[podman-check]   memTotal: 5754064896
[podman-check]   networkBackend: netavark
[podman-check]   networkBackendInfo:
[podman-check]     backend: netavark
[podman-check]     dns:
[podman-check]       package: aardvark-dns-1.10.0-1.fc40.aarch64
[podman-check]       path: /usr/libexec/podman/aardvark-dns
[podman-check]       version: aardvark-dns 1.10.0
[podman-check]     package: netavark-1.10.3-3.fc40.aarch64
[podman-check]     path: /usr/libexec/podman/netavark
[podman-check]     version: netavark 1.10.3
[podman-check]   ociRuntime:
[podman-check]     name: crun
[podman-check]     package: crun-1.14.4-1.fc40.aarch64
[podman-check]     path: /usr/bin/crun
[podman-check]     version: |-
[podman-check]       crun version 1.14.4
[podman-check]       commit: a220ca661ce078f2c37b38c92e66cf66c012d9c1
[podman-check]       rundir: /run/user/0/crun
[podman-check]       spec: 1.0.0
[podman-check]       +SYSTEMD +SELINUX +APPARMOR +CAP +SECCOMP +EBPF +CRIU +LIBKRUN +WASM:wasmedge +YAJL
[podman-check]   os: linux
[podman-check]   pasta:
[podman-check]     executable: /usr/bin/pasta
[podman-check]     package: passt-0^20240426.gd03c4e2-1.fc40.aarch64
[podman-check]     version: |
[podman-check]       pasta 0^20240426.gd03c4e2-1.fc40.aarch64-pasta
[podman-check]       Copyright Red Hat
[podman-check]       GNU General Public License, version 2 or later
[podman-check]         <https://www.gnu.org/licenses/old-licenses/gpl-2.0.html>
[podman-check]       This is free software: you are free to change and redistribute it.
[podman-check]       There is NO WARRANTY, to the extent permitted by law.
[podman-check]   remoteSocket:
[podman-check]     exists: true
[podman-check]     path: tcp:0.0.0.0:2375
[podman-check]   security:
[podman-check]     apparmorEnabled: false
[podman-check]     capabilities: CAP_CHOWN,CAP_DAC_OVERRIDE,CAP_FOWNER,CAP_FSETID,CAP_KILL,CAP_NET_BIND_SERVICE,CAP_SETFCAP,CAP_SETGID,CAP_SETPCAP,CAP_SETUID,CAP_SYS_CHROOT
[podman-check]     rootless: false
[podman-check]     seccompEnabled: true
[podman-check]     seccompProfilePath: /usr/share/containers/seccomp.json
[podman-check]     selinuxEnabled: true
[podman-check]   serviceIsRemote: true
[podman-check]   slirp4netns:
[podman-check]     executable: /usr/bin/slirp4netns
[podman-check]     package: slirp4netns-1.2.2-2.fc40.aarch64
[podman-check]     version: |-
[podman-check]       slirp4netns version 1.2.2
[podman-check]       commit: 0ee2d87523e906518d34a6b423271e4826f71faf
[podman-check]       libslirp: 4.7.0
[podman-check]       SLIRP_CONFIG_VERSION_MAX: 4
[podman-check]       libseccomp: 2.5.3
[podman-check]   swapFree: 0
[podman-check]   swapTotal: 0
[podman-check]   uptime: 7h 1m 39.00s (Approximately 0.29 days)
[podman-check]   variant: v8
[podman-check] plugins:
[podman-check]   authorization: null
[podman-check]   log:
[podman-check]   - k8s-file
[podman-check]   - none
[podman-check]   - passthrough
[podman-check]   - journald
[podman-check]   network:
[podman-check]   - bridge
[podman-check]   - macvlan
[podman-check]   - ipvlan
[podman-check]   volume:
[podman-check]   - local
[podman-check] registries:
[podman-check]   giteaaaa.cnoe.localtest.me:8443:
[podman-check]     Blocked: false
[podman-check]     Insecure: true
[podman-check]     Location: giteaaaa.cnoe.localtest.me:8443
[podman-check]     MirrorByDigestOnly: false
[podman-check]     Mirrors: null
[podman-check]     Prefix: giteaaaa.cnoe.localtest.me:8443
[podman-check]     PullFromMirror: ""
[podman-check]   search:
[podman-check]   - docker.io
[podman-check] store:
[podman-check]   configFile: /usr/share/containers/storage.conf
[podman-check]   containerStore:
[podman-check]     number: 1
[podman-check]     paused: 0
[podman-check]     running: 1
[podman-check]     stopped: 0
[podman-check]   graphDriverName: overlay
[podman-check]   graphOptions:
[podman-check]     overlay.imagestore: /usr/lib/containers/storage
[podman-check]     overlay.mountopt: nodev,metacopy=on
[podman-check]   graphRoot: /var/lib/containers/storage
[podman-check]   graphRootAllocated: 106769133568
[podman-check]   graphRootUsed: 71081590784
[podman-check]   graphStatus:
[podman-check]     Backing Filesystem: xfs
[podman-check]     Native Overlay Diff: "false"
[podman-check]     Supports d_type: "true"
[podman-check]     Supports shifting: "true"
[podman-check]     Supports volatile: "true"
[podman-check]     Using metacopy: "true"
[podman-check]   imageCopyTmpDir: /var/tmp
[podman-check]   imageStore:
[podman-check]     number: 181
[podman-check]   runRoot: /run/containers/storage
[podman-check]   transientStore: false
[podman-check]   volumePath: /var/lib/containers/storage/volumes
[podman-check] version:
[podman-check]   APIVersion: 5.0.3
[podman-check]   Built: 1715299200
[podman-check]   BuiltTime: Fri May 10 02:00:00 2024
[podman-check]   GitCommit: ""
[podman-check]   GoVersion: go1.22.2
[podman-check]   Os: linux
[podman-check]   OsArch: linux/arm64
[podman-check]   Version: 5.0.3
```

## Issue fixed

The problem reported here is related to an issue with the coreDNS [rewrite](https://coredns.io/plugins/rewrite/) rules as discussed here:

https://github.com/cnoe-io/idpbuilder/issues/398#issuecomment-2396906079

If we change the existing rule
```bash
rewrite stop {
    name regex (.*).{{ .Host }} ingress-nginx-controller.ingress-nginx.svc.cluster.local
}
```
with this one
```bash
 rewrite stop {
   name regex (.*).{{ .Host }} ingress-nginx-controller.ingress-nginx.svc.cluster.local answer auto
 }
 rewrite name exact cnoe.localtest.me ingress-nginx-controller.ingress-nginx.svc.cluster.local
```
then `buildah push` works internally.

As discussed within the ticket [here](https://github.com/cnoe-io/idpbuilder/issues/398#issuecomment-2400467418), the problem can be fixed if the `rewrite rule`
includes an `answer auto` as documented [here](https://coredns.io/plugins/rewrite/#auto-response-name-rewrite) as an answer response is needed by the tools running part of a fedora, podman, buildah, skopeo images. Such an answer rewrite of the requests is required as some DNS resolvers treat mismatches between the QUESTION SECTION and ANSWER SECTION as a man-in-the-middle attack (MITM) !




