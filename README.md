# Cluster-DNS-avec-BIND9-
# Configuration et Installation de BIND9

# Installation de BIND9 ainsi que ses fonctionnalités :

# Sur chaque serveur DNS (Srv1-D12 et Srv2-D12), exécutez les commandes suivantes en tant qu'utilisateur root ou via sudo :

# Mettre à jour les paquets
    apt-get update

# Installer bind9 et son utilitaire dig
    apt-get install bind9 bind9utils -y

# Activer et démarrer le service bind9
    systemctl enable bind9 && systemctl start bind9

# Vérifier le statut du service
    systemctl status bind9

# Configuration de zone principale sur le Srv1-D12 :

# Éditez le fichier named.conf.local situé dans le répertoire /etc/bind. Remplacez le contenu actuel par ce qui suit, en adaptant bien sûr les séries de numéro IP aux adresses des machines concernées :

    zone "dieudo.local" {
    type master;
    file "/etc/bind/db.dieudo.local";
    };

    zone "12.149.in-addr.arpa" {
    type master;
    file "/etc/bind/db.149";
    };

# Créez ensuite le fichier /etc/bind/db.dieudo.local contenant les informations relatives à votre nom de domaine personnalisé :

    $TTL    604800
    @       IN      SOA     ns1.dieudo.local. admin.dieudo.local. (
                  1         ; Serial
         604800         ; Refresh
          86400         ; Retry
        2419200         ; Expire
         604800 )       ; Negative Cache TTL
    ;
    @       IN      NS      ns1.dieudo.local.
    @       IN      A       10.132.149.151
    ns1     IN      A       10.132.149.151
    www     IN      CNAME   @

# De même, créez le fichier /etc/bind/db.149 relatif au reverse DNS :

    $TTL    604800
    @       IN      SOA     ns1.dieudo.local. admin.dieudo.local. (
                  1         ; Serial
         604800         ; Refresh
          86400         ; Retry
        2419200         ; Expire
         604800 )       ; Negative Cache TTL
    ;
    @       IN      NS      ns1.dieudo.local.
    151     IN      PTR     ns1.dieudo.local.

# Mettez à jour le serial dans chacun des nouveaux fichiers créés (par exemple, remplacez le nombre 1 par 2022030701, puis incrémentez-le à chaque modification) :

    echo "$TTL    604800
    @       IN      SOA     ns1.dieudo.local. admin.dieudo.local. (" > /etc/bind/db.dieudo.local
    echo "@       IN      NS      ns1.dieudo.local." >> /etc/bind/db.dieudo.local
    echo "@       IN      A       10.132.149.151" >> /etc/bind/db.dieudo.local
    echo "ns1     IN      A       10.132.149.151" >> /etc/bind/db.dieudo.local
    echo "www     IN      CNAME   @" >> /etc/bind/db.dieudo.local
    echo "2022030701" > /etc/bind/db.dieudo.local.serial

    echo "$TTL    604800
    @       IN      SOA     ns1.dieudo.local. admin.dieudo.local. (" > /etc/bind/db.149
    echo "@       IN      NS      ns1.dieudo.local." >> /etc/bind/db.149
    echo "151     IN      PTR     ns1.dieudo.local." >> /etc/bind/db.149
    echo "2022030701" > /etc/bind/db.149.serial

# Vérifiez la syntaxe de vos fichiers de zones :

    named-checkzone dieudo.local /etc/bind/db.dieudo.local
    named-checkzone 149.132.10.in-addr.arpa /etc/bind/db.149

# Redémarrez le service BIND9 pour prendre en compte les modifications :

    systemctl restart bind9

# Configuration de zone secondaire sur le Srv2-D12 :

# Sur Srv2-D12, éditez le fichier /etc/bind/named.conf.options afin d'ajouter une ligne permettant la communication entre les deux serveurs DNS :

    allow-transfer {"none";};
    + allow-transfer { Srv1-D12_private_ip; };

# Ensuite, modifiez le fichier /etc/bind/named.conf.local comme ceci :

    zone "dieudo.local" {
    type slave;
    masters { Srv1-D12_private_ip; };
    file "/etc/bind/db.dieudo.local";
    };

    zone "12.149.in-addr.arpa" {
    type slave;
    masters { Srv1-D12_private_ip; };
    file "/etc/bind/db.149";
    };

# Puis, copiez les fichiers de zones depuis Srv1-D12 vers Srv2-D12 :

    scp user@Srv1-D12_private_ip:/etc/bind/db.* /etc/bind/

# Et redémarrez le service BIND9 sur Srv2-D12 :

    systemctl restart bind9

# Tests et Vérifications :

# Testez la résolution des noms FQDN en utilisant la commande dig sur un hôte client connecté au même réseau local :

    dig @Srv1-D12_private_ip ns1.dieudo.local
    dig @Srv1-D12_private_ip www.dieudo.local
    dig @Srv2-D12_private_ip ns1.dieudo.local
    dig @Srv2-D12_private_ip www.dieudo.local

# Assurez-vous également de pouvoir effectuer des requêtes inversées :

    dig -x 10.132.149.151 +short
    dig -x 10.132.149.152 +short

# Keepalived pour la gestion de l'IP Virtuelle :

# Installez keepalived sur les deux serveurs DNS :

    apt-get install keepalived -y

# Configurez le maître (Srv1-D12) en éditant le fichier /etc/keepalived/keepalived.conf :

    global_defs {
       router_id LVS_DEVEL
    }

    vrrp_instance VI_1 {
    state MASTER
    interface ens160 # Change this to your network interface name
    virtual_router_id 51
    priority 101
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass mysecret
       }
    virtual_ipaddress {
        10.132.149.155
       }
     }
# Configurez l'esclave (Srv2-D12) en éditant le fichier /etc/keepalived/keepalived.conf :

    global_defs {
       router_id LVS_BACKUP
    }

      vrrp_instance VI_1 {
      state BACKUP
      interface ens160 # Change this to your network interface name
      virtual_router_id 51
      priority 100
      advert_int 1
      authentication {
        auth_type PASS
        auth_pass mysecret
      }
      virtual_ipaddress {
        10.132.149.155
      }
    }

# Activez et démarrez le service keepalived sur chaque nœud :

    systemctl enable keepalived && systemctl start keepalived

# Vérifiez que l'adresse IP virtuelle est correctement affectée au maître (Srv1-D12) :

