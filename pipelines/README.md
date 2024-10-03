## Instructions

Add the needed registry credentials to podman 
```bash
podman login -u='ch007m+dabou' -p='KTT2EFO...4SGJN7WVHGGP' quay.io/ch007m
```

and create the secret file within the target namespace

```bash
kubectl create secret generic dockerconfig-secret --from-file=config.json=$HOME/.config/containers/auth.json -n demo
```