Need to use Calico CNI for network policies to matter; couldn't get them working with flannel.

Update file watch limits (KubeVirt pods error out otherwise)
```
sysctl fs.inotify.max_user_instances=1280
sysctl fs.inotify.max_user_watches=655360
```

Configure KUBECONFIG so minikube knows the cluster config.
https://devops.stackexchange.com/a/16044
```
export KUBECONFIG=~/.kube/config
mkdir ~/.kube 2> /dev/null
kubectl config view --raw > "$KUBECONFIG"
chmod 600 "$KUBECONFIG"
```
Optionally, edit ~/.bashrc to include `export KUBECONFIG=~/.kube/config` for persistance

Run KubeVirt on the cluster.
export VERSION=$(curl -s https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
kubectl create -f https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-operator.yaml
kubectl create -f https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-cr.yaml


Allow the cluster to fully start before proceeding. You want everything to be N/N (e.g. 1/1, 2/2, 3/3). 
kubectl get kubevirt.kubevirt.io/kubevirt -n kubevirt
kubectl get all -n kubevirt

Check that your system supports hardware virtualization.
```shell
egrep -c '(vmx|svm)' /proc/cpuinfo
```

If this command outputs 0, it doesn't support hardware virtualization.  Run the command below to switch to emulated virtualization.

```shell
kubectl -n kubevirt patch kubevirt kubevirt --type=merge --patch '{"spec":{"configuration":{"developerConfiguration":{"useEmulation":true}}}}'
```

Install virtctl, a command line tool for manipulating kubevirt.
```
export VERSION=$(curl -s https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
export ARCH=$(uname -s | tr A-Z a-z)-$(uname -m | sed 's/x86_64/amd64/') || windows-amd64.exe
echo ${ARCH}
curl -L -o virtctl https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/virtctl-${VERSION}-${ARCH}
chmod +x virtctl
sudo install virtctl /usr/local/bin
```

