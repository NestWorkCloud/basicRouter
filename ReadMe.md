> [!CAUTION]
> - 1 serveurs sous Debian 12.11.00 (Utilitaires usuels du système uniquement) :
>   - Un serveur Router (basicRouter)
> - Réseau :
>     -  Réseau WAN :
>         -  Réseau     : 192.168.20.0/24
>         -  Passerelle : 192.168.20.254
>     -  Réseau LAN1   :
>         -  Réseau     : 192.168.50.0/24
>         -  Passerelle : N/A
>      -  Réseau LAN2   :
>         -  Réseau     : 192.168.50.0/24
>         -  Passerelle : N/A
>      -  Réseau LAN3   :
>         -  Réseau     : 192.168.50.0/24
>         -  Passerelle : N/A
> - Toutes les commandes sur les différents serveurs sont à exécuter en tant que 'root' sauf mention contraire !

# Mise à jour de la distribution
```
apt-get update && apt-get -y upgrade && apt-get -y dist-upgrade && apt-get -y full-upgrade 
```

# Installation des utilitaires
```
echo iptables-persistent iptables-persistent/autosave_v4 boolean true | sudo debconf-set-selections
echo iptables-persistent iptables-persistent/autosave_v6 boolean true | sudo debconf-set-selections
apt-get -y install nano htop openssh-server iptables iptables-persistent isc-dhcp-server sudo
```

# Configuration des interfaces
## Sauvegarde du fichier de configuration
```
mv /etc/network/interfaces /etc/network/interfaces.bak
```

## Edition du fichier de configuration
```
cat <<EOF > /etc/network/interfaces

EOF
```

## Redémarrage du service réseau
```
systemctl restart networking
```

# Mise en place du routage
## Activation permanente de l'IP forwarding
```
sed -i -e "s/^net.ipv4.ip_forward.*/net.ipv4.ip_forward=1/g" /etc/sysctl.conf
```

## Rechargement de la configuration
```
sysctl -p /etc/sysctl.conf
```

## Activation du NAT
> [!Note]
> Modifier "ens33" par l'interface réseau WAN
```
iptables -t nat -A POSTROUTING -o ens33 -j MASQUERADE
```

## Sauvegarde de la configuration
```
iptables-save > /etc/iptables/rules.v4
```

## Configuration du dhcp
```
sed -i -e "s/^DHCPDv4_CONF.*/DHCPDv4_CONF=/etc/dhcp/dhcpd.conf/g" /etc/default/isc-dhcp-server
sed -i -e "s/^INTERFACESv4.*/INTERFACESv4="ens33"/g" /etc/default/isc-dhcp-server
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
### Etendue 1
```
cat <<EOF >> /etc/dhcp/dhcpd.conf

# Déclaration de l'étendue DHCP ""
subnet 192.168.14.0 netmask 255.255.255.0 {
        range   192.168.14.100 192.168.14.120;
        option domain-name-servers 8.8.8.8;
        option routers 192.168.14.2;
}
EOF
```

### Etendue 2
```
cat <<EOF >> /etc/dhcp/dhcpd.conf

# Déclaration de l'étendue DHCP ""
subnet 192.168.14.0 netmask 255.255.255.0 {
        range   192.168.14.100 192.168.14.120;
        option domain-name-servers 8.8.8.8;
        option routers 192.168.14.2;
}
EOF
```

### Etendue 3
```
cat <<EOF >> /etc/dhcp/dhcpd.conf

# Déclaration de l'étendue DHCP ""
subnet 192.168.14.0 netmask 255.255.255.0 {
        range   192.168.14.100 192.168.14.120;
        option domain-name-servers 8.8.8.8;
        option routers 192.168.14.2;
}
EOF
```

### Redémarrage du service dhcp
```
systemctl restart isc-dhcp-server.service
```

# Sources 
> [Routage](https://alexbacher.fr/unixlinux/routagedeb/)  
> [IP forwarding](https://www.it-connect.fr/activer-lip-forwarding-sous-linux-ipv4ipv6/)  
> [DHCP](https://www.it-connect.fr/serveur-dhcp-sous-linux/)  
