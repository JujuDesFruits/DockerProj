# DockerProj

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

suivit du tuto tp

4. Docker registry

