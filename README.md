
# Configuration de l'Infrastructure Réseau

## Description du Projet

Ce projet vise à configurer une infrastructure réseau avec les éléments suivants :
- **Routeur** en mode Bridge (192.168.1.1/24)
- **GW01** en mode Bridge (192.168.1.2/24) et Host-Only (172.16.1.1/24)
- **MG01** en mode Host-Only (172.16.1.3/24)
- **APP01** en mode Host-Only (172.16.1.4/24)
- **CLI01** en mode Host-Only (172.16.1.2/24)

## Pré-requis

- Installer Debian 10 Buster sur toutes les machines (GW01, MG01, APP01, CLI01).
- Configuration de l'adressage IP et des interfaces réseau.

## Étape 1 : Configuration des Interfaces Réseau

### GW01

```bash
# /etc/network/interfaces
auto eth1
iface eth1 inet static
  address 192.168.1.2
  netmask 255.255.255.0
  gateway 192.168.1.1

auto eth2
iface eth2 inet static
  address 172.16.1.1
  netmask 255.255.255.0
```

### MG01

```bash
# /etc/network/interfaces
auto eth1
iface eth1 inet static
  address 172.16.1.3
  netmask 255.255.255.0
  gateway 172.16.1.1
```

### APP01

```bash
# /etc/network/interfaces
auto eth1
iface eth1 inet static
  address 172.16.1.4
  netmask 255.255.255.0
  gateway 172.16.1.1
```

### CLI01

```bash
# /etc/network/interfaces
auto eth1
iface eth1 inet static
  address 172.16.1.2
  netmask 255.255.255.0
  gateway 172.16.1.1
```

Redémarrer le service réseau :
```bash
sudo systemctl restart networking
```

## Étape 2 : Installer et Configurer le Serveur DHCP sur MG01

### Installation du Serveur DHCP

```bash
sudo apt-get update
sudo apt-get install isc-dhcp-server
```

### Configuration de `/etc/dhcp/dhcpd.conf`

```bash
subnet 172.16.1.0 netmask 255.255.255.0 {
  range 172.16.1.10 172.16.1.20;
  option routers 172.16.1.1;
  option domain-name-servers 172.16.1.3;
}
host APP01 {
  hardware ethernet <MAC-ADDRESS>;
  fixed-address 172.16.1.4;
}
```

### Redémarrer le Service DHCP

```bash
sudo systemctl restart isc-dhcp-server
```

## Étape 3 : Configurer le DNS sur MG01

### Installation du Serveur DNS (BIND)

```bash
sudo apt-get install bind9
```

### Configuration des Fichiers de Zone DNS

Ajouter une zone pour `eftw.local` dans le fichier `/etc/bind/named.conf.local` :

```bash
zone "eftw.local" {
  type master;
  file "/etc/bind/db.eftw.local";
};
```

Créer le fichier `/etc/bind/db.eftw.local` :

```bash
$TTL 604800
@   IN  SOA ns.eftw.local. root.eftw.local. (
              2     ; Serial
          604800     ; Refresh
           86400     ; Retry
         2419200     ; Expire
          604800 )   ; Negative Cache TTL
;
@   IN  NS  ns.eftw.local.
ns  IN  A   172.16.1.3
gw01    IN  A   172.16.1.1
mg01    IN  A   172.16.1.3
app01   IN  A   172.16.1.4
cli01   IN  A   172.16.1.2
```

### Redémarrer le Service BIND

```bash
sudo systemctl restart bind9
```

## Étape 4 : Configurer les Règles de Pare-feu avec Iptables sur GW01

### Installation d'Iptables

```bash
sudo apt-get install iptables
```

### Configuration des Règles NAT et de Filtrage

Activer le forwarding dans `/etc/sysctl.conf` :

```bash
net.ipv4.ip_forward=1
```

Appliquer la configuration :

```bash
sudo sysctl -p
```

Ajouter les règles iptables :

```bash
sudo iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE
sudo iptables -A FORWARD -i eth2 -o eth1 -j ACCEPT
sudo iptables -A FORWARD -i eth1 -o eth2 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

Sauvegarder les règles iptables :

```bash
sudo iptables-save > /etc/iptables/rules.v4
```

## Étape 5 : Configurer SSH et Fail2Ban sur MG01 et APP01

### Configurer SSH pour l'Authentification par Clés

Générer des clés SSH sur votre machine locale :

```bash
ssh-keygen
```

Copier la clé publique sur MG01 et APP01 :

```bash
ssh-copy-id user@mg01
ssh-copy-id user@app01
```

Désactiver l'authentification par mot de passe dans `/etc/ssh/sshd_config` :

```bash
PasswordAuthentication no
```

### Installation de Fail2Ban

```bash
sudo apt-get install fail2ban
```

Configurer `/etc/fail2ban/jail.local` pour SSH :

```bash
[sshd]
enabled = true
port = ssh
logpath = /var/log/auth.log
maxretry = 3
```

Redémarrer Fail2Ban :

```bash
sudo systemctl restart fail2ban
```

## Étape 6 : Installer et Configurer GitLab sur APP01

### Installation de GitLab

Suivre les instructions sur le site officiel de GitLab pour installer la version Community.

### Configuration de GitLab avec le Domaine `gitlab.eftw.local`

Ajouter un enregistrement DNS pour GitLab :

```bash
gitlab IN A 172.16.1.4
```

## Étape 7 : Installer Centreon sur APP01

### Installation de Centreon

Suivre les instructions sur le site officiel de Centreon pour l'installation.

### Ajouter les Configurations Nécessaires pour la Supervision

Ajouter les enregistrements DNS pour Centreon :

```bash
centreon IN A 172.16.1.4
```

## Étape 8 : Tests et Validation

### Vérifier la Connectivité Réseau et les Résolutions DNS

- Ping entre les machines.
- Résolution des noms de domaine internes avec `nslookup` ou `dig`.

### Tester les Services (DHCP, DNS, SSH, GitLab, Centreon)

Assurez-vous que tous les services fonctionnent correctement et que les configurations respectent les exigences du projet.
