## DRBL + NFS : Guide détaillé prêt pour GitHub

### Objectif du Projet
Ce projet consiste à configurer un serveur DRBL (Diskless Remote Boot in Linux) couplé à un serveur NFS (Network File System) afin de faciliter la gestion et le déploiement d'images disque sur un réseau local.

### Composants clés
- **DRBL** : Permet aux machines clientes un démarrage réseau via PXE sans disque.
- **NFS** : Fournit un stockage centralisé des images disques accessible par les clients.
- **Clonezilla** : Outil permettant de cloner ou restaurer les images stockées sur le serveur NFS.

### Avantages du projet
- Simplifie les déploiements massifs d'images.
- Centralise le stockage des images disque.
- Automatise les processus de sauvegarde et de restauration.

### Configuration Réseau (exemple avec réseau 192.168.50.0/24)
- Interface Internet (`ens33`) via DHCP.
- Interface interne statique (`ens37`) : `192.168.50.10`.

---

## Guide détaillé étape par étape

### Étape 1 : Configuration du Réseau et TFTP

Éditez le fichier Netplan :
```bash
sudo nano /etc/netplan/01-netcfg.yaml
```

Exemple de configuration :
```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: yes
    ens37:
      dhcp4: no
      addresses: [192.168.50.10/24]
```
Appliquer les modifications :
```bash
sudo netplan apply
```
[Documentation Netplan](https://netplan.io/reference/)

Éditer TFTP (`/etc/default/tftpd-hpa`) :
```conf
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/tftpboot/nbi_img"
TFTP_ADDRESS="0.0.0.0:69"
TFTP_OPTIONS="--secure --ipv4"
```

### Mise à jour système et Installation des paquets
```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install drbl nfs-kernel-server tftpd-hpa isc-dhcp-server syslinux-common unzip wget -y
```

### Ajout de la clé GPG DRBL et du dépôt
```bash
wget -O - https://drbl.org/GPG-KEY-DRBL | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/drbl.gpg
sudo nano /etc/apt/sources.list
```
Ajouter ces lignes :
```conf
deb http://archive.ubuntu.com/ubuntu noble main restricted universe multiverse
deb http://free.nchc.org.tw/drbl-core drbl stable
```

### Installation et Configuration de DRBL
```bash
sudo apt update
sudo apt install drbl -y
sudo drblsrv -i
sudo drblpush -i
```

### Configuration DHCP pour démarrage PXE
Modifier `/etc/dhcp/dhcpd.conf` :
```conf
subnet 192.168.50.0 netmask 255.255.255.0 {
  option subnet-mask 255.255.255.0;
  option routers 192.168.50.1;
  option domain-name-servers 8.8.8.8, 8.8.4.4;
  next-server 192.168.50.10;

  pool {
    range 192.168.50.100 192.168.50.150;
    allow unknown-clients;
  }

  filename "pxelinux.0";
}
```

### Configuration NFS
Éditer `/etc/exports` :
```conf
/tftpboot/node_root/clonezilla-live 192.168.50.0/24(rw,sync,no_subtree_check,no_root_squash)
/home/partimag 192.168.50.0/24(rw,sync,no_subtree_check,no_root_squash)
```
Appliquer les modifications :
```bash
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
sudo chmod -R 777 /home/partimag
```
[Documentation NFS](https://help.ubuntu.com/community/NFSv4Howto)

### Montage automatique NFS Synology
Éditer `/etc/fstab` :
```bash
192.168.50.20:/volume1/drbl-images /mnt/synology_partimag nfs defaults,_netdev 0 0
/mnt/synology_partimag /home/partimag none bind 0 0
```
Appliquer le montage automatique :
```bash
sudo mount -a
df -h /home/partimag
```

### Configuration PXE Clonezilla
Ajouter à `/tftpboot/nbi_img/pxelinux.cfg/default` :
```conf
label Clonezilla-live
  MENU LABEL Clonezilla: sauvegarde ou restauration ultérieure
  KERNEL clonezilla-live/vmlinuz
  APPEND initrd=clonezilla-live/initrd.img boot=live union=overlay username=user hostname=clonezilla config loglevel=0 netboot=nfs nfsroot=192.168.50.10:/tftpboot/node_root/clonezilla-live/ ocs_server="192.168.50.10" ocs_daemonon="ssh" ocs_prerun="mount -t nfs -o vers=3 192.168.50.20:/volume1/drbl-images /home/partimag"
```

---

### Important
- Remplacer les noms d'interfaces réseau (`ens33`, `ens37`) par ceux adaptés à votre système.
- Modifier les IP selon votre configuration réseau locale.

[Documentation Clonezilla](https://clonezilla.org/livepxe.php)

