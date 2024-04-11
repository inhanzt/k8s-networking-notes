Need to use Calico CNI for network policies to matter; couldn't get them working with flannel.

Create some network policies that will control network traffic.  
By default, all pods can communicate with all other pods, so the first policy is to disallow that.
```shell
cat <<EOF | kubectl apply -f -
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: deny-by-default
spec:
  podSelector: {}
  ingress: []
EOF
```

Next we want to allow certain traffic, which we accomplish with another network policy.  This one allows traffic by label.
```shell
cat <<EOF | kubectl apply -f -
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-by-label-my-label-by-value
spec:
  podSelector:
    matchLabels:
      my-label: my-value
  ingress:
    - from:
      - podSelector:
          matchLabels:
            my-label: my-value 
EOF
```

Now we create some vms, one without the label and two with it.
```shell
cat <<EOF | kubectl apply -f -
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
EOF
```

```shell
cat <<EOF | kubectl apply -f -
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
        my-label: my-value
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
EOF
```

```shell
cat <<EOF | kubectl apply -f -
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
        my-label: my-value
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

Check the ips of our vmis so we know which ones to ping.
```shell
$ kubectl get vmis
NAME      AGE    PHASE     IP               NODENAME       READY
testvm1   101s   Running   10.244.205.204   minikube-m02   True
testvm2   90s    Running   10.244.151.12    minikube-m03   True
testvm3   81s    Running   10.244.205.205   minikube-m02   True
```

Verify that `testvm1` can't access the others.
```shell
$ virtctl console testvm1
Successfully connected to testvm1 console. The escape sequence is ^]
login as 'cirros' user. default password: 'gocubsgo'. use 'sudo' for root.
testvm1 login: cirros
Password:
$ ping -c 3 -w 10 10.244.151.12
PING 10.244.151.12 (10.244.151.12): 56 data bytes
--- 10.244.151.12 ping statistics ---
10 packets transmitted, 0 packets received, 100% packet loss
$ ping -c 3 -w 10 10.244.205.205
PING 10.244.205.205 (10.244.205.205): 56 data bytes
--- 10.244.205.205 ping statistics ---
10 packets transmitted, 0 packets received, 100% packet loss
```

Verify that `testvm2` can access `testvm3`, but not `testvm1`.
```shell
$ virtctl console testvm2
Successfully connected to testvm2 console. The escape sequence is ^]
login as 'cirros' user. default password: 'gocubsgo'. use 'sudo' for root.
testvm2 login: cirros
Password: 
$ ping -c 3 -w 10 10.244.205.205
PING 10.244.205.205 (10.244.205.205): 56 data bytes
64 bytes from 10.244.205.205: seq=0 ttl=60 time=7.196 ms
64 bytes from 10.244.205.205: seq=1 ttl=60 time=4.213 ms
64 bytes from 10.244.205.205: seq=2 ttl=60 time=26.977 ms
--- 10.244.205.205 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 4.213/12.795/26.977 ms
$ ping -c 3 -w 10 10.244.205.204
PING 10.244.205.204 (10.244.205.204): 56 data bytes
--- 10.244.205.204 ping statistics ---
10 packets transmitted, 0 packets received, 100% packet loss
```

Cleanup if desired.
```shell
kubectl delete vm testvm1 testvm2 testvm3
kubectl delete networkpolicies deny-by-default allow-by-label-my-label-by-value
```
