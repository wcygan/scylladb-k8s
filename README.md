# ScyllaDB on Kubernetes

Reference: https://operator.docs.scylladb.com/stable/installation/helm.html

```bash
# Simply
skaffold run

# Or, run each module individually
skaffold run -m cert-manager-config
skaffold run -m scylla-operator-config
skaffold run -m scylla-config
skaffold run -m scylla-manager-config
```

Check for the pods to be ready:

```bash
kubectl -n scylla-operator get all
kubectl -n scylla get all
```

## Deployment Ordering

This is fairly important; deployment ordering is as follows:

1. Cert Manager
2. Scylla Operator
3. Scylla
4. Scylla Manager

## Issues encountered along the way

### Scylla API not deploying properly

It appears that the scylla-api pod is not deploying properly. The logs show the following:

```bash
Insufficient Disk Space:
- Error Message: ERROR:root:Filesystem at /var/lib/scylla/data has only 2977710080 bytes available; that is less than the recommended 10 GB. Please free up space and run scylla_io_setup again.
```

## Trying from Docs

- <https://www.scylladb.com/2021/05/11/using-helm-charts-to-deploy-scylla-on-kubernetes/>
- <https://operator.docs.scylladb.com/v1.13/generic.html>
