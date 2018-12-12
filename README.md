# NSD 2018 - Workshop Acorus Networks

## Topology

```
                    Host - NAT - SSH 
    -----------------------------------------
      |                                   |
      |                                   |
  em0 |       10.1.2.1    10.1.2.2  eth0  |
=============  xe-0/0/0     eth1   =============
|           | -------------------- |   srv     |
|   vqfx    |                      =============
|           |                
=============              
  em1 |                  
=============             
| vqfx-pfe  |             
============= 
```  

## Pre-requis

- Vagrant (test avec 2.2)
- VirtualBox (test avec 5.2)

## Outils

- Ansible installé sur le srv
- Bird installé sur le srv (avec une session BGP active)
- Vault

## Installation

#### via github

```
$ git clone https://github.com/alexandrecorso/nsd2018.git
```

#### via usb

Copier le contenu de la clé USB dans votre repertoire
```
$ cp /Volumes/NSD/* ~/
```

Aller dans le repertoire home et initialiser les VM
```
$ cd ~/
$ vagrant box add vqfx10k-re.box --name juniper/vqfx10k-re
$ vagrant box add vqfx10k-pfe.box --name juniper/vqfx10k-pfe
$ vagrant box add xenial64.box --name ubuntu/xenial64
```

Aller dans le repertoire vagrant
```
$ ~/vqfx10k-vagrant/full-1qfx-1srv
```

Lancer les VM
```
$ vagrant up
```

Attention, le demarrage des VMs peut etre long, il depend de l'ordinateur utilisé

Une fois, les 3 VMs demarrées, on peut faire quelque tests, exemple de quelques commandes :
#### VQFX
```
$ vagrant ssh vqfx
vagrant@vqfx-re> show configuration
```
OUTPUT
```
## Last commit: 2018-12-11 23:48:57 UTC by vagrant
version 17.4R1.16;
system {
    host-name vqfx-re;
...
```

#### Serveur

 * user: vagrant
 * password: vagrant

```
$ vagrant ssh srv
vagrant@server:~$ ifconfig
vagrant@server:~$ ping 10.1.2.1
vagrant@server:~$ ssh vqfx
```

#### Ansible
```
vagrant@server:~$ vagrant ssh srv
vagrant@server:~/ansible$ cd ansible
vagrant@server:~/ansible$ ansible-playbook -i inventories/hosts pb.test.netconf.yaml --vault-id ~/.vault_pass.txt
```
OUTPUT
```
TASK [Print version] ****************************************************************************************************************
ok: [vqfx] => {
    "junos.version": "17.4R1.16"
}
```


## Utilisation

### Etape 1: Ajouter un utilisateur

#### Recuperer le HASH

Avoir le HASH du mot de passe ou le creer (ici on utilise un JunOS)
```
$ vagrant ssh vqfx

vagrant@vqfx-re> edit
vagrant@vqfx-re# set system login user {YOUR_USER} class read-only authentication plain-text-password
New password:
Retype new password:

vagrant@vqfx-re# show | compare
```

On utilise le HASH genere qui commence par $6$:
```
+    user acorso {
+        class read-only;
+        authentication {
+            encrypted-password "$6$GAMRvF.5$.yhfl02NtcTn03p.9Z/Mts0Rhhif0qibapCXdlMa8DvzfGidRknsFQRVH6VTMUyc4DST9.hWZoB5EjE0D3cdC/"; ## SECRET-DATA
+        }
+    }
```

#### Modifier l'inventaire
```
vagrant@server:~/ansible$ ansible-vault edit inventories/group_vars/all/users.yaml
Vault password:
```

Le mot de passe du vault est : ```gN4PFTz5nmWWqpiYzmrcdSwHif6ePkYy7zwyhfkeFAQW9HwAVM3edKjDAmM5nKa```

Editer le fichier pour ajouter a la fin (attention à alignement):
```
  - name: YOUR_USER
    fullname: "Insert description here"
    class: "read-only"
    password: "HASH_TO_COPY"
```

#### Pousser la configuration

```
vagrant@server:~/ansible$ ansible-playbook -i inventories/hosts pb.juniper.users.yaml --vault-id ~/.vault_pass.txt
```
OUTPUT
```
TASK [Print the difference if exists] ***********************************************************************************************
ok: [vqfx] => {
    "response.diff_lines": [
        "",
        "[edit system login]",
        "+    user acorso {",
        "+        full-name \"Alexandre Corso\";",
        "+        uid 2006;",
        "+        class read-only;",
        "+        authentication {",
        "+            encrypted-password \"$6$GAMRvF.5$.yhfl02NtcTn03p.9Z/Mts0Rhhif0qibapCXdlMa8DvzfGidRknsFQRVH6VTMUyc4DST9.hWZoB5EjE0D3cdC/\"; ## SECRET-DATA",
        "+        }",
        "+    }",
    ]
}

PLAY RECAP **************************************************************************************************************************
vqfx                       : ok=7    changed=3    unreachable=0    failed=0

```

#### Verification 

```
vagrant@vqfx-re> show configuration system login user {YOUR_USER}
```
OUTPUT
```
full-name "Alexandre Corso";
uid 2006;
class read-only;
authentication {
    encrypted-password "$6$GAMRvF.5$.yhfl02NtcTn03p.9Z/Mts0Rhhif0qibapCXdlMa8DvzfGidRknsFQRVH6VTMUyc4DST9.hWZoB5EjE0D3cdC/"; ## SECRET-DATA
}
```

```
vagrant@server:~$ ssh {YOUR_USER}@10.1.2.1
Password:
```


### Etape 2: Configurer une session BGP

Avant
```
vagrant@vqfx-re> show bgp summary
BGP is not running
```

Le template est déjà present, il suffit de l'appliquer

```
vagrant@server:~/ansible$ ansible-playbook -i inventories/hosts pb.juniper.bgp.yaml --vault-id ~/.vault_pass.txt
```
OUTPUT
```
TASK [Print the difference if exists] ***********************************************************************************************
ok: [vqfx] => {
    "response.diff_lines": [
        "",
        "[edit protocols]",
        "+   bgp {",
        "+       group ipv4-transit-123 {",
        "+           type external;",
        "+           description \"IPV4 AS123\";",
        "+           local-address 10.1.2.1;",
        "+           import ipv4-as123;",
        "+           family inet {",
        "+               unicast;",
        "+           }",
        "+           peer-as 123;",
        "+           local-as 65500;",
        "+           neighbor 10.1.2.2;",
        "+       }",
        "+   }",
        "[edit]",
        "+  policy-options {",
        "+      policy-statement ipv4-as123 {",
        "+          then accept;"
        "+      }",
        "+  }"
    ]
}

PLAY RECAP **************************************************************************************************************************
vqfx                       : ok=7    changed=4    unreachable=0    failed=0
```

#### Verification

Session BGP UP
```
vagrant@vqfx> show bgp summary

```

Routes reçues
```
vagrant@vqfx> show route protocol bgp
```
OUTPUT
```
inet.0: 9 destinations, 9 routes (9 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1.0.0.0/24         *[BGP/170] 00:02:26, localpref 100
                      AS path: 123 I, validation-state: unverified
                    > to 10.1.2.2 via xe-0/0/0.0
2.0.0.0/24         *[BGP/170] 00:02:48, localpref 100
                      AS path: 123 I, validation-state: unverified
                    > to 10.1.2.2 via xe-0/0/0.0
```

### Etape 3: Modifier les imports

Ajouter des variables dans l'iventaire pour autoriser seulement 1 prefixe
```
vagrant@server:~/ansible$ nano inventories/group_vars/all/bgp_transit_filter.yaml
```

Le format est le suivant, on autorise seulement le prefix 2.0.0.0/24 a etre annonce par l'AS 123
```
---
transit_filtered_prefixes:
  123:
    ipv4:
    - 2.0.0.0/24
```

On pousse la modification via ansible
```
vagrant@server:~/ansible$ ansible-playbook -i inventories/hosts pb.juniper.bgp.yaml --vault-id ~/.vault_pass.txt
```
OUTPUT
```
TASK [Print the difference if exists] ***********************************************************************************************
ok: [vqfx] => {
    "response.diff_lines": [
        "",
        "[edit policy-options policy-statement ipv4-as123]",
        "+    term accepted {",
        "+        from {",
        "+            route-filter 2.0.0.0/24 exact;",
        "+        }",
        "+        then accept;",
        "+    }",
        "+    term other {",
        "+        then reject;",
        "+    }",
        "[edit policy-options policy-statement ipv4-as123]",
        "-    then accept;"
    ]
}
```

#### Verification

On peut verifier qu'il n'y a plus qu'une route acceptée
```
vagrant@vqfx-re> show route receive-protocol bgp 10.1.2.2
```
OUTPUT
```
inet.0: 9 destinations, 9 routes (8 active, 0 holddown, 1 hidden)
  Prefix      Nexthop        MED     Lclpref    AS path
* 2.0.0.0/24              10.1.2.2                                123 I
```

On peut aussi voir la route refusée
```
vagrant@vqfx-re> show route receive-protocol bgp 10.1.2.2 hidden
```
OUTPUT
```
inet.0: 9 destinations, 9 routes (8 active, 0 holddown, 1 hidden)
  Prefix      Nexthop        MED     Lclpref    AS path
  1.0.0.0/24              10.1.2.2                                123 I
```

### Etape 4: Supprimer un filtre

Find a solution :)


