Kubevirt vms have a status section in the yaml to describe if they're running and ready.
```yaml
status:
  ...
  printableStatus: Running
  ready: true
  ...
```

Using these variables along with the entrypoint docker image filtering on the custom vm resource lets us create a blocking initcontainer, stopping one vm from starting without another reporting ready.
https://github.com/airshipit/kubernetes-entrypoint?tab=readme-ov-file#custom-resource
https://kubernetes.io/docs/concepts/workloads/pods/init-containers/

But kubevirt doesn't allow initContainers, so this won't work.
