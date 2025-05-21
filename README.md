# MEC
This repository provides a step-by-step guide for implementing OAI-MEC in a 5G network.

# Déploiement du 5G-MEC avec OpenAirInterface (OAI)


# Déploiement des composants Core et RAN - OAI 5G

## 1.1 Déploiement des composants Core et RAN

---

### 1.1.1 Déploiement du Réseau Central OAI 5G

Nous optons pour la version `develop` du déploiement de base du réseau central OAI 5G avec `oai-spgwu-UPF`. Le code source du projet OAI-CN est accessible sur le GitLab officiel de l'OAI [[29]](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed).

Le déploiement du cœur du réseau s'appuie sur des images Docker de chaque composant fournies par OAI. Après récupération des images, il est nécessaire de modifier et configurer le fichier `docker-compose.yaml` afin de permettre une communication réussie et stable entre les différents composants.

**Important :** Par défaut, le trafic des conteneurs Docker n'est pas routé vers l'extérieur. Il est donc impératif d’activer le **transfert d’IP** à l’aide des commandes suivantes :

```bash
sudo sysctl net.ipv4.conf.all.forwarding=1
sudo iptables -P FORWARD ACCEPT
```

Ensuite, les **informations d'identification des abonnés (SIM)** pour les UE (User Equipment) OAI doivent être ajoutées dans le fichier `oai_db.sql2`.

Le déploiement du réseau central est simplifié par un **script Python** (wrapper autour de `docker-compose`), permettant de démarrer l’ensemble du cœur du réseau avec la commande :

```bash
python3 core-network.py --type start-basic
```

---

### 1.1.2 Déploiement du Gestionnaire de Configuration OAI (OAI-CM)

Le **gestionnaire de configuration OAI (OAI-CM)** a pour rôle de s’abonner aux événements générés par les fonctions du réseau principal, et de les exposer à d'autres composants du système via des interfaces spécifiques.

À l’heure actuelle, le dépôt est en phase initiale de développement, mais permet déjà de recevoir des notifications d’événements en provenance de **l'OAI-AMF** et **l'OAI-SMF** via les points d’extrémité Northbound suivants :

- `/subscribe/notification/amf`  
- `/subscribe/notification/smf`

Des informations supplémentaires, ainsi que le code source du gestionnaire de configuration, sont disponibles sur le GitLab officiel de l'OAI [[27]](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-oai-cm).

---

### 1.1.3 Déploiement des Composants RAN

Le code source du projet **OAI-RAN** est accessible sur le GitLab officiel de l’OAI [[30]](https://gitlab.eurecom.fr/oai/openairinterface5g).

#### 1. Clonage du dépôt

```bash
git clone https://gitlab.eurecom.fr/oai/openairinterface5g gnbs
```

#### 2. Compilation avec flexRIC

Depuis le répertoire cloné :

```bash
cd gnbs/cmake_targets/
./build_oai -I -w USRP --gNB --nrUE --build-e2 --ninja
```

- `-I` : installe les prérequis.
- `-w USRP` : sélectionne le support USRP via `liboai_usrpdevif.so`.
- `--gNB` et `--nrUE` : construisent les exécutables `nr-softmodem`, `nr-cuup`, `nr-uesoftmodem`, etc.
- `--build-e2` : active l’agent E2.
- `--ninja` : compilation rapide avec ninja.

Un dossier `flexric` est généré dans `gnbs/openair2/E2AP/flexric`.

#### 3. Configuration du gNB

Modifier le fichier `gnb.sa.band78.fr1.106PRB.usrpb210.conf` pour qu’il soit cohérent avec les informations SIM du cœur de réseau.

#### 4. Lancement des composants

```bash
cd flexric
./build/examples/ric/nearRT-RIC
```

Lancement de RabbitMQ :

```bash
docker-compose -f docker-compose/docker-compose-ran.yaml up -d rabbitmq
```

Exécution du xApp :

```bash
python3 build/examples/xApp/python3/rnisxapp.py
```

Vérifier via l’interface :

```
http://192.168.70.166:15672/#/queues
```

- **Identifiants** :  
  - Utilisateur : `user`  
  - Mot de passe : `password`

Vérifiez la présence de la file `rnis_xapp`.

---

## 1.2 Déploiement d'OAI-MEP

L’architecture d’**OAI-MEP** comprend une passerelle (gateway) qui agit comme **point d’entrée** pour rediriger le trafic `mp1` vers les services appropriés. Dans cette configuration, **Kong** est utilisé comme **proxy inverse**.

Le **Service de Découverte et d'Enregistrement (DaRS)** constitue le cœur de la plateforme MEC. Il expose une API REST sur l’interface `mp1`, permettant de :

- **Enregistrer** un service MEC (requête `POST`)
- **Découvrir** les services MEC hébergés (requête `GET`)
- **Filtrer** les services par type (requête `GET`)
- **Supprimer** un service hébergé (requête `DELETE`)

#### 1. Clonage du dépôt

```bash
git clone https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-mep.git
```

#### 2. Déploiement avec Docker Compose

```bash
docker-compose -f docker-compose/docker-compose-mep.yaml up -d
```

#### 3. Configuration DNS ou /etc/hosts

Il est impératif de configurer le **FQDN** de l’instance `oai-mep-gateway` (ou `kong`) pour permettre le routage du trafic `mp1`. Ajouter la ligne suivante dans le fichier `/etc/hosts` :

```bash
192.168.70.2 oai-mep.org
```

> Sans cette configuration, le MEP ne pourra pas router correctement les requêtes vers les services hébergés.

Des détails supplémentaires sont disponibles sur le GitLab OAI [[27]](https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-mep).

---

## 1.3 Déploiement d'OAI-RNIS

Le **Radio Network Information Service (RNIS)** est un service MEC permettant d’exposer les informations radios spécifiques à un utilisateur, conformément à la spécification **ETSI MEC GS 012**, adaptée à l’architecture 5G SA.

Le RNIS repose sur plusieurs modules appelés `rnisApp` ou `rnisService` :

- **Northbound API**
- **Data Convergence Service**
- **Notification Service**
- **Core Network Wrapper Service**
- **KPIs-xApp Service**

### Fonctionnement

Le RNIS extrait les données du réseau 3GPP via l’interface `mp2` et les rend accessibles aux applications MEC via `mp1`.

- Le **Core Network Wrapper Service** reçoit les événements du cœur de réseau 5G exposés par **OAI-CM**.
- Le **KPIs-xApp Service** consomme des métriques radio exposées par une **xApp O-RAN spécifique (rnis-xApp)**.
- Le **Data Convergence Service** traite et fusionne les données, puis déclenche des événements.
- La **Northbound API** expose les données via HTTP REST.
- Le **Notification Service** diffuse les événements aux abonnés (clients) intéressés.

### Déploiement

#### 1. Clonage du dépôt

```bash
git clone https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-rnis.git
```

Le RNIS s’enregistre automatiquement auprès du MEP à son démarrage.

#### 2. Accès aux KPIs radio

**Méthode 1 — Requête GET manuelle :**

```bash
curl -X 'GET' 'http://oai-mep.org/rnis/v2/queries/layer2_meas' -H 'accept: application/json'
```

**Méthode 2 — Souscription via une application MEC :**

Une application MEC basée sur Flask est fournie pour s’abonner automatiquement aux KPIs exposés par le RNIS :

```bash
python3 examples/example-mec-app.py
```

Le démarrage de l’**UE** est requis pour générer des KPIs significatifs. Les données peuvent ensuite être consultées via l’application MEC ou la commande `curl` précédente.

Plus d’informations sont disponibles [[32]](https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-rnis).

---

