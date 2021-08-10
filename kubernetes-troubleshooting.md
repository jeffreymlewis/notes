# Kubernetes Troubleshooting
Here are some random notes on Kubernetes troubleshooting.

## Get Events
Today I was troubleshooting the following error. This came about when terraform hit a timeout on a `helm_release` resource. The error message is non-descript.

```
Warning: Helm release "gloo" was created but has a failed status. Use the `helm` command to investigate the error, correct it, then run Terraform again.
```

Unfortunately helm doesn't save any log output, so you need to dig into kubernetes to find out what happened. The `kubectl get events` can help us here. It returns a list of events similar to what you'd see with the `kubectl describe ...` command.

```
kubectl -n namespace get events --sort-by='{.lastTimestamp}'
kubectl -n namespace get events --sort-by=.metadata.creationTimestamp
```
