# Configure Image Registry

Before start you should retrieve the following information:

- Nutanix Objects endpoint
- Nutanix Objects bucket
- Nutanix objects user Access key
- Nutanix Objects Secret key

1. Create Image registry secret

```bash
oc create secret generic image-registry-private-configuration-user \
    --from-literal=REGISTRY_STORAGE_S3_ACCESSKEY=MYKEY \
    --from-literal=REGISTRY_STORAGE_S3_SECRETKEY=MYSECRET \
    --namespace openshift-image-registry
```

2. Patch Image Registry

```bash
oc patch configs.imageregistry.operator.openshift.io/cluster \
    --type='json' \
    --patch='[
        {"op": "remove", "path": "/spec/storage" },
        {"op": "add", "path": "/spec/storage", "value": {"s3":{"bucket": "ocp2-image-registry", "regionEndpoint": "https://objects.dachlab.net", "encrypt": false, "region": "us-east-1"}}}
    ]'
```

3. Enable image Registry

```bash
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed"},{"replicas":"2"}}'
```