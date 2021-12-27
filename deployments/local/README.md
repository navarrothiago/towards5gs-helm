# Deploy locally without multus in multi-node cluster

The deployment is based on the image below. 

![](https://github.com/Orange-OpenSource/towards5gs-helm/raw/main/pictures/Setup-free5gc-on-multiple-clusters-and-test-with-UERANSIM-Architecture.png)

However, we are not going to deploy multus. All the pods will have a static IP. Besides, all the pods will be deployed in one cluster (multi-nodes).

## Provision K8s using Vagrant

TODO: put the reference of the playground k8s.

```bash
vagrant up
```

**Taint**
If you want to set the taint
```
echo "[TASK 5] Remove taint from control-plane node"
kubectl taint nodes --all node-role.kubernetes.io/master:NoSchedule-
```

Install the gtp5g in each node:
```
vagrand ssh <node-id>
"sudo apt update && sudo apt install -y build-essential tcpdump iputils-ping mtr && \
sudo rm -R gtp5g || true && git clone -b v0.4.0 https://github.com/free5gc/gtp5g.git && \
cd gtp5g && make && sudo make install && mkdir /home/vagrant/kubedata || true"
```
:memo: The default name is master.k8s and node1.k8s.

## Install Calico

Install calico CNI:

```
kubectl apply -f ./calico.yaml
```

The file is based on `https://docs.projectcalico.org/manifests/calico.yaml`. The modification was done in `CALICO_IPV4POOL_IPIP` that was changed to `Never`.

## Install Multus

:warning: 
```
kubectl apply -f ./multus-daemonset-calico.yml

```

## Install Free5gc

Volume-Install PV:

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-local-pv9
  labels:
    project: free5gc
spec:
  capacity:
    storage: 8Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  local:
    path: /home/vagrant/kubedata
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node1
EOF
```

Install free5gc:
```
helm  install free5gc /work/mantisnet/towards5gs-helm/charts/free5gc/ \
--set global.n2network.masterIf=enp0s8 \
--set global.n3network.masterIf=enp0s8 \
--set global.n4network.masterIf=enp0s8 \
--set global.n6network.masterIf=enp0s8 \
--set global.n9network.masterIf=enp0s8 \
--set global.gke=true
```

## Install UERANSIM

```
helm  install ueransim /work/mantisnet/towards5gs-helm/charts/ueransim/ \
--set global.n2network.masterIf=enp0s8 \
--set global.n3network.masterIf=enp0s8 \
--set global.gke=true
```

## Clean up

```
helm uninstall free5gc
kubectl delete pv example-local-pv9 --grace-period=0 --force
kubectl patch pv example-local-pv9 -p '{"spec":{"claimRef": null}}'
```