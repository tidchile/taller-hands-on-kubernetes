# taller-hands-on-kubernetes
Taller para instalar kubernetes (paso a paso, the hard way) en un cluster de coreOS en tu Laptop.


* Bajar el repo
* unzippear el repo
* cd
* vagrant up
* vagrant status
* vagrant ssh core-01
* ifconfig
* 172.17.8.101 - esta es la "ip publica"
* sudo mkdir -p /etc/systemd/system/etcd2.service.d
* sudo vim /etc/systemd/system/etcd2.service.d/40-listen-address.conf

```
[Service]
Environment=ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379
Environment=ETCD_ADVERTISE_CLIENT_URLS=http://172.17.8.101:2379
```

* sudo systemctl start etcd2
* sudo systemctl enable etcd2
* systemctl status etcd2
* curl http://172.17.8.101:2379/v2/keys

# Generemos los elements TLS

Necesitamos conocer _a priori_ la IP del servidor que escojamos como master.

Usemos el script de este mismo repositorio. Supongamos que la IP de master es
`172.17.8.102`, haremos

```
	./hack/generate-tls-assets.sh 172.17.8.102
```

# Master




