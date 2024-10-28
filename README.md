# tp-dns

# Nom & Prenom : ZAOUI Zakariae
# Manipulations et Analyses
## Visualisation

L'AFNIC (Association Française pour le Nommage Internet en Coopération) est l'organisation responsable de la gestion des noms de domaine en France, notamment les domaines de premier niveau.
En plus de son rôle technique, l’AFNIC soutient des initiatives pour réduire la fracture numérique et favoriser l’innovation au sein de l’écosystème Internet français.

On test sur DNSViz les domaines suivants

  - www.afnic.fr
  - ![www afnic fr-2024-10-14-15_12_02-UTC](https://github.com/user-attachments/assets/6cc7439b-bec7-4114-84ff-21281476a15d)
  - univ-amu.fr
  - ![univ-amu fr-2024-10-14-15_19_22-UTC](https://github.com/user-attachments/assets/e08cd247-a184-4e21-8515-a72f52aafd25)
  - Pour le domaine univ-amu.fr, plusieurs erreurs critiques ont été détectées lors des requêtes DNS. Les serveurs n'ont pas répondu aux requêtes en UDP, ce qui pourrait indiquer une surcharge, une mauvaise configuration ou des restrictions imposées par le pare-feu. Les tentatives répétées d'obtenir des réponses, notamment pour les enregistrements NS, confirment une connectivité potentiellement défaillante. Ces problèmes compromettent la sécurité et la disponibilité du domaine, le rendant vulnérable à des attaques, comme le spoofing DNS. Il est essentiel de vérifier la configuration des serveurs DNS et de s'assurer que les pare-feu n'interfèrent pas avec les requêtes, afin de garantir une résolution fiable des noms.
  - www.univ-amu.fr
  - ![www univ-amu fr-2024-10-14-14_16_50-UTC (1)](https://github.com/user-attachments/assets/97f2aeb1-680f-43a6-abf4-572117c4ae4a)
  - L'analyse des serveurs DNS du domaine univ-amu.fr a révélé qu'ils ne répondaient pas aux requêtes UDP, comme l'indique l'erreur : "The server(s) were not responsive to queries over UDP." Ce problème peut être causé par une surcharge du serveur, une mauvaise configuration ou une restriction au niveau du pare-feu. Conformément à la RFC 1035, section 4.2, l'absence de réponse aux requêtes DNS en UDP peut entraîner des problèmes de connectivité et de résolution de noms.
  - www.lis-lab.fr
  - L'analyse du domaine a montré une absence de réponse lors de la requête des enregistrements DNSKEY, révélant que le serveur à l'adresse IP 192.33.4.12 n'a pas répondu via UDP malgré quatre tentatives.

## Analyse

Analyse de DNSSEC avec dig +dnssec +short : Cette commande affiche une réponse simplifiée avec DNSSEC pour vérifier si les signatures sont présentes.

```cmd
dig www.afnic.fr +dnssec +short
dig univ-amu.fr +dnssec +short
dig www.univ-amu.fr +dnssec +short
dig www.lis-lab.fr +dnssec +short
```

EXEMPLE SCREEN SUR AFNIC :

![image](https://github.com/user-attachments/assets/07bec0af-cba4-488a-a9b4-8ef1744c5fe3)

Pour les domaines testés (univ-amu.fr, www.afnic.fr, www.univ-amu.fr, www.lis-lab.fr), les réponses sont obtenues sans erreurs et chaque domaine est correctement résolu. Cependant, aucune des réponses n'indique de validation DNSSEC, ce qui suggère que :
- Les domaines ou sous-domaines pourraient ne pas être signés avec DNSSEC.
- Si DNSSEC est implémenté, la validation n’a pas été réalisée dans ces réponses.

Trace des enregistrements DS avec +trace : La commande +trace permet de suivre la résolution DNS jusqu'aux serveurs de noms d'autorité, ce qui est utile pour détecter des problèmes de délégation ou d’enregistrement DS (Delegation Signer).

```cmd
dig DS www.afnic.fr +trace
dig DS univ-amu.fr +trace
dig DS www.univ-amu.fr +trace
dig DS www.lis-lab.fr +trace
```

# Analyse de la sécurité

Vérification l'enregistrement A (IPv4) et l'enregistrement AAAA (IPv6)

- Enregistrement A (IPv4) :

![image](https://github.com/user-attachments/assets/4d02eafe-93b8-4b9e-9527-ecb7449263b1)

L'enregistrement A pour univ-amu.fr est accessible, mais l'absence de signature DNSSEC le rend vulnérable aux attaques de type spoofing, car il n'est pas protégé par une authentification cryptographique.

- Enregistrement AAAA (IPv6) :

![image](https://github.com/user-attachments/assets/641b100e-b7f1-4fb6-853c-ef3fa9671111)

Aucune adresse IPv6 n'est renvoyée pour univ-amu.fr. À la place, on obtient un enregistrement SOA (Start of Authority) indiquant le serveur d’autorité pour la zone.

# Statut DNSSEC

- Accéder à la VM :

```bash
cd dns
vagrant ssh
```

![image](https://github.com/user-attachments/assets/6fefdca2-173b-45dd-8e14-81ed5fc69b46)

- Vérifier la configuration DNSSEC dans BIND :

```bash
cat /etc/bind/named.conf.options
```

![image](https://github.com/user-attachments/assets/7dcee302-1d7f-4f10-9166-a0e498667f10)

on trouve pas les options dnssec mais meme si ils ne sont pas spécifiées  DNSSEC peut être activé par défaut.

#  Mise en oeuvre de DNSSEC

## les Clés DNSSEC :

- on utilise dnssec-keygen pour générer une paire de clés de signature de zone (ZSK) et une clé de signature de clé (KSK) pour la zone .nuage.

```bash
sudo dnssec-keygen -a RSASHA256 -b 1024 -n ZONE nuage
sudo dnssec-keygen -a RSASHA256 -b 2048 -n ZONE -f KSK nuage
```

![image](https://github.com/user-attachments/assets/5fa84590-a44b-42fd-b4e4-189952f626e7)

Ces commandes produiront deux paires de fichiers pour ZSK et KSK dans le répertoire actuel. Les fichiers .key contiennent les clés publiques, et les fichiers .private contiennent les clés privées.

## Ajout des Clés DNSKEY au Fichier de Zone

On ouvre le fichier de configuration de la zone (par exemple, /etc/bind/for.nuage) et puis l ajout les enregistrements DNSKEY pour la ZSK et la KSK en incluant les fichiers .key générés.

![image](https://github.com/user-attachments/assets/3248ae75-4d42-45d0-87c7-2214d9cd00d7)

## Signature de la Zone et Validation DNSSEC

Utilisation de dnssec-signzone pour signer la zone avec les clés créées précédemment. Cette commande génère des enregistrements RRSIG et des enregistrements NSEC/NSEC3 pour sécuriser la zone.

```bash
sudo dnssec-signzone -A -o nuage -N INCREMENT -f /etc/bind/zones/db.nuage.signed /etc/bind/zones/db.nuage
```

![image](https://github.com/user-attachments/assets/f0d2cbba-c44f-48ef-9560-44f44d7f4c07)

La zone for.nuage est désormais signée avec DNSSEC, et un fichier signé (for.nuage.signed) a été généré.

## Mettre à jour le Fichier de Zone dans la Configuration de BIND

Dans le fichier de configuration de BIND:

```bash
zone "nuage" {
        type master;
        file "/etc/bind/for.nuage.signed";
 };
```

## Redémarrer BIND 

On redemarre BIND pour appliquer les nouvelles configurations :

```bash
sudo systemctl restart bind9
```

![image](https://github.com/user-attachments/assets/6c4f23e2-0dbe-40b5-b94f-3a567b97c953)

## Vérifier les Signatures avec dig

Pour vérifier que les enregistrements des serveurs webi sont correctement signés, on utilise les commandes dig pour voir les signatures DNSSEC :

```bash
dig @localhost web1.nuage +dnssec
```

![image](https://github.com/user-attachments/assets/bc55a43e-a816-4f79-935d-9e25c2a13242)

La zone nuage est désormais protégée par DNSSEC, et les enregistrements DNS comme web1.nuage sont correctement signés. Cela signifie que toute tentative de falsification ou de modification des enregistrements DNS pour cette zone sera détectable grâce aux signatures DNSSEC.



# Création et Sécurisation de la Zone Déléguée

## Ajout de la zone déléguée dans la zone principale nuage :

Dans le fichier de zone for.nuage, on ajoute une délégation pour preprod.nuage en pointant vers un serveur de noms ns.preprod.nuage.

```bash
preprod    IN NS    ns.preprod.nuage.
ns.preprod IN A     10.16.10.3
```

![image](https://github.com/user-attachments/assets/75abba4f-5c93-46c2-9b7f-f8b560e3b441)

## Création un fichier de zone pour preprod.nuage :

On cree un nouveau fichier de zone pour preprod.nuage,  for.preprod.nuage, avec les enregistrements pour web.preprod.nuage et le serveur de noms ns.preprod.nuage.

![image](https://github.com/user-attachments/assets/424e83b7-0201-4aa8-aa59-220d10df1271)

## Activation DNSSEC pour preprod.nuage :

On génére des clés DNSSEC pour la zone déléguée preprod.nuage :

```bash
sudo dnssec-keygen -a RSASHA256 -b 2048 -n ZONE preprod.nuage
sudo dnssec-keygen -a RSASHA256 -b 1024 -n ZONE -f KSK preprod.nuage
```

![image](https://github.com/user-attachments/assets/84682e5c-47f7-4063-b780-f16ba1609d30)

Inclure les clés DNSKEY dans for.preprod.nuage :

```bash
$INCLUDE "Kpreprod.nuage.+008+37597>.key"
$INCLUDE "Kpreprod.nuage.+008+01969>.key"
```

Signer la zone preprod.nuage :

```bash
sudo dnssec-signzone -A -o preprod.nuage -N INCREMENT -f /etc/bind/for.preprod.nuage.signed /etc/bind/for.preprod.nuage
```

![image](https://github.com/user-attachments/assets/00ad4696-df29-44f8-af10-3eae6e005399)

Mettre à jour la configuration de BIND pour la zone déléguée dans named.conf.local :

```bash
zone "preprod.nuage" {
  type master;
  file "/etc/bind/for.preprod.nuage.signed";
};
```

![image](https://github.com/user-attachments/assets/816acadd-43a9-407b-9598-cd8c75420fb3)

## Redémarrer le service BIND :

```bash
sudo systemctl restart bind9
```

## On Vérifie la configuration DNSSEC de preprod.nuage avec dig :

```bash
dig @localhost web.preprod.nuage +dnssec
```

![image](https://github.com/user-attachments/assets/ed5ad31f-c291-45c8-8f65-6b119d6ef16d)

# Limites de cette sécurisation et compléments pour une mise en œuvre complète

Limites de DNSSEC :

- Complexité de gestion des clés : La rotation régulière des clés (ZSK et KSK) est nécessaire pour maintenir la sécurité, ce qui ajoute de la complexité.
- Pas de confidentialité : DNSSEC garantit l'authenticité et l'intégrité des enregistrements DNS, mais les données restent en clair et ne sont pas chiffrées.
- Risque de mauvaise configuration : Une erreur dans la configuration ou la gestion des clés DNSSEC peut entraîner des pannes de résolution DNS, rendant les domaines inaccessibles.



