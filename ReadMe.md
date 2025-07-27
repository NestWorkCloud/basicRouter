> [!CAUTION]
> - 1 serveur sous Debian 12.11.00 (Utilitaires usuels du système uniquement) :
>   - Un serveur Router (basicRouter)
> - Réseau :
>     -  Réseau WAN :
>         -  Réseau     : 192.168.1.193/24
>         -  Passerelle : 192.168.1.254
>         -  DNS : 8.8.8.8
>     -  Réseau LAN1   :
>         -  Réseau     : 172.16.10.0/24
>         -  Passerelle : 172.16.10.254
>         -  DNS : 8.8.8.8
>      -  Réseau LAN2   :
>         -  Réseau     : 172.16.20.0/24
>         -  Passerelle : 172.16.20.254
>         -  DNS : 8.8.8.8
>      -  Réseau LAN3   :
>         -  Réseau     : 172.16.30.0/24
>         -  Passerelle : 172.16.20.254
>         -  DNS : 8.8.8.8
> - Toutes les commandes sur les différents serveurs sont à exécuter en tant que 'root' sauf mention contraire !

# Partie 1 : Préparation du serveur
## Mise à jour de la distribution
```
apt-get update && apt-get -y upgrade && apt-get -y dist-upgrade && apt-get -y full-upgrade 
```

## Installation des utilitaires
```
echo iptables-persistent iptables-persistent/autosave_v4 boolean true | debconf-set-selections
echo iptables-persistent iptables-persistent/autosave_v6 boolean true | debconf-set-selections
apt-get -y install nano htop openssh-server iptables iptables-persistent isc-dhcp-server sudo iputils-ping
```

## Modification du nom du serveur
```
hostnamectl set-hostname basicRouter
sed -i -e "s/^127.0.1.1.*$/127.0.1.1 basicRouter/" /etc/hosts
hostnamectl
```

## Configuration des interfaces
### Sauvegarde du fichier de configuration
```
mv /etc/network/interfaces /etc/network/interfaces.bak
```

### Édition du fichier de configuration
> [!NOTE]
> - Modifier les interfaces :
>   - "enp0s3" par le nom de l'interface réseau WAN
>   - "enp0s8" par le nom de l'interface réseau Lan1
>   - "enp0s9" par le nom de l'interface réseau Lan2
>   - "enp0s10" par le nom de l'interface réseau Lan3
> - Modifier les adresses :
>   - "192.168.1.193/24" par l'adresse ip de l'interface WAN
>   - "192.168.1.254" par l'adresse ip de la box internet
>   - "8.8.8.8" par l'adresse du serveur DNS à utiliser
>   - "172.16.10.254/24" par l'adresse du router à utiliser pour le Lan1
>   - "172.16.20.254/24" par l'adresse du router à utiliser pour le Lan2
>   - "172.16.30.254/24" par l'adresse du router à utiliser pour le Lan3
```
cat <<EOF > /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The Wan interface
allow-hotplug enp0s3
iface enp0s3 inet static
  address 192.168.1.193/24
  gateway 192.168.1.254
  dns-nameservers 8.8.8.8

# The Lan1 interface
auto enp0s8
iface enp0s8 inet static
        address 172.16.10.254/24

# The Lan2 interface
auto enp0s9
iface enp0s9 inet static
        address 172.16.20.254/24

# The Lan3 interface
auto enp0s10
iface enp0s10 inet static
        address 172.16.30.254/24

EOF
```

### Redémarrage du service réseau
```
systemctl restart networking
```

# Partie 2 : Mise en place du routage
## Activation permanente de l'IP forwarding
```
sed -i -e "s/.*net.ipv4.ip_forward.*/net.ipv4.ip_forward=1/g" /etc/sysctl.conf
```

## Rechargement de la configuration
```
sysctl -p /etc/sysctl.conf
```

## Activation du NAT
> [!Note]
> Modifier "enp0s3" par l'interface réseau WAN
```
iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
```

## Sauvegarde de la configuration
```
iptables-save > /etc/iptables/rules.v4
```

# Partie 3 : Configuration du dhcp
## Configuration des options de base du DHCP
```
sed -i -e "s/.*DHCPDv4_CONF.*/DHCPDv4_CONF=\/etc\/dhcp\/dhcpd.conf/g" /etc/default/isc-dhcp-server
sed -i -e 's/.*INTERFACESv4.*/INTERFACESv4="enp0s8 enp0s9 enp0s10"/g' /etc/default/isc-dhcp-server
cat <<EOF >> /etc/dhcp/dhcpd.conf
# Nom de domaine
option domain-name "basicrouter.local";

# Durée pour les baux DHCP en secondes (default 4 jours et 8 jours maximum)
default-lease-time 345600;
max-lease-time 691200;

# Serveur DHCP principal sur ce réseau local
authoritative;

# Logs
log-facility local7;
EOF
```

## Définition des étendues dhcp
### Étendue Lan1
> [!Note]
> - Modifiez les adresses :
>   - "172.16.10.0" par l'adresse réseau du Lan1
>   - "255.255.255.0" par le masque de sous-réseau du Lan1
>   - "172.16.10.100" par la première adresse à distribuer par DHCP sur le Lan1
>   - "172.16.10.120" par la dernière adresse à distribuer par DHCP sur le Lan1
>   - "8.8.8.8" par l'adresse du serveur DNS à distribuer par DHCP sur le Lan1
>   - "172.16.10.254" par l'adresse du routeur à distribuer par DHCP sur le Lan1
```
cat <<EOF >> /etc/dhcp/dhcpd.conf

# Déclaration de l'étendue DHCP "Lan1"
subnet 172.16.10.0 netmask 255.255.255.0 {
        range   172.16.10.100 172.16.10.120;
        option domain-name-servers 8.8.8.8;
        option routers 172.16.10.254;
}
EOF
```

### Étendue Lan2
> [!Note]
> - Modifiez les adresses :
>   - "172.16.20.0" par l'adresse réseau du Lan2
>   - "255.255.255.0" par le masque de sous-réseau du Lan2
>   - "172.16.20.100" par la première adresse à distribuer par DHCP sur le Lan2
>   - "172.16.20.120" par la dernière adresse à distribuer par DHCP sur le Lan2
>   - "8.8.8.8" par l'adresse du serveur DNS à distribuer par DHCP sur le Lan2
>   - "172.16.20.254" par l'adresse du routeur à distribuer par DHCP sur le Lan2
```
cat <<EOF >> /etc/dhcp/dhcpd.conf

# Déclaration de l'étendue DHCP "Lan2"
subnet 172.16.20.0 netmask 255.255.255.0 {
        range   172.16.20.100 172.16.20.120;
        option domain-name-servers 8.8.8.8;
        option routers 172.16.20.254;
}
EOF
```

### Étendue Lan3
> [!Note]
> - Modifiez les adresses :
>   - "172.16.30.0" par l'adresse réseau du Lan3
>   - "255.255.255.0" par le masque de sous-réseau du Lan3
>   - "172.16.30.100" par la première adresse à distribuer par DHCP sur le Lan3
>   - "172.16.30.120" par la dernière adresse à distribuer par DHCP sur le Lan3
>   - "8.8.8.8" par l'adresse du serveur DNS à distribuer par DHCP sur le Lan3
>   - "172.16.30.254" par l'adresse du routeur à distribuer par DHCP sur le Lan3
```
cat <<EOF >> /etc/dhcp/dhcpd.conf

# Déclaration de l'étendue DHCP "Lan3"
subnet 172.16.30.0 netmask 255.255.255.0 {
        range   172.16.30.100 172.16.30.120;
        option domain-name-servers 8.8.8.8;
        option routers 172.16.30.254;
}
EOF
```

### Redémarrage du service dhcp
```
systemctl restart isc-dhcp-server.service
```

# Partie 4 : Baux static
## Attribuer une adresse IP static
> [!Note]
> - Remplacer "ceph1" par le nom de la machine à configurer
> - Remplacer "08:00:27:45:F1:68" par l'adresse MAC de la machine à configurer
> - Remplacer "172.16.10.10" par l'adresse IP souhaitée pour la machine à configurer
```
cat <<EOF >> /etc/dhcp/dhcpd.conf
host ceph1 {
  hardware ethernet 08:00:27:45:F1:68;
  fixed-address 172.16.10.10;
}
EOF
```

# Partie 5 : Ouverture de port
## Ouverture du port SSH
> [!Note]
> - Modifier "enp0s3" par l'interface réseau WAN
> - Modifier "2201" par le port externe à ouvrir (Port à ouvrir sur l'adresse IP de l'interface WAN)
> - Modifier "22" par le port de la machine à mapper
> - Modifier "172.16.10.10" par l'adresse de la machine à mapper
```
iptables -t nat -A PREROUTING -i enp0s3 -p tcp --dport 2201 -j DNAT --to-destination 172.16.10.10:22
```

## Sauvegarde de la configuration
```
iptables-save > /etc/iptables/rules.v4
```

# Sources 
> [Routage](https://alexbacher.fr/unixlinux/routagedeb/)  
> [IP forwarding](https://www.it-connect.fr/activer-lip-forwarding-sous-linux-ipv4ipv6/)  
> [DHCP](https://www.it-connect.fr/serveur-dhcp-sous-linux/)  
