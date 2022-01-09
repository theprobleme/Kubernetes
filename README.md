# Kubernetes

## Objectifs

Pour la master : 
- On peut utiliser les images de conteneur déjà existantes. Pas besoin de repartir de la solution Docker.
- Au niveau du site web à héberger, on peut soit partir d'un wordpress soit on suit des tutoriels du site de Kubernetes et on envisage l'installation d'une application Node.js par exemple.

## Pré-requis

Il faut changer le CPU à 2 pour que la machine fonctionne.



## Installation

### Assurer que br_netfilter est chargé

`````bash
lsmod | grep br_netfilter
br_netfilter		32768	0
bridge				253952	1	br_netfilter
`````

### Modification des configuration sysctl

````bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
````

### Installation de kubectl

#### Installer le binaire de kubectl avec curl sur Linux

````bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
````

#### Rendre le binaire kubectl exécutable

````bash
chmod +x ./kubectl
````

#### Déplacement du binaire dans le PATH

````bash
sudo mv ./kubectl /usr/local/bin/kubectl
````

#### Test

````bash
kubectl version --client
Client Version: version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.0", GitCommit:"ab69524f795c42094a6630298ff53f3c3ebab7f4", GitTreeState:"clean", BuildDate:"2021-12-07T18:16:20Z", GoVersion:"go1.17.3", Compiler:"gc", Platform:"linux/amd64"}
````

### Installation de kubelet and kubeadm

````bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
````



## Création d'un Cluster a master unique avec kubeadm

### Retirer la swapp mod

````bash
swapoff -a
````

### Initialiser kubadmn

````bash
kubeadm init --prod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.4.94

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.4.94:6443 --token ocil0s.lv2cenw9i02no9b6 \
	--discovery-token-ca-cert-hash sha256:3bfa5152efef305183ccb64808a53a8756bcb0174fd1d8aeb6035e8f2b7b1209 
````

### Configurations

````bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf
````

### Installation de flannel

````bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
````

### Vérification

````bash
kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                             READY   STATUS    RESTARTS   AGE    IP             NODE          NOMINATED NODE   READINESS GATES
kube-system   coredns-78fcd69978-45qht         1/1     Running   0          101m   10.244.0.2     debian        <none>           <none>
kube-system   coredns-78fcd69978-h9mfq         1/1     Running   0          101m   10.244.0.3     debian        <none>           <none>
kube-system   etcd-debian                      1/1     Running   2          102m   192.168.4.94   debian        <none>           <none>
kube-system   kube-apiserver-debian            1/1     Running   2          102m   192.168.4.94   debian        <none>           <none>
kube-system   kube-controller-manager-debian   1/1     Running   0          102m   192.168.4.94   debian        <none>           <none>
kube-system   kube-flannel-ds-njzkg            1/1     Running   0          10m    192.168.4.93   node-paulin   <none>           <none>
kube-system   kube-flannel-ds-pv9lk            1/1     Running   0          98m    192.168.4.94   debian        <none>           <none>
kube-system   kube-flannel-ds-z49c4            1/1     Running   0          13m    192.168.4.91   node-boutin   <none>           <none>
kube-system   kube-proxy-4zqzl                 1/1     Running   0          101m   192.168.4.94   debian        <none>           <none>
kube-system   kube-proxy-hszbw                 1/1     Running   0          13m    192.168.4.91   node-boutin   <none>           <none>
kube-system   kube-proxy-vhvwj                 1/1     Running   0          10m    192.168.4.93   node-paulin   <none>           <none>
kube-system   kube-scheduler-debian            1/1     Running   5          102m   192.168.4.94   debian        <none>           <none>
````

Tous les pods affichés correspondent aux pods du namespace `kube-system`, soit ceux qui font tourner le cluster kubernetes. Si on exécute la commande `kubectl get pods`, on a aucun pods car il n'affiche que ceux qui sont créé soit-même :

````bash
kubectl get pods
No resources found in default namespace.
````



## Installation des nodes

### Connexion au noeud

````
kubeadm reset

kubeadm join 192.168.4.94:6443 --token crr1qv.fr1s5q2251upo9ql \
	--discovery-token-ca-cert-hash sha256:6d334d33b3c1922786389eabc01247e8bf6a8dc1e3ebde424422e8a11645af6a
````

### Changement de nom l'hostname

````
hostnamectl set-hostname node-boutin
````



## Installation du dashboard

### Ajout du dashboard

````bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended.yaml
````

### Création d'un compte administrateur

`````yaml
# dashboard-admin.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
`````

### Ajout de l'utilisateur

`````yaml
# dashboard-admin.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: read-only-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
  name: read-only-clusterrole
  namespace: default
rules:
- apiGroups:
  - ""
  resources: ["*"]
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - extensions
  resources: ["*"]
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - apps
  resources: ["*"]
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-only-binding
roleRef:
  kind: ClusterRole
  name: read-only-clusterrole
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: read-only-user
  namespace: kubernetes-dashboard
`````


### Récupération du token

````yaml
kubectl get secret -n kubernetes-dashboard $(kubectl get serviceaccount admin-user -n kubernetes-dashboard -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | base64 --decode

eyJhbGciOiJSUzI1NiIsImtpZCI6IlJTcnZEWUhtbVJTTS1rYjZPTTY4R2RJUWhkSXVOaGNVdEw2c3dpRjIxOG8ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTVqMmdiIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIyNjEzYTJlYy1mZjU4LTQ2YzEtYmQzNS01NjY0NTNiYzE3NTciLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.USuYRuGledT02eCLfohXGWzk0XnVDNoJySdMMzMZq32tShN5sIYpdXe1JM4DcToCTh2WcHDYUwYqCh4MxDItk0NLveYU2ebLCV0XMPvN-rPZgq5fZoNly_PbHuc78j_Q428npzrTK85VPIKDZDg0VgXK7TOy0XFu7yB0t_y8eDm8MeM8UJmWWm2aJFoX52oUHa9dcV1da3_3P_i9JVG-Q73plSjsb0lavE1SRicL59o1XliIoDf5laOJ1RjjjWUe3nd8KL1AySFyX-muN8kXX8WIlKA3U2PJhZMxTP5D3DynId2qf6jX5IdV0e5uZyvt-eBPByEypto6bfIy4p4WJ
````

### Création du tunnel ssh sur la machine hôte

````
ssh -L 8001:127.0.0.1:8001 master@192.168.23.110
````

### Lancement du proxy

````
kubectl proxy
````

### Accès au site

http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/persistentvolume?namespace=default

## Ajout de stockage sur le proxmox

`````
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   32G  0 disk 
├─sda1   8:1    0   31G  0 part /
├─sda2   8:2    0    1K  0 part 
└─sda5   8:5    0  975M  0 part [SWAP]
sdb      8:16   0   32G  0 disk 
sr0     11:0    1  377M  0 rom 
`````

```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   32G  0 disk 
├─sda1   8:1    0   31G  0 part /
├─sda2   8:2    0    1K  0 part 
└─sda5   8:5    0  975M  0 part [SWAP]
sdb      8:16   0   32G  0 disk 
sr0     11:0    1  377M  0 rom
```

Ajout de disque de 32G sur les 2 noeuds.

## OpenRBS

### Pré-requis

#### Installation de l'iSCSI initiator service

````
sudo apt-get update
sudo apt-get install open-iscsi
sudo systemctl enable --now iscsid
````

#### Vérifications

````
sudo cat /etc/iscsi/initiatorname.iscsi
## DO NOT EDIT OR REMOVE THIS FILE!
## If you remove this file, the iSCSI daemon will not start.
## If you change the InitiatorName, existing access control lists
## may reject this initiator.  The InitiatorName must be unique
## for each iSCSI initiator.  Do NOT duplicate iSCSI InitiatorNames.
InitiatorName=iqn.1993-08.org.debian:01:57849750d672
````

````
sudo systemctl enable --now iscsid
● iscsid.service - iSCSI initiator daemon (iscsid)
     Loaded: loaded (/lib/systemd/system/iscsid.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2022-01-07 09:54:58 CET; 42s ago
       Docs: man:iscsid(8)
   Main PID: 5211 (iscsid)
      Tasks: 2 (limit: 4663)
     Memory: 3.5M
        CPU: 8ms
     CGroup: /system.slice/iscsid.service
             ├─5210 /sbin/iscsid
             └─5211 /sbin/iscsid

janv. 07 09:54:58 node1 systemd[1]: Starting iSCSI initiator daemon (iscsid)...
janv. 07 09:54:58 node1 iscsid[5209]: iSCSI logger with pid=5210 started!
janv. 07 09:54:58 node1 systemd[1]: Started iSCSI initiator daemon (iscsid).
janv. 07 09:54:59 node1 iscsid[5210]: iSCSI daemon with pid=5211 started!
````

### Installation

#### OpenEBS

 ````
 kubectl apply -f https://openebs.github.io/charts/openebs-operator.yaml
 ````

#### Vérification

````
kubectl get pods -n openebs
openebs-localpv-provisioner-857f8dd5d6-tjh64    1/1     Running   0          53m
openebs-ndm-5pxk6                               1/1     Running   0          19m
openebs-ndm-7z5bx                               1/1     Running   0          23m
openebs-ndm-cluster-exporter-54b8b94955-qn8gf   1/1     Running   0          53m
openebs-ndm-node-exporter-jns4r                 1/1     Running   0          19m
openebs-ndm-node-exporter-qbtm8                 1/1     Running   0          23m
openebs-ndm-operator-5b578fd9c6-n4wlk           1/1     Running   0          53m
````

#### Liste des storage class

````
NAME               PROVISIONER        RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
openebs-device     openebs.io/local   Delete          WaitForFirstConsumer   false                  92m
openebs-hostpath   openebs.io/local   Delete          WaitForFirstConsumer   false                  92m
````

#### Ajout de cStor

```è
kubectl apply -f https://openebs.github.io/charts/cstor-operator.yaml
```

````
kubectl get pod -n openebs
NAME                                             READY   STATUS    RESTARTS   AGE
cspc-operator-64c67c894c-tmftl                   1/1     Running   0          2m50s
cvc-operator-5697fb984f-5kzmn                    1/1     Running   0          2m50s
openebs-cstor-admission-server-78898d4d6-2zt4p   1/1     Running   0          2m47s
openebs-cstor-csi-controller-0                   6/6     Running   0          2m51s
openebs-cstor-csi-node-6f5z4                     2/2     Running   0          2m51s
openebs-cstor-csi-node-npnz4                     2/2     Running   0          2m51s
openebs-localpv-provisioner-857f8dd5d6-tjh64     1/1     Running   0          3h32m
openebs-ndm-cluster-exporter-b5f8f4745-qbwf7     1/1     Running   0          2m23s
openebs-ndm-lrndf                                1/1     Running   0          2m
openebs-ndm-node-exporter-4frd2                  1/1     Running   0          2m
openebs-ndm-node-exporter-dzxw7                  1/1     Running   0          2m25s
openebs-ndm-operator-7769f77f8b-5xbcq            1/1     Running   0          2m29s
openebs-ndm-wqglh                                1/1     Running   0          2m28s
````

````
kubectl get blockdevices -n openebs
NAME                                           NODENAME   SIZE          CLAIMSTATE   STATUS   AGE
blockdevice-248fa5856da7b3878a387139061a2962   node2      34358672896   Unclaimed    Active   178m
blockdevice-d5bcbb9366e869a7613ce1f3d585f186   node1      34358672896   Unclaimed    Active   3h1m
````

`````yaml
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
 name: cstor-disk-pool
 namespace: openebs
spec:
 pools:
   - nodeSelector:
       kubernetes.io/hostname: "node1"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-d5bcbb9366e869a7613ce1f3d585f186"
     poolConfig:
       dataRaidGroupType: "stripe"

   - nodeSelector:
       kubernetes.io/hostname: "node2"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-248fa5856da7b3878a387139061a2962"
     poolConfig:
       dataRaidGroupType: "stripe"
`````

```
kubectl apply -f CStorPoolCluster.yaml
```

 ```
 kubectl get cspc -n openebs
 NAME              HEALTHYINSTANCES   PROVISIONEDINSTANCES   DESIREDINSTANCES   AGE
 cstor-disk-pool                      2                      2                  54s
 ```

````
kubectl get cspi -n openebs
NAME                   HOSTNAME   FREE     CAPACITY    READONLY   PROVISIONEDREPLICAS   HEALTHYREPLICAS   STATUS   AGE
cstor-disk-pool-97tb   node2      30800M   30800614k   false      0                     0                 ONLINE   2m27s
cstor-disk-pool-jt46   node1      30800M   30800614k   false      0                     0                 ONLINE   2m28s
````

````
apiVersion: storage.k8s.io/v1
metadata:
  name: cstor-csi-disk
provisioner: cstor.csi.openebs.io
allowVolumeExpansion: true
parameters:
  cas-type: cstor
  # cstorPoolCluster should have the name of the CSPC 
  cstorPoolCluster: cstor-disk-pool
  # replicaCount should be <= no. of CSPI created in the selected CSPC
  replicaCount: "2"
````

```
kubectl apply -f CStorStorageClasses
storageclass.storage.k8s.io/cstor-csi-disk created
```

```
kubectl get sc
NAME             PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
cstor-csi-disk   cstor.csi.openebs.io   Delete          Immediate           true                   57s
```

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: cstor-pvc
spec:
  storageClassName: cstor-csi-disk
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

```
kubectl apply -f pvc.yaml
persistentvolumeclaim/cstor-pvc created
```

````
kubectl get pvc
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
cstor-pvc   Bound    pvc-3e574ae7-ee25-4972-a280-b8ee6b800148   10Gi       RWO            cstor-csi-disk   49s
````
