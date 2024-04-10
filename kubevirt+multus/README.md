Create some network definitions.  These must exist both in the `kube-system` namespaceand the `default` namespace (or the namespace you're using).
```shell
cat <<EOF | kubectl create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: vlan-network-1
  namespace: kube-system
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "macvlan",
      "master": "eth0",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.1.0/24",
        "rangeStart": "192.168.1.200",
        "rangeEnd": "192.168.1.216",
        "routes": [
          { "dst": "0.0.0.0/0" }
        ],
        "gateway": "192.168.1.1"
      }
    }'
EOF
```

```shell
cat <<EOF | kubectl create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: vlan-network-2
  namespace: kube-system
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "macvlan",
      "master": "eth0",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.2.0/24",
        "rangeStart": "192.168.2.200",
        "rangeEnd": "192.168.2.216",
        "routes": [
          { "dst": "0.0.0.0/0" }
        ],
        "gateway": "192.168.2.1"
      }
    }'
EOF
```

```shell
cat <<EOF | kubectl create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: vlan-network-1
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "macvlan",
      "master": "eth0",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.1.0/24",
        "rangeStart": "192.168.1.200",
        "rangeEnd": "192.168.1.216",
        "routes": [
          { "dst": "0.0.0.0/0" }
        ],
        "gateway": "192.168.1.1"
      }
    }'
EOF
```

```shell
cat <<EOF | kubectl create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: vlan-network-2
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "macvlan",
      "master": "eth0",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.2.0/24",
        "rangeStart": "192.168.2.200",
        "rangeEnd": "192.168.2.216",
        "routes": [
          { "dst": "0.0.0.0/0" }
        ],
        "gateway": "192.168.2.1"
      }
    }'
EOF
```

Create some vms to test the networks with.
```shell
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: testvm1
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/size: small
        kubevirt.io/domain: testvm
      annotations:
        k8s.v1.cni.cncf.io/networks: vlan-network-1
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
          - name: default
            masquerade: {}
        resources:
          requests:
            memory: 64M
      networks:
      - name: default
        pod: {}
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/kubevirt/cirros-container-disk-demo
        - name: cloudinitdisk
          cloudInitNoCloud:
            userDataBase64: SGkuXG4=
```

```shell
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: testvm2
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/size: small
        kubevirt.io/domain: testvm
      annotations:
        k8s.v1.cni.cncf.io/networks: vlan-network-2
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
          - name: default
            masquerade: {}
        resources:
          requests:
            memory: 64M
      networks:
      - name: default
        pod: {}
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/kubevirt/cirros-container-disk-demo
        - name: cloudinitdisk
          cloudInitNoCloud:
            userDataBase64: SGkuXG4=
```

```shell
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: testvm3
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/size: small
        kubevirt.io/domain: testvm
      annotations:
        k8s.v1.cni.cncf.io/networks: vlan-network-2
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
          - name: default
            masquerade: {}
        resources:
          requests:
            memory: 64M
      networks:
      - name: default
        pod: {}
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/kubevirt/cirros-container-disk-demo
        - name: cloudinitdisk
          cloudInitNoCloud:
            userDataBase64: SGkuXG4=
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

```shell
virtctl testvm1
# login
```

```shell
$ ping   -c 3 -w 10 192.168.1.201
PING 192.168.1.201 (192.168.1.201): 56 data bytes
--- 192.168.1.201 ping statistics ---
5 packets transmitted, 0 packets received, 100% packet loss
```



