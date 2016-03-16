# taller-hands-on-kubernetes

Taller para instalar kubernetes (paso a paso, the hard way) en un cluster de coreOS en tu Laptop.

## Preliminares

Lo primero es clonar este repositorio, y levantar las maquinas

```
    git clone https://github.com/tidchile/taller-hands-on-kubernetes
    cd taller-hands-on-kubernetes
    vagrant up
    vagrant status
```

Las maquinas están arriba!

Una muy buena aclaración es que porque estemos usando Vagrant, no significa que
necesitemos Vagrant para kubernetes: Simplemente es una forma cómoda de levantar
un número de maquinas virtuales en tu laptop. El escenario ideal es tener los
servidores remotos. Que es donde debería correr la solución de kubernetes.

## ETCD Server

Entramos a la primera maquina. Esta es la que designaremos para correr kubernetes.

```
    vagrant ssh core-01
    ifconfig # Nos entregaría la "IP Pública de la maquina, en este ejemplo será 172.17.8.101"
```

Creamos el drop-in

```
    sudo mkdir -p /etc/systemd/system/etcd2.service.d
    sudo vim /etc/systemd/system/etcd2.service.d/40-listen-address.conf
```

Dentro del editor:

```
[Service]
Environment=ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379
Environment=ETCD_ADVERTISE_CLIENT_URLS=http://172.17.8.101:2379
```

Levantemos el servicio, dejemoslo amarrado al startup y revisemoslo

```
    sudo systemctl start etcd2
    sudo systemctl enable etcd2
    systemctl status etcd2
```

Para comprobar si funciona de afuera

```
    curl http://172.17.8.101:2379/v2/keys
```
Recuerden salir de la máquina core-01 para seguir ejecutando las siguientes sentencias.

## Generemos los elementos TLS

Necesitamos conocer _a priori_ la IP del servidor que escojamos como master.

Usemos el script de este mismo repositorio. Supongamos que la IP de master es
`172.17.8.102`, haremos

```
    ./hack/generate-tls-assets.sh 172.17.8.102
```

## Master

### SSL

Movamos los siguientes archivos a la maquina master

```
    scp -i ~/.vagrant.d/insecure_private_key secrets/{ca,apiserver,apiserver-key}.pem core@172.17.8.102:.
```

Y dentro de la maquina...

```
    sudo mkdir -p /etc/kubernetes/ssl
    sudo mv {ca,apiserver,apiserver-key}.pem /etc/kubernetes/ssl/.
    sudo chmod 600 /etc/kubernetes/ssl/*-key.pem
    sudo chown root:root /etc/kubernetes/ssl/*-key.pem
```

### Flannel

```
    sudo mkdir /etc/flannel
    sudo vim /etc/flannel/options.env
```

Dentro del editor

```
FLANNELD_IFACE=172.17.8.102
FLANNELD_ETCD_ENDPOINTS=http://172.17.8.101:2379
```

```
    sudo mkdir /etc/systemd/system/flanneld.service.d
    sudo vim /etc/systemd/system/flanneld.service.d/40-ExecStartPre-symlink.conf
```

### Docker

```
    sudo mkdir /etc/systemd/system/docker.service.d
    sudo vim /etc/systemd/system/docker.service.d/40-flannel.conf
```

```
[Unit]
Requires=flanneld.service
After=flanneld.service
```

### Kubernetes Services

```
    sudo curl -sSL -o /opt/bin/kubelet https://storage.googleapis.com/kubernetes-release/release/v1.1.8/bin/linux/amd64/kubelet
    sudo chmod +x /opt/bin/kubelet
    sudo mkdir /etc/kubernetes/manifests
    sudo mkdir -p /srv/kubernetes/manifests
```

```
    sudo vim /etc/systemd/system/kubelet.service
```

```
[Service]
ExecStart=/opt/bin/kubelet \
  --api_servers=http://127.0.0.1:8080 \
  --register-node=false \
  --allow-privileged=true \
  --config=/etc/kubernetes/manifests \
  --hostname-override=172.17.8.102 \
  --cluster-dns=10.3.0.10 \
  --cluster-domain=cluster.local
Restart=always
RestartSec=10
[Install]
WantedBy=multi-user.target
```

```
    sudo vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-apiserver
    image: quay.io/coreos/hyperkube:v1.1.8_coreos.0
    command:
    - /hyperkube
    - apiserver
    - --bind-address=0.0.0.0
    - --etcd-servers=http://172.17.8.101:2379
    - --allow-privileged=true
    - --service-cluster-ip-range=10.3.0.0/24
    - --secure-port=443
    - --advertise-address=172.17.8.102
    - --admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota
    - --tls-cert-file=/etc/kubernetes/ssl/apiserver.pem
    - --tls-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem
    - --client-ca-file=/etc/kubernetes/ssl/ca.pem
    - --service-account-key-file=/etc/kubernetes/ssl/apiserver-key.pem
    ports:
    - containerPort: 443
      hostPort: 443
      name: https
    - containerPort: 8080
      hostPort: 8080
      name: local
    volumeMounts:
    - mountPath: /etc/kubernetes/ssl
      name: ssl-certs-kubernetes
      readOnly: true
    - mountPath: /etc/ssl/certs
      name: ssl-certs-host
      readOnly: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/ssl
    name: ssl-certs-kubernetes
  - hostPath:
      path: /usr/share/ca-certificates
    name: ssl-certs-host
```

```
    sudo vim /etc/kubernetes/manifests/kube-proxy.yaml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: kube-proxy
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-proxy
    image: quay.io/coreos/hyperkube:v1.1.8_coreos.0
    command:
    - /hyperkube
    - proxy
    - --master=http://127.0.0.1:8080
    - --proxy-mode=iptables
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /etc/ssl/certs
      name: ssl-certs-host
      readOnly: true
  volumes:
  - hostPath:
      path: /usr/share/ca-certificates
    name: ssl-certs-host
```

```
    sudo vim /etc/kubernetes/manifests/kube-podmaster.yaml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: kube-podmaster
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: scheduler-elector
    image: gcr.io/google_containers/podmaster:1.1
    command:
    - /podmaster
    - --etcd-servers=http://172.17.8.101:2379
    - --key=scheduler
    - --whoami=172.17.8.102
    - --source-file=/src/manifests/kube-scheduler.yaml
    - --dest-file=/dst/manifests/kube-scheduler.yaml
    volumeMounts:
    - mountPath: /src/manifests
      name: manifest-src
      readOnly: true
    - mountPath: /dst/manifests
      name: manifest-dst
  - name: controller-manager-elector
    image: gcr.io/google_containers/podmaster:1.1
    command:
    - /podmaster
    - --etcd-servers=http://172.17.8.101:2379
    - --key=controller
    - --whoami=172.17.8.102
    - --source-file=/src/manifests/kube-controller-manager.yaml
    - --dest-file=/dst/manifests/kube-controller-manager.yaml
    terminationMessagePath: /dev/termination-log
    volumeMounts:
    - mountPath: /src/manifests
      name: manifest-src
      readOnly: true
    - mountPath: /dst/manifests
      name: manifest-dst
  volumes:
  - hostPath:
      path: /srv/kubernetes/manifests
    name: manifest-src
  - hostPath:
      path: /etc/kubernetes/manifests
    name: manifest-dst
```

```
    sudo vim /srv/kubernetes/manifests/kube-controller-manager.yaml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-controller-manager
    image: quay.io/coreos/hyperkube:v1.1.8_coreos.0
    command:
    - /hyperkube
    - controller-manager
    - --master=http://127.0.0.1:8080
    - --service-account-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem
    - --root-ca-file=/etc/kubernetes/ssl/ca.pem
    livenessProbe:
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10252
      initialDelaySeconds: 15
      timeoutSeconds: 1
    volumeMounts:
    - mountPath: /etc/kubernetes/ssl
      name: ssl-certs-kubernetes
      readOnly: true
    - mountPath: /etc/ssl/certs
      name: ssl-certs-host
      readOnly: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/ssl
    name: ssl-certs-kubernetes
  - hostPath:
      path: /usr/share/ca-certificates
    name: ssl-certs-host
```

```
    sudo vim /srv/kubernetes/manifests/kube-scheduler.yaml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: kube-scheduler
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-scheduler
    image: quay.io/coreos/hyperkube:v1.1.8_coreos.0
    command:
    - /hyperkube
    - scheduler
    - --master=http://127.0.0.1:8080
    livenessProbe:
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10251
      initialDelaySeconds: 15
      timeoutSeconds: 1
```

OK. Echemos a andar master!

```
sudo systemctl daemon-reload

curl -X PUT -d "value={\"Network\":\"10.2.0.0/16\",\"Backend\":{\"Type\":\"vxlan\"}}" "http://172.17.8.101:2379/v2/keys/coreos.com/network/config"

sudo systemctl start kubelet
sudo systemctl enable kubelet
```

Comprobemos si anda

```
    systemctl status kubelet
    docker ps -a
```

Debería mostrarnos algunos containers que ya estan siendo bajados

Para terminar con master, creemos el namespace de kube-system. Este es útil para un número de razones

Primero veamos si anda la API de kubernetes

```
    curl http://127.0.0.1:8080/version
```

```
{
  "major": "1",
  "minor": "1",
  "gitVersion": "v1.1.8_coreos.0",
  "gitCommit": "197bd0e32d7f81ae3cac410a959a957f88e48419",
  "gitTreeState": "clean"
}
```

Ya. Creemos el namespace

```
    curl -XPOST -d'{"apiVersion":"v1","kind":"Namespace","metadata":{"name":"kube-system"}}' "http://127.0.0.1:8080/api/v1/namespaces"
```

## Workers

### TLS

Movamos para cada maquina los archivos

```
    scp -i ~/.vagrant.d/insecure_private_key secrets/{ca,worker,worker-key}.pem core@172.17.8.103:.
    scp -i ~/.vagrant.d/insecure_private_key secrets/{ca,worker,worker-key}.pem core@172.17.8.104:.
```

Y dentro de cada maquina

```
    sudo mkdir -p /etc/kubernetes/ssl
    sudo mv {ca,worker,worker-key}.pem /etc/kubernetes/ssl/.
    sudo chmod 600 /etc/kubernetes/ssl/*-key.pem
    sudo chown root:root /etc/kubernetes/ssl/*-key.pem
```

### Flannel

sudo mkdir /etc/flannel
sudo vim /etc/flannel/options.env

FLANNELD_IFACE=172.17.8.103
FLANNELD_ETCD_ENDPOINTS=http://172.17.8.101:2379

sudo mkdir /etc/systemd/system/flanneld.service.d
sudo vim /etc/systemd/system/flanneld.service.d/40-ExecStartPre-symlink.conf

[Service]
ExecStartPre=/usr/bin/ln -sf /etc/flannel/options.env /run/flannel/options.env

### Docker

sudo mkdir /etc/systemd/system/docker.service.d
sudo vim /etc/systemd/system/docker.service.d/40-flannel.conf

```
[Unit]
Requires=flanneld.service
After=flanneld.service
```

### Kubelet

```
    sudo mkdir -p /opt/bin
    sudo curl -sSL -o /opt/bin/kubelet https://storage.googleapis.com/kubernetes-release/release/v1.1.8/bin/linux/amd64/kubelet
    sudo chmod +x /opt/bin/kubelet
    sudo mkdir /etc/kubernetes/manifests sudo mkdir -p /srv/kubernetes/manifests
```

```
    sudo vim /etc/systemd/system/kubelet.service
```

```
[Service]
ExecStartPre=/usr/bin/mkdir -p /etc/kubernetes/manifests

ExecStart=/opt/bin/kubelet \
  --api_servers=https://172.17.8.102 \
  --register-node=true \
  --allow-privileged=true \
  --config=/etc/kubernetes/manifests \
  --hostname-override=172.17.8.103 \
  --cluster-dns=10.3.0.10 \
  --cluster-domain=cluster.local \
  --kubeconfig=/etc/kubernetes/worker-kubeconfig.yaml \
  --tls-cert-file=/etc/kubernetes/ssl/worker.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/worker-key.pem
Restart=always
RestartSec=10
[Install]
WantedBy=multi-user.target
```

```
    sudo vim /etc/kubernetes/manifests/kube-proxy.yaml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: kube-proxy
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-proxy
    image: quay.io/coreos/hyperkube:v1.1.8_coreos.0
    command:
    - /hyperkube
    - proxy
    - --master=https://172.17.8.102
    - --kubeconfig=/etc/kubernetes/worker-kubeconfig.yaml
    - --proxy-mode=iptables
    securityContext:
      privileged: true
    volumeMounts:
      - mountPath: /etc/ssl/certs
        name: "ssl-certs"
      - mountPath: /etc/kubernetes/worker-kubeconfig.yaml
        name: "kubeconfig"
        readOnly: true
      - mountPath: /etc/kubernetes/ssl
        name: "etc-kube-ssl"
        readOnly: true
  volumes:
    - name: "ssl-certs"
      hostPath:
        path: "/usr/share/ca-certificates"
    - name: "kubeconfig"
      hostPath:
        path: "/etc/kubernetes/worker-kubeconfig.yaml"
    - name: "etc-kube-ssl"
      hostPath:
        path: "/etc/kubernetes/ssl"

```

```
sudo vim /etc/kubernetes/worker-kubeconfig.yaml
```

```
apiVersion: v1
kind: Config
clusters:
- name: local
  cluster:
    certificate-authority: /etc/kubernetes/ssl/ca.pem
users:
- name: kubelet
  user:
    client-certificate: /etc/kubernetes/ssl/worker.pem
    client-key: /etc/kubernetes/ssl/worker-key.pem
contexts:
- context:
    cluster: local
    user: kubelet
  name: kubelet-context
current-context: kubelet-context
```

### Start Services

```
    sudo systemctl daemon-reload
    sudo systemctl start kubelet
    sudo systemctl enable kubelet
    sytemctl status kubelet
```

## Kubectl (manejemos la cosa)

kubectl es la magia acá. Te hace "sentir" que no manejas un cluster, sino que te estás entendiendo
con un servidor.

```
    curl -O https://storage.googleapis.com/kubernetes-release/release/v1.1.8/bin/darwin/amd64/kubectl

    chmod +x kubectl
    mv kubectl /usr/local/bin/kubectl

    kubectl config set-cluster default-cluster --server=https://${MASTER_HOST} --certificate-authority=${CA_CERT}
    kubectl config set-credentials default-admin --certificate-authority=${CA_CERT} --client-key=${ADMIN_KEY} --client-certificate=${ADMIN_CERT}
    kubectl config set-context default-system --cluster=default-cluster --user=default-admin
    kubectl config use-context default-system
```

Por ejemplo yo hice:

```
    kubectl config set-cluster default-cluster --server=https://172.17.8.102 --certificate-authority=ca.pem
    kubectl config set-credentials default-admin --certificate-authority=ca.pem --client-key=admin-key.pem --client-certificate=admin.pem
    kubectl config set-context default-system --cluster=default-cluster --user=default-admin
    kubectl config use-context default-system
```

YMMV of course

### Veamos si funcionó!

```
    kubectl get nodes
    kubectl get po --namespace=kube-system
```

## DNS Addon

De aquí en adelante, no deberíamos meternos más (en teoría a los servidores), o sea, jovenes, `kubectl` es tu amigo

```
    kubectl create -f dns-addon.yml

    kubectl get pods --name-space=kube-system | grep kube-dns-v9
```

Para seguir, veamos este ejemplo https://github.com/hermanjunge/k8s-examples

Muchas Gracias.

HJ.-
