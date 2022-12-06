# Despliegue-de-cluster-de-kubernetes-con-la-herramienta-kubespray
En el siguiente documento se realizara la instalacion de kubespray la cual es muy usada en ambiente industrial, esta es montada sobre la VM en la cual se despliega


# Herramientas necesarias

Antes de proceder con la instalacion se necesitan de unas herramientas previas las cuales son:

- [helm-install](https://github.com/mgodll/Despliegue-y-gestion-de-contenedores-usando-la-herramienta-helm)
- [Kubectl](https://github.com/mgodll/Despliegue-y-gestion-de-contenedores-usando-la-herramienta-helm)
- [Docker](https://github.com/mgodll/Despliegue-y-gestion-de-contenedores-usando-la-herramienta-helm)

# Requisitos de maquina

Requisitos minimos para la instalacion

| Estructura       | Valores            |
|------------------|--------------------|
|OS usado          | Ubuntu 20.04 LTS   |
|vCPU              | 8                  |
|RAM (GB)          | 16                 |
|Disk (GB)         | 140                |
|Home user         | mateo              |
|Number of NICs    | 2 (ens160, ens192) |
|Kubespray Version | v2.19.1            |



> **NOTA** se recomienda habilitar dos interfaces de red para vincular la maquina a un PVC o un volumen

# Exportacion de variables de entorno

Se exportaran unas variables para hacer el montaje mas comodo

```bash
export NODE_USER="mtoror"
export NODE_HOSTNAME_PREFIX="mateo"
export NODE_IP="10.182.0.10"
export KUBESPRAY_TAG="v2.19.1"
export CLUSTER_NAME="idt-cluster"
```

> **NOTA** se recomienda usar los siguientes comandos para saber el NODE_USER (whoami) y NODE_HOSTNAME_PREFIX (hostname), el node ip es la ip con acceso a internet de la maquina

# Configurando servidor

Como primer paso se actualizara la interfaz y se realizara una modificacion de privilegios

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install vim
sudo visudo
```

Siguiente se abrira una ventana y se anexara

```bash
# User privilege specification
root    ALL=(ALL:ALL) ALL
+ $NODE_USER  ALL=(ALL) NOPASSWD:ALL

# Members of the admin group may gain root privileges
%admin ALL=(ALL) ALL
```

> **NOTA** Para todo los procesos a continuacion se recomiendo estar dentro de super usuario con el comando `sudo su` para tener privilegios de `root`

Para crear una key con auto-acces en el servisor se asignaran un usuario y contraseña de la siguiente manera

```bash
ssh-keygen
# Generating public/private rsa key pair.
# Enter file in which to save the key (/root/.ssh/id_rsa):
# Enter passphrase (empty for no passphrase):
# Enter same passphrase again:
# Your identification has been saved in /root/.ssh/id_rsa.
# Your public key has been saved in /root/.ssh/id_rsa.pub.
# The key fingerprint is:
# SHA256:t0qr5YF1pg9GqE2MvwVkjzamiNYUIvLUsc/KX47znis root@kubernetes
# The key's randomart image is:
# +---[RSA 2048]----+
# |    .            |
# |   . o           |
# |o...o o          |
# |oo. .B +         |
# |  ... % S +      |
# | .oo X * = .     |
# |....= + X .      |
# |.    .E@ B       |
# |      =*X..      |
# +----[SHA256]-----+
ssh-copy-id -i ~/.ssh/id_rsa $NODE_USER@$NODE_IP
# /usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
# /usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
# /usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
# ubuntu@10.1.3.152's password:
# Number of key(s) added: 1
# 
# Now try logging into the machine, with:   "ssh 'ubuntu@10.1.3.152'"
# and check to make sure that only the key(s) you wanted were added.
```

> **NOTA** si en llegado caso al tirar el comando `ssh-copy-id -i ~/.ssh/id_rsa $NODE_USER@$NODE_IP` retorna `$NODE_USER@localhost: Permission denied (publickey)`, se recomienda ingresar lo siguiente a la terminar con el fin de copiar manual las key `cat ~/.ssh/id_rsa.pub >> /home/$NODE_USER/.ssh/authorized_keys`

ahora se accede al cluster o se inicia sesion en el servidor creado

```bash
ssh $NODE_USER@$NODE_IP
```

> **NOTA** En este paso al acceder a la maquina nos sacara de super usuario, volver acceder y seguin con el proceso como root

Se instala Python como requisito para la maquina

```bash
sudo apt install python3-pip
sudo pip3 install --upgrade pip
```

Clonar el repositorio de Kuberspray

```bash
# Clone Kubespray repository
cd && git clone https://github.com/kubernetes-sigs/kubespray.git ~/kubespray
cd ~/kubespray && git checkout tags/$KUBESPRAY_TAG -b $KUBESPRAY_TAG
# Install dependencies from requirements.txt
sudo pip3 install -r requirements.txt
# Copy inventory/sample as inventory/$CLUSTER_NAME
cp -rfp inventory/sample inventory/$CLUSTER_NAME
# declare IPs of the ubuntu nodes
declare -a IPS=($NODE_IP)
# Update the ansible inventory with the variables related to the nodes
HOST_PREFIX=$NODE_HOSTNAME_PREFIX CONFIG_FILE=inventory/$CLUSTER_NAME/hosts.yml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```

> **NOTA** En la parte de `${IPS[@]}` no tiene necesidad de reemplazar la direccion ip

Se creara un .yaml con el fin de desplegar la maquina y darle la informacion necesaria para ello se ingresa lo siguiente

```bash
sudo vim ~/kubespray/inventory/$CLUSTER_NAME/deploy_ansible_vars.yaml
```

Y anexamos lo siguiente

```bash
# Ansible variables
ansible_user: $NODE_USER
bootstrap_os: $NODE_USER

# App variables
helm_enabled: true
kubevirt_enabled: false
local_volume_provisioner_enabled: true

# Networking configuration
kube_network_plugin: calico
kube_network_plugin_multus: true
multus_version: stable
kube_pods_subnet: 10.233.64.0/18
kube_apiserver_port: 6443
kube_network_node_prefix: 24
kube_service_addresses: 10.233.0.0/18
kube_proxy_mode: ipvs
kube_apiserver_node_port_range: 30000-36767

# Kubernetes configuration
kubeadm_control_plane: true
etcd_kubeadm_enabled: true
kube_config_dir: /etc/kubernetes
kube_version: v1.23.7
dashboard_enabled: true

# cGroup configuration
docker_cgroup_driver: systemd
kubelet_cgroup_driver: systemd

# DNS configuration
dns_mode: coredns
resolvconf_mode: host_resolvconf
dns_domain: cluster.local
enable_nodelocaldns: true
nameservers:
  - 8.8.8.8
  - 8.8.4.4
upstream_dns_servers:
  - 8.8.8.8
  - 8.8.4.4

# Swap configuration
disable_swap: true
```

verificamos haciendo un cat a la configuracion del .yaml `cat ~/kubespray/inventory/$CLUSTER_NAME/hosts.yml`

```bash
all:
  hosts:
    kubernetes1:
      ansible_host: 10.1.3.152
      ip: 10.1.3.152
      access_ip: 10.1.3.152
  children:
    kube_control_plane:
      hosts:
        kubernetes1:
    kube_node:
      hosts:
        kubernetes1:
    etcd:
      hosts:
        kubernetes1:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
```

# Despliegue del cluster

Como ultimo paso se desplegara el cluster de kubernetes usando

```bash
cd ~/kubespray
ansible-playbook -b -i inventory/$CLUSTER_NAME/hosts.yml cluster.yml --user $NODE_USER -e @inventory/$CLUSTER_NAME/deploy_ansible_vars.yaml
```

Se espera a que el proceso termine, normalmente suele durar entre 10 a 15 minutos

> **NOTA** Para los siguientes pasos se sale del root, con el comando exit

Se configura el archivo Kubeconfig para el servidor con las siguientes lineas

```bash
# In the server:
mkdir -p ~/.kube
sudo cat /root/.kube/config > ~/.kube/config
sudo chown $NODE_USER:$NODE_USER ~/.kube/config
sudo chmod go-r ~/.kube/config
```

Y por ultimo se verifica que el cluster de kubernetes este en linea con kubectl

```bash
kubectl get pods -n kube-system
NAME                                          READY   STATUS             RESTARTS        AGE
calico-kube-controllers-6dd874f784-klbvw      1/1     Running            1 (3m52s ago)   3m54s
calico-node-xc6zt                             1/1     Running            0               4m31s
coredns-76b4fb4578-q48sh                      1/1     Running            0               3m22s
dns-autoscaler-7874cf6bcf-ddhws               1/1     Running            0               3m18s
etcd-pruebas1                                 1/1     Running            0               5m31s
kube-apiserver-pruebas1                       1/1     Running            1               5m31s
kube-controller-manager-pruebas1              1/1     Running            1               5m31s
kube-multus-ds-amd64-52k97                    0/1     CrashLoopBackOff   7 (43s ago)     3m58s
kube-proxy-5cwjd                              1/1     Running            0               4m34s
kube-scheduler-pruebas1                       1/1     Running            1               5m31s
kubernetes-dashboard-584bfbb648-8rgz2         1/1     Running            0               3m14s
kubernetes-metrics-scraper-5dc755864d-nx2vz   1/1     Running            0               3m14s
local-volume-provisioner-2j8x6                1/1     Running            0               3m39s
nodelocaldns-6chf2                            1/1     Running            0               3m17s
```
# Correccion de CrashLoopBackOff kube-multus-ds-amd64-52k97

Como se observar en los servicios desplegados del cluster el multus presenta un crash por lo tanto para solucionar este problema se mirara los servicios del daemonset con el siguiente comando

```bash
kubectl get daemonset -n kube-system
NAME                       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR              AGE
calico-node                1         1         1       1            1           kubernetes.io/os=linux     24m
kube-multus-ds-amd64       1         1         0       1            0           kubernetes.io/arch=amd64   23m
kube-proxy                 1         1         1       1            1           kubernetes.io/os=linux     25m
local-volume-provisioner   1         1         1       1            1           <none>                     23m
nodelocaldns               1         1         1       1            1           kubernetes.io/os=linux     23m
```

Tambien se puede usar `kubectl get daemonset -A`, por lo tanto se ve que el servicio tiene como nombre `kube-multus-ds-amd64`, para borrarlo se digita el siguiente comando

```bash
kubectl delete daemonset kube-multus-ds-amd64 -n kube-system
```

Para volver a correr el multus se crean dos archivos .yml `multus-daemonset-crio.yml` y `multus-daemonset.yml`, por lo tanto se creara una carpeta con el nombre de multus

> **NOTA** Se recomienda consultar informacion de los siguientes repositorios [multus-cni](https://github.com/k8snetworkplumbingwg/multus-cni/tree/master/deployments) y [Change cni version to 0.4.0](https://github.com/k8snetworkplumbingwg/multus-cni/issues/738)

```bash
mkdir multus
cd multus
vim multus-daemonset-crio.yml
```
se pegara lo siguiente

```bash
# Note:
#   This deployment file is designed for 'quickstart' of multus, easy installation to test it,
#   hence this deployment yaml does not care about following things intentionally.
#     - various configuration options
#     - minor deployment scenario
#     - upgrade/update/uninstall scenario
#   Multus team understand users deployment scenarios are diverse, hence we do not cover
#   comprehensive deployment scenario. We expect that it is covered by each platform deployment.
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: network-attachment-definitions.k8s.cni.cncf.io
spec:
  group: k8s.cni.cncf.io
  scope: Namespaced
  names:
    plural: network-attachment-definitions
    singular: network-attachment-definition
    kind: NetworkAttachmentDefinition
    shortNames:
    - net-attach-def
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          description: 'NetworkAttachmentDefinition is a CRD schema specified by the Network Plumbing
            Working Group to express the intent for attaching pods to one or more logical or physical
            networks. More information available at: https://github.com/k8snetworkplumbingwg/multi-net-spec'
          type: object
          properties:
            apiVersion:
              description: 'APIVersion defines the versioned schema of this represen
                tation of an object. Servers should convert recognized schemas to the
                latest internal value, and may reject unrecognized values. More info:
                https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources'
              type: string
            kind:
              description: 'Kind is a string value representing the REST resource this
                object represents. Servers may infer this from the endpoint the client
                submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
              type: string
            metadata:
              type: object
            spec:
              description: 'NetworkAttachmentDefinition spec defines the desired state of a network attachment'
              type: object
              properties:
                config:
                  description: 'NetworkAttachmentDefinition config is a JSON-formatted CNI configuration'
                  type: string
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: multus
rules:
  - apiGroups: ["k8s.cni.cncf.io"]
    resources:
      - '*'
    verbs:
      - '*'
  - apiGroups:
      - ""
    resources:
      - pods
      - pods/status
    verbs:
      - get
      - update
  - apiGroups:
      - ""
      - events.k8s.io
    resources:
      - events
    verbs:
      - create
      - patch
      - update
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: multus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: multus
subjects:
- kind: ServiceAccount
  name: multus
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: multus
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: multus-cni-config
  namespace: kube-system
  labels:
    tier: node
    app: multus
data:
  # NOTE: If you'd prefer to manually apply a configuration file, you may create one here.
  # In the case you'd like to customize the Multus installation, you should change the arguments to the Multus pod
  # change the "args" line below from
  # - "--multus-conf-file=auto"
  # to:
  # "--multus-conf-file=/tmp/multus-conf/70-multus.conf"
  # Additionally -- you should ensure that the name "70-multus.conf" is the alphabetically first name in the
  # /etc/cni/net.d/ directory on each node, otherwise, it will not be used by the Kubelet.
  cni-conf.json: |
    {
      "name": "multus-cni-network",
      "type": "multus",
      "capabilities": {
        "portMappings": true
      },
      "delegates": [
        {
          "cniVersion": "0.3.1",
          "name": "default-cni-network",
          "plugins": [
            {
              "type": "flannel",
              "name": "flannel.1",
                "delegate": {
                  "isDefaultGateway": true,
                  "hairpinMode": true
                }
              },
              {
                "type": "portmap",
                "capabilities": {
                  "portMappings": true
                }
              }
          ]
        }
      ],
      "kubeconfig": "/etc/cni/net.d/multus.d/multus.kubeconfig"
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-multus-ds
  namespace: kube-system
  labels:
    tier: node
    app: multus
    name: multus
spec:
  selector:
    matchLabels:
      name: multus
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        tier: node
        app: multus
        name: multus
    spec:
      hostNetwork: true
      tolerations:
      - operator: Exists
        effect: NoSchedule
      - operator: Exists
        effect: NoExecute
      serviceAccountName: multus
      containers:
      - name: kube-multus
        # crio support requires multus:latest for now. support 3.3 or later.
        image: ghcr.io/k8snetworkplumbingwg/multus-cni:stable
        command: ["/entrypoint.sh"]
        args:
        - "--cni-version=0.3.1"
        - "--cni-bin-dir=/host/usr/libexec/cni"
        - "--multus-conf-file=auto"
        - "--restart-crio=true"
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: true
          capabilities:
            add: ["SYS_ADMIN"]
        volumeMounts:
        - name: run
          mountPath: /run
          mountPropagation: HostToContainer
        - name: cni
          mountPath: /host/etc/cni/net.d
        - name: cnibin
          mountPath: /host/usr/libexec/cni
        - name: multus-cfg
          mountPath: /tmp/multus-conf
      terminationGracePeriodSeconds: 10
      volumes:
        - name: run
          hostPath:
            path: /run
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: cnibin
          hostPath:
            path: /usr/libexec/cni
        - name: multus-cfg
          configMap:
            name: multus-cni-config
            items:
            - key: cni-conf.json
              path: 70-multus.conf
```

para el kube-multus-ds-amd64 tambien se digitara `vim multus-daemonset.yml` y se pegara lo siguiente

```bash
# Note:
#   This deployment file is designed for 'quickstart' of multus, easy installation to test it,
#   hence this deployment yaml does not care about following things intentionally.
#     - various configuration options
#     - minor deployment scenario
#     - upgrade/update/uninstall scenario
#   Multus team understand users deployment scenarios are diverse, hence we do not cover
#   comprehensive deployment scenario. We expect that it is covered by each platform deployment.
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: network-attachment-definitions.k8s.cni.cncf.io
spec:
  group: k8s.cni.cncf.io
  scope: Namespaced
  names:
    plural: network-attachment-definitions
    singular: network-attachment-definition
    kind: NetworkAttachmentDefinition
    shortNames:
    - net-attach-def
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          description: 'NetworkAttachmentDefinition is a CRD schema specified by the Network Plumbing
            Working Group to express the intent for attaching pods to one or more logical or physical
            networks. More information available at: https://github.com/k8snetworkplumbingwg/multi-net-spec'
          type: object
          properties:
            apiVersion:
              description: 'APIVersion defines the versioned schema of this represen
                tation of an object. Servers should convert recognized schemas to the
                latest internal value, and may reject unrecognized values. More info:
                https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources'
              type: string
            kind:
              description: 'Kind is a string value representing the REST resource this
                object represents. Servers may infer this from the endpoint the client
                submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
              type: string
            metadata:
              type: object
            spec:
              description: 'NetworkAttachmentDefinition spec defines the desired state of a network attachment'
              type: object
              properties:
                config:
                  description: 'NetworkAttachmentDefinition config is a JSON-formatted CNI configuration'
                  type: string
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: multus
rules:
  - apiGroups: ["k8s.cni.cncf.io"]
    resources:
      - '*'
    verbs:
      - '*'
  - apiGroups:
      - ""
    resources:
      - pods
      - pods/status
    verbs:
      - get
      - update
  - apiGroups:
      - ""
      - events.k8s.io
    resources:
      - events
    verbs:
      - create
      - patch
      - update
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: multus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: multus
subjects:
- kind: ServiceAccount
  name: multus
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: multus
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: multus-cni-config
  namespace: kube-system
  labels:
    tier: node
    app: multus
data:
  # NOTE: If you'd prefer to manually apply a configuration file, you may create one here.
  # In the case you'd like to customize the Multus installation, you should change the arguments to the Multus pod
  # change the "args" line below from
  # - "--multus-conf-file=auto"
  # to:
  # "--multus-conf-file=/tmp/multus-conf/70-multus.conf"
  # Additionally -- you should ensure that the name "70-multus.conf" is the alphabetically first name in the
  # /etc/cni/net.d/ directory on each node, otherwise, it will not be used by the Kubelet.
  cni-conf.json: |
    {
      "name": "multus-cni-network",
      "type": "multus",
      "capabilities": {
        "portMappings": true
      },
      "delegates": [
        {
          "cniVersion": "0.3.1",
          "name": "default-cni-network",
          "plugins": [
            {
              "type": "flannel",
              "name": "flannel.1",
                "delegate": {
                  "isDefaultGateway": true,
                  "hairpinMode": true
                }
              },
              {
                "type": "portmap",
                "capabilities": {
                  "portMappings": true
                }
              }
          ]
        }
      ],
      "kubeconfig": "/etc/cni/net.d/multus.d/multus.kubeconfig"
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-multus-ds
  namespace: kube-system
  labels:
    tier: node
    app: multus
    name: multus
spec:
  selector:
    matchLabels:
      name: multus
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        tier: node
        app: multus
        name: multus
    spec:
      hostNetwork: true
      tolerations:
      - operator: Exists
        effect: NoSchedule
      - operator: Exists
        effect: NoExecute
      serviceAccountName: multus
      containers:
      - name: kube-multus
        image: ghcr.io/k8snetworkplumbingwg/multus-cni:stable
        command: ["/entrypoint.sh"]
        args:
        - "--multus-conf-file=auto"
        - "--cni-version=0.3.1"
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: true
        volumeMounts:
        - name: cni
          mountPath: /host/etc/cni/net.d
        - name: cnibin
          mountPath: /host/opt/cni/bin
        - name: multus-cfg
          mountPath: /tmp/multus-conf
      initContainers:
        - name: install-multus-binary
          image: ghcr.io/k8snetworkplumbingwg/multus-cni:stable
          command:
            - "cp"
            - "/usr/src/multus-cni/bin/multus"
            - "/host/opt/cni/bin/multus"
          resources:
            requests:
              cpu: "10m"
              memory: "15Mi"
          securityContext:
            privileged: true
          volumeMounts:
            - name: cnibin
              mountPath: /host/opt/cni/bin
              mountPropagation: Bidirectional
      terminationGracePeriodSeconds: 10
      volumes:
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: cnibin
          hostPath:
            path: /opt/cni/bin
        - name: multus-cfg
          configMap:
            name: multus-cni-config
            items:
            - key: cni-conf.json
              path: 70-multus.conf
```

Se lanzan los archivos de la siguiente manera

```bash
kubectl -n kube-system apply -f ./multus-daemonset-crio.yml
kubectl -n kube-system apply -f ./multus-daemonset.yml
```

Por ultimo se verifica que el servicio de multus este funcionando

```bash
kubectl get pods -n kube-system
NAME                                          READY   STATUS    RESTARTS      AGE
calico-kube-controllers-6dd874f784-klbvw      1/1     Running   1 (64m ago)   64m
calico-node-xc6zt                             1/1     Running   0             64m
coredns-76b4fb4578-q48sh                      1/1     Running   0             63m
dns-autoscaler-7874cf6bcf-ddhws               1/1     Running   0             63m
etcd-pruebas1                                 1/1     Running   0             65m
kube-apiserver-pruebas1                       1/1     Running   1             65m
kube-controller-manager-pruebas1              1/1     Running   1             65m
kube-multus-ds-lnn57                          1/1     Running   0             17s
kube-proxy-5cwjd                              1/1     Running   0             64m
kube-scheduler-pruebas1                       1/1     Running   1             65m
kubernetes-dashboard-584bfbb648-8rgz2         1/1     Running   0             63m
kubernetes-metrics-scraper-5dc755864d-nx2vz   1/1     Running   0             63m
local-volume-provisioner-2j8x6                1/1     Running   0             63m
nodelocaldns-6chf2                            1/1     Running   0             63m
```

Kubespray Ready !
