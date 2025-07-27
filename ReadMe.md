# Mise à jour de la distribution
```
apt-get update && apt-get -y upgrade && apt-get -y dist-upgrade && apt-get -y full-upgrade 
```

# Installation des utilitaires
```
echo iptables-persistent iptables-persistent/autosave_v4 boolean true | sudo debconf-set-selections
echo iptables-persistent iptables-persistent/autosave_v6 boolean true | sudo debconf-set-selections
apt-get -y install nano htop openssh-server iptables iptables-persistent
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

# Sources 
> [Routage](https://alexbacher.fr/unixlinux/routagedeb/)  
> [IP forwarding](https://www.it-connect.fr/activer-lip-forwarding-sous-linux-ipv4ipv6/)  
