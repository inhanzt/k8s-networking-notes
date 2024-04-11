Create some network definitions.  These must exist both in the `kube-system` namespaceand the `default` namespace (or the namespace you're using).
```shell
cat <<EOF | kubectl create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: bridge-test-1
  namespace: kube-system
spec:
  config: '{
      "cniVersion": "0.3.1",
      "name": "bridge-test-1",
      "type": "bridge",
      "bridge": "br1",
      "ipam": {
        "type": "host-local",
        "subnet": "10.250.250.0/24"
      }
    }'
EOF
```

```shell
cat <<EOF | kubectl create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: bridge-test-1
spec:
  config: '{
      "cniVersion": "0.3.1",
      "name": "bridge-test-1",
      "type": "bridge",
      "bridge": "br1",
      "ipam": {
        "type": "host-local",
        "subnet": "10.250.250.0/24"
      }
    }'
EOF
```

```shell
cat <<EOF | kubectl create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: bridge-test-2
  namespace: kube-system
spec:
  config: '{
      "cniVersion": "0.3.1",
      "name": "bridge-test-2",
      "type": "bridge",
      "bridge": "br2",
      "ipam": {
        "type": "host-local",
        "subnet": "10.252.252.0/24"
      }
    }'
EOF
```

```shell
cat <<EOF | kubectl create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: bridge-test-2
spec:
  config: '{
      "cniVersion": "0.3.1",
      "name": "bridge-test-2",
      "type": "bridge",
      "bridge": "br2",
      "ipam": {
        "type": "host-local",
        "subnet": "10.252.252.0/24"
      }
    }'
EOF
```

Create some vms to test the networks with. 
The `step.template.spec.networks` section is determining which network a VM runs in.
The `spec.template.spec.preferredDuringSchedulingIgnoredDuringExecution` section is determining which node to run on.  The names of these nodes may need changed out to match your cluster. `kubectl get nodes` will provide your node names.  The goal is to get a pair of vms running on each node, one pair with matching networks and one pair without.
```shell
cat <<EOF | kubectl create -f -
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: testvm5
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/size: small
        kubevirt.io/domain: testvm
    spec:
      nodeSelector:
        kubernetes.io/hostname: minikube-m02
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
          - name: test1
            bridge: {}
        resources:
          requests:
            memory: 64M
      networks:
        - name: test1
          multus: # Multus network as default
            default: true
            networkName: bridge-test-1
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/kubevirt/cirros-container-disk-demo
        - name: cloudinitdisk
          cloudInitNoCloud:
            userDataBase64: SGkuXG4=
EOF
```

```shell
cat <<EOF | kubectl create -f -
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: testvm6
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/size: small
        kubevirt.io/domain: testvm
    spec:
      nodeSelector:
        kubernetes.io/hostname: minikube-m02
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
          - name: test1
            bridge: {}
        resources:
          requests:
            memory: 64M
      networks:
        - name: test1
          multus: # Multus network as default
            default: true
            networkName: bridge-test-1
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/kubevirt/cirros-container-disk-demo
        - name: cloudinitdisk
          cloudInitNoCloud:
            userDataBase64: SGkuXG4=
EOF
```

```shell
cat <<EOF | kubectl create -f -
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: testvm7
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/size: small
        kubevirt.io/domain: testvm
    spec:
      nodeSelector:
        kubernetes.io/hostname: minikube-m03
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
          - name: test1
            bridge: {}
        resources:
          requests:
            memory: 64M
      networks:
        - name: test1
          multus: # Multus network as default
            default: true
            networkName: bridge-test-1
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/kubevirt/cirros-container-disk-demo
        - name: cloudinitdisk
          cloudInitNoCloud:
            userDataBase64: SGkuXG4=
EOF
```


```shell
cat <<EOF | kubectl create -f -
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: testvm8
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/size: small
        kubevirt.io/domain: testvm
    spec:
      nodeSelector:
        kubernetes.io/hostname: minikube-m03
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
          - name: test1
            bridge: {}
        resources:
          requests:
            memory: 64M
      networks:
        - name: test1
          multus: # Multus network as default
            default: true
            networkName: bridge-test-2
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/kubevirt/cirros-container-disk-demo
        - name: cloudinitdisk
          cloudInitNoCloud:
            userDataBase64: SGkuXG4=
EOF
```


Wait for the VMIs to start, checking on them occasionally.
You may notice that testvm5 and testvm 7 have idential IPs.  That's expected, as bridge mode only allows intra-node communication, so the IPs are actually on separate networks.
```shell
$ kubectl get vmis
NAME      AGE   PHASE     IP             NODENAME       READY
testvm5   23m   Running   10.250.250.2   minikube-m02   True
testvm6   20m   Running   10.250.250.3   minikube-m02   True
testvm7   19m   Running   10.250.250.2   minikube-m03   True
testvm8   18m   Running   10.252.252.2   minikube-m03   True
```

Verify that `testvm5` and `testvm6` can communicate, and that `testvm5` cannot reach `testvm8`.
```shell
$ virtctl console testvm5
Successfully connected to testvm5 console. The escape sequence is ^]
login as 'cirros' user. default password: 'gocubsgo'. use 'sudo' for root.
testvm5 login: cirros
Password: 
$ ping 10.250.250.3 -c 3 -w 10
PING 10.250.250.3 (10.250.250.3): 56 data bytes
64 bytes from 10.250.250.3: seq=0 ttl=64 time=31.129 ms
64 bytes from 10.250.250.3: seq=1 ttl=64 time=2.951 ms
64 bytes from 10.250.250.3: seq=2 ttl=64 time=2.395 ms
--- 10.250.250.3 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 2.395/12.158/31.129 ms
$ ping 10.252.252.2 -c 3 -w 10
PING 10.252.252.2 (10.252.252.2): 56 data bytes
ping: sendto: Network is unreachable
```

Verify that `testvm7` and `testvm8` cannot communicate.  Note that this simultaneously proves `testvm7` cannot reach `testvm5`.
```shell
$ virtctl console testvm7
Successfully connected to testvm7 console. The escape sequence is ^]
login as 'cirros' user. default password: 'gocubsgo'. use 'sudo' for root.
testvm7 login: cirros
Password: 
$ ping -c 3 -w 10 10.252.252.2
PING 10.252.252.2 (10.252.252.2): 56 data bytes
ping: sendto: Network is unreachable
```



