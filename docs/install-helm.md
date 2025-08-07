# Install with Helm

[back](../README.md)

```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts

helm upgrade -i velero vmware-tanzu/velero \
  --create-namespace \
  -n velero \
  --set-file credentials.secretContents.cloud=/tmp/credentials-velero \
  -f helm/values.yaml
  --version 8.3.0
```

## REFERENCE

- [Chart Readme](https://github.com/vmware-tanzu/helm-charts/blob/main/charts/velero/README.md)
