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
      preferredDuringSchedulingIgnoredDuringExecution:
        preference:
          matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - minikube-m02
EOF
```

Wait for the VMs to start, checking periodically on their pods.

```shell
$ kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
virt-launcher-testvm1-78wq9   3/3     Running   0          15m
virt-launcher-testvm2-sq8mb   3/3     Running   0          15m
virt-launcher-testvm3-5gmqv   3/3     Running   0          14m
```

Check the ip given by our network definitions by checking the vm launcher.
```shell
$ kubectl describe pod virt-launcher-testvm1-78wq9
Name:             virt-launcher-testvm1-78wq9
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube-m02/192.168.49.3
Start Time:       Wed, 10 Apr 2024 13:55:00 -0400
Labels:           kubevirt.io=virt-launcher
                  kubevirt.io/created-by=54da0b99-0a59-4b99-9025-386dbdc61706
                  kubevirt.io/domain=testvm
                  kubevirt.io/nodeName=minikube-m02
                  kubevirt.io/size=small
                  vm.kubevirt.io/name=testvm1
Annotations:      cni.projectcalico.org/containerID: 274d05b78949d3b9b1f5d517007132d93302f8d237b73bb0c27729e65f11beed
                  cni.projectcalico.org/podIP: 10.244.205.203/32
                  cni.projectcalico.org/podIPs: 10.244.205.203/32
                  k8s.v1.cni.cncf.io/network-status:
                    [{
                        "name": "k8s-pod-network",
                        "ips": [
                            "10.244.205.203"
                        ],
                        "default": true,
                        "dns": {}
                    },{
                        "name": "default/vlan-network-1",
                        "interface": "net1",
                        "ips": [
                            "192.168.1.201"
                        ],
                        "mac": "7e:42:93:f5:74:76",
                        "dns": {},
                        "gateway": [
                            "\u003cnil\u003e"
                        ]
                    }]
                  k8s.v1.cni.cncf.io/networks: 
                  kubectl.kubernetes.io/default-container: compute
                  kubevirt.io/domain: testvm1
                  kubevirt.io/migrationTransportUnix: true
                  kubevirt.io/vm-generation: 1
                  post.hook.backup.velero.io/command: ["/usr/bin/virt-freezer", "--unfreeze", "--name", "testvm1", "--namespace", "default"]
                  post.hook.backup.velero.io/container: compute
                  pre.hook.backup.velero.io/command: ["/usr/bin/virt-freezer", "--freeze", "--name", "testvm1", "--namespace", "default"]
                  pre.hook.backup.velero.io/container: compute
                  traffic.sidecar.istio.io/kubevirtInterfaces: k6t-eth0
```

Verify limited connectivity by logging into a vm on each network and checking that it can reach the vm it should (if any) and can't reach the ones it can't.

`testvm1` won't be able to reach `testvm2` or `testvm3`
```shell
virtctl testvm1
Successfully connected to testvm1 console. The escape sequence is ^]
# login
$ ping -c 3 -w 10 192.168.2.201
PING 192.168.2.201 (192.168.2.201): 56 data bytes
--- 192.168.2.201 ping statistics ---
10 packets transmitted, 0 packets received, 100% packet loss
$ ping -c 3 -w 10 192.168.2.202
PING 192.168.2.202 (192.168.2.202): 56 data bytes
--- 192.168.2.202 ping statistics ---
10 packets transmitted, 0 packets received, 100% packet loss
```

`testvm2` should be able to reach `testvm3`, but not `testvm1`
```shell
$ virtctl console testvm2
Successfully connected to testvm2 console. The escape sequence is ^]
$ ping -c 3 -w 10 192.168.2.202
PING 192.168.2.202 (192.168.2.202): 56 data bytes
64 bytes from 192.168.2.202: seq=0 ttl=63 time=0.778 ms
64 bytes from 192.168.2.202: seq=1 ttl=63 time=0.806 ms
64 bytes from 192.168.2.202: seq=2 ttl=63 time=0.408 ms
--- 192.168.2.202 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.408/0.664/0.806 ms
$ ping -c 3 -w 10 192.168.1.201
PING 192.168.1.201 (192.168.1.201): 56 data bytes
--- 192.168.1.201 ping statistics ---
10 packets transmitted, 0 packets received, 100% packet loss
```


