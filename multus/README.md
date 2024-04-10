# k8s-networking-notes

Grab Multus and make a couple of networks.

```shell
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset-thick.yml
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
        "gateway": "192.168.1.1",
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

Make some pods that use those networks.

```shell
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: samplepod1
  annotations:
    k8s.v1.cni.cncf.io/networks: vlan-network-1
spec:
  containers:
  - name: samplepod
    command: ["/bin/ash", "-c", "trap : TERM INT; sleep infinity & wait"]
    image: alpine
EOF
```

```shell
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: samplepod2
  annotations:
    k8s.v1.cni.cncf.io/networks: vlan-network-2
spec:
  containers:
  - name: samplepod
    command: ["/bin/ash", "-c", "trap : TERM INT; sleep infinity & wait"]
    image: alpine
EOF
```

```shell
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: samplepod3
  annotations:
    k8s.v1.cni.cncf.io/networks: vlan-network-2
spec:
  containers:
  - name: samplepod
    command: ["/bin/ash", "-c", "trap : TERM INT; sleep infinity & wait"]
    image: alpine
EOF
```

Verify that the our network attachment definitions exist and that pods are running.
```shell
$ kubectl get network-attachment-definitions
NAME             AGE
vlan-network-1   5d21h
vlan-network-2   5d21h
$ kubectl get pods
NAME         READY   STATUS       RESTARTS   AGE
samplepod1   1/1     Running      0          4m9s
samplepod2   1/1     Running      0          3m57s
samplepod3   1/1     Running      0          3m45s
```

Check the status of a pod.  The annotations section defines the networks the pod is a member of.
```shell
$ kubectl describe pod samplepod2
Name:             samplepod2
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube-m02/192.168.49.3
Start Time:       Wed, 10 Apr 2024 09:18:22 -0400
Labels:           <none>
Annotations:      cni.projectcalico.org/containerID: 173a1262a4e4d56c3da4e3cd0a60ad8eac1736a291abbfd86c1b9ff66009e625
                  cni.projectcalico.org/podIP: 10.244.205.201/32
                  cni.projectcalico.org/podIPs: 10.244.205.201/32
                  k8s.v1.cni.cncf.io/network-status:
                    [{
                        "name": "k8s-pod-network",
                        "ips": [
                            "10.244.205.201"
                        ],
                        "default": true,
                        "dns": {}
                    },{
                        "name": "default/vlan-network-2",
                        "interface": "net1",
                        "ips": [
                            "192.168.2.202"
                        ],
                        "mac": "02:42:c0:a8:31:03",
                        "dns": {},
                        "gateway": [
                            "\u003cnil\u003e"
                        ]
                    }]
                  k8s.v1.cni.cncf.io/networks: vlan-network-2
...

Check network connectivity between pods on the same network.  In this case, the IP comes from `vlan-network-2` in the `describe samplepod2` command.

```shell
$ kubectl exec -it samplepod3 -- ping -c 3 -w 10 192.168.2.203
PING 192.168.2.203 (192.168.2.203): 56 data bytes
64 bytes from 192.168.2.203: seq=0 ttl=64 time=0.187 ms
64 bytes from 192.168.2.203: seq=1 ttl=64 time=0.089 ms
64 bytes from 192.168.2.203: seq=2 ttl=64 time=0.068 ms
--- 192.168.2.203 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss

round-trip min/avg/max = 0.068/0.114/0.187 ms


