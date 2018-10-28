# DockerProj

## Part 1 : Gitlab

1. Ajout d'un disque

En utilisant virtualbox, deux disques virtuels ont été définit (MAIN et DATA)
En manipulant la commande fdisk puis mkfs.ext4 sur le dossier crée à la main /data/docker,j'ai liée le répertoire au deuxième disque data.
Les commandes on été les suivante:
```
mkfs.ext4 /dev/sdb1
echo "/dev/sdb1 /data/docker ext4 defaults 0 0" >> /etc/fstab
mount -a
```
Suite au problème de connexion, il a été necessaire de modifier le fichier : /etc/sysconfig/network-scripts/ifcfg-enp0s3
modification --> ONBOOT=yes

2. Installer Docker

l'installation est la même que pour le tp1

pour modifier le repertoire de donnée, crée le fichier /etc/docker/daemon.json
```
{
  "graph": "/data/docker"
}
```
et relancer le daemon

3. Installer gitlab-ce

suivit du tuto tp.
pour le parametrage du firewall nous nous somme contenté de taper:
```
firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --zone=public --add-port=443/tcp --permanent
systemctl reload firewalld
```

4. Docker registry

j'ai récupérer le dossier git via
```
git clone git@192.168.56.101:julien/test.git
```
pour cela penser à ajouter une clé rsa via la command ssh-keygen et de la donner dans le repertoire git
suite à la modifcation de /etc/gitlab/gitlab.rb j'ai du effectuer un changemant sur ma clée ssl pour pouvoir effectuer un "docker login" (dossier /etc/gitlab/ssl/)
```
echo subjectAltName = IP:127.0.0.1 > extfile.cnf
openssl x509 -req -days 365 -in <FQDN>.crt -CA ca.pem -CAkey ca-key.pem -CAcreateserial \
   -out server-cert.pem -extfile extfile.cnf
gitlab-ctl reconigure
```
## Part 2 : Docker swarm

1. Création du swarm

l'installation reste la même qu'avant pour docker et l'on clone la machine comme recommandé.
J'ai donc swarm1 --> 192.168.56.101
          swarm2 --> 192.168.56.102
          swarm2 --> 192.168.56.103
swarm1 est définit comme manager sur notre config swarm:
```
docker swarm init --advertise-addr 192.168.56.101
```
les deux autres vm ont le role de worker via la commande :
```
docker swarm join --token SWMTKN-1-0bukf0avzh9dk54cqqzyqo2bflmyrg7o8r6nxxnjb5rrpcdwd3-bb547wb6wr1nysmiwhe1dn5dm 192.168.56.101:2377
```
(attention il s'agit du token définit par docker lors que l'on a définit un manager)

sur les machines on créé un doc /etc/systemd/system/docker.service/docker.conf  :
```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// \
  --storage-driver=overlay2 \
  --dns 8.8.4.4 --dns 8.8.8.8 \
  --log-driver json-file \
  --log-opt max-size=50m --log-opt max-file=10 \
  --experimental=true \
  --metrics-addr 0.0.0.0:9323
```
et relancer docker.service pour récolter des métriques sur le swarm

2. Lancer une app sur le swarm

En premier lieu, nous avons installé un registry docker en suivant la doc officiel
```
docker service create --name registry --publish published=5000,target=5000 registry:2
```
puis nous rajoutons cette ligne ci-dessous dans notre daemon.json
```
{
  "insecure-registries" : ["http://localhost:5000"]
}
```
en réutilisant le docker-compose du TP1, nous modifions ce dernier en rajoutant les lignes suivantes et le tour est joué:
```
deploy:
  mode: replicated
  replicas: 5
  labels:
    label1: 'value1'
  placement:
    constraints:
      - node.role == worker
```

NGINX peut être simplement remplacé par le service web énoncé dans le dossier docker-compose.yml
```
services:
  web:
    image: 127.0.0.1:5000/repo:tag
    build: .
    ports:
      - "8000:8000"
```

3. Faire joujou: obtenir de la visualisation et de métriques

le Portainer a été lancé via
```
curl -L https://portainer.io/download/portainer-agent-stack.yml -o portainer-agent-stack.yml
docker stack deploy --compose-file=portainer-agent-stack.yml portainer
```

dans le cas de weave cloud, après avoir un compte et demandé un token, on a utilisé cette ligne pour le déployer:
```
docker run -it --rm -v /var/run/docker.sock:/var/run/docker.sock weaveworks/swarm-agents install doy46duhecrsty8z64nd6qb4sz5zadg4
```
un conteneur avec les images swarm-agents:latest et scope:1.9.1 est téléchargé. un conteneur weaveworks/scope:1.9.1 permet donc d'interconnecter différent docker-machine

et enfin prometheus peut être lancé avec la commande suivante sur le swarm-manager
```
docker service create --replicas 1 --name my-prometheus \
    --mount type=bind,source=/tmp/prometheus.yml,destination=/etc/prometheus/prometheus.yml \
    --publish published=9090,target=9090,protocol=tcp \
    prom/prometheus
```
pour atteindre la page web il suffit d'aller sur http://localhost:9090
