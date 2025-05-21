# MEC
This repository provides a step-by-step guide for implementing OAI-MEC in a 5G network.

# 📡 Déploiement du 5G-MEC avec OpenAirInterface (OAI)

Ce document fournit un guide complet pour déployer une infrastructure 5G MEC (Multi-access Edge Computing) basée sur les composants OpenAirInterface (OAI), incluant le réseau central (Core), le RAN, le gestionnaire de configuration (OAI-CM), la plateforme MEC (OAI-MEP) et le service RNIS.

---

## 1.1 Déploiement des composants Core et RAN

### 1.1.1 Déploiement du Réseau Central OAI 5G

Ce guide explique le déploiement de la version `develop` du cœur de réseau OAI 5G, en utilisant notamment le composant `oai-spgwu-UPF`.

Le projet **OAI-CN** (Core Network) est disponible sur le GitLab officiel : [https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed)

#### Étapes de déploiement :

1. **Récupération des images Docker** :
   OAI fournit des images Docker pré-construites pour chaque composant (AMF, SMF, UPF, etc.). Après extraction, chaque image doit être référencée dans le fichier `docker-compose.yaml`.

2. **Modification du fichier `docker-compose.yaml`** :
   Configurez ce fichier pour garantir une communication stable entre tous les composants du réseau.

3. **Activer le transfert IP** :
   Par défaut, le trafic des conteneurs Docker n'est pas redirigé vers l'extérieur. Activez le transfert IP avec les commandes suivantes :
   ```bash
   sudo sysctl net.ipv4.conf.all.forwarding=1
   sudo iptables -P FORWARD ACCEPT
   ```

4. **Configurer les abonnés (SIM)** :
   Ajoutez les détails d'identification de l'UE OAI dans le fichier `oai_db.sql2`, qui contient les paramètres SIM tels que l'IMSI, la clé, l’opérateur, etc.

5. **Lancement du cœur de réseau** :
   Utilisez le script Python fourni pour simplifier le démarrage du réseau :
   ```bash
   python3 core-network.py --type start-basic
   ```

   Ce script agit comme une surcouche de `docker-compose` et `docker` afin de faciliter le déploiement.

---

> **Remarques** :
> - Assurez-vous d'avoir `docker`, `docker-compose` et `python3` installés sur votre machine.
> - Si nécessaire, autorisez les ports requis par les composants (AMF, SMF, UPF, etc.) via votre pare-feu.
> - Il est recommandé d’utiliser une machine avec un noyau Linux récent et le support des namespaces réseau pour une compatibilité optimale.
---

## 1.1.2 Déploiement du Gestionnaire de Configuration OAI (OAI-CM)

Le Gestionnaire de Configuration OAI (OAI-CM) joue un rôle essentiel dans l'architecture 5G-MEC avec OpenAirInterface. Il permet de souscrire aux événements générés par les fonctions du réseau central, en particulier l'AMF (Access and Mobility Function) et le SMF (Session Management Function), et d’exposer ces événements à d’autres composants du système MEC.

### Fonctionnalités principales :
- **Souscription aux notifications d'événements AMF** via le point de terminaison :
  ```
  /subscribe/notification/amf
  ```
- **Souscription aux notifications d'événements SMF** via le point de terminaison :
  ```
  /subscribe/notification/smf
  ```

Ces endpoints northbound permettent à des composants tiers (comme le RNIS, un orchestrateur MEC ou des xApps) de recevoir des événements temps réel liés aux états de l’UE, aux sessions PDU, ou autres événements de signalisation pertinents.

> ⚠️ À ce jour, le dépôt OAI-CM est au stade initial de développement et prend uniquement en charge les abonnements aux événements provenant de l’AMF et du SMF d’OAI.

### Références :
- Code source et informations détaillées disponibles sur le GitLab OAI : [https://gitlab.eurecom.fr/oai](https://gitlab.eurecom.fr/oai) (voir référentiel `oai-cn5g-smf` ou `oai-cn5g-amf`, selon les mises à jour récentes).

## 1.1.3 Déploiement des Composants RAN

Ce guide décrit le processus de déploiement des composants **RAN** d’OpenAirInterface (OAI), en particulier l’intégration avec **oai-mep** sur **n'importe quelle branche de `oai-gnb`**. Le projet source est disponible ici :  
👉 [OAI RAN GitLab](https://gitlab.eurecom.fr/oai/openairinterface5g)

---

### 📦 Clonage du projet OAI-RAN

Commencez par cloner le dépôt :

```bash
git clone https://gitlab.eurecom.fr/oai/openairinterface5g gnbs
cd gnbs
```

---

### ⚙️ Compilation avec `flexric` et support USRP

Rendez-vous dans le répertoire de compilation et exécutez :

```bash
cd cmake_targets/
./build_oai -I -w USRP --gNB --nrUE --build-e2 --ninja
```

#### Détails des options :

- `-I` : installe les dépendances nécessaires (à faire une seule fois ou après modification des prérequis).
- `-w USRP` : active le support pour la tête radio **USRP**, via la bibliothèque `oai device`.
- `--gNB` : construit l'exécutable **`nr-softmodem`** (gNB).
- `--nrUE` : construit l'exécutable **`nr-uesoftmodem`** (UE).
- `--build-e2` : compile et intègre l’agent **E2** au gNB.
- `--ninja` : utilise **ninja** pour accélérer la compilation.

> 🔧 Note : `liboai_device.so` est lié dynamiquement à `liboai_usrpdevif.so` (USRP) pour l'exécution.

---

### 🏗️ Arborescence et flexRIC

Après compilation :

- Un dossier `flexric` est généré automatiquement :
  ```
  gnbs/openair2/E2AP/flexric
  ```

Des instructions complémentaires pour compiler les programmes supplémentaires de `flexric` sont disponibles dans la référence [31].

---

### 📝 Configuration du gNB

Le fichier de configuration du gNB (`gnb.sa.band78.fr1.106PRB.usrpb210.conf`) doit être **cohérent avec les informations SIM** déjà configurées dans la base de données du réseau central (`oai_db.sql2`).

---

### 🚀 Lancement des composants

#### Démarrage du gNB :

Suivre les instructions de lancement fournies dans le dépôt GitLab principal :  
👉 [OAI RAN GitLab](https://gitlab.eurecom.fr/oai/openairinterface5g)

#### Lancement de near-RT RIC :

```bash
cd flexric
./build/examples/ric/nearRT-RIC
```

---

### 📨 RabbitMQ & xApp Python

#### Démarrage de RabbitMQ :

```bash
docker-compose -f docker-compose/docker-compose-ran.yaml up -d rabbitmq
```

#### Lancement du xApp Python (récupération des statistiques RAN) :

```bash
python3 build/examples/xApp/python3/rnisxapp.py
```

---

### ✅ Vérification via interface RabbitMQ

Accédez à l’interface web de gestion de RabbitMQ :

🔗 [http://192.168.70.166:15672/#/queues](http://192.168.70.166:15672/#/queues)

**Identifiants :**

- **User** : `user`
- **Password** : `password`

> La présence d’une file d’attente nommée `rnis_xapp` confirme que toutes les étapes précédentes ont été réalisées avec succès.

---

### 📬 À propos de RabbitMQ

RabbitMQ est un **middleware de messagerie** implémentant le protocole **AMQP (Advanced Message Queuing Protocol)**.  
Il fonctionne sur un modèle "file d’attente", où les **producteurs** envoient des messages à une file, et les **consommateurs** les récupèrent de cette file.

---

**Références** :

- [30] OAI RAN GitLab : https://gitlab.eurecom.fr/oai/openairinterface5g  
- [31] Documentation flexRIC : *https://gitlab.eurecom.fr/mosaic5g/flexric*


## 1.2 Déploiement d’OAI-MEP

Ce guide décrit le processus de déploiement de la plateforme **OAI-MEP** (OpenAirInterface MEC Platform), un composant essentiel de l’architecture MEC (Multi-access Edge Computing) dans l’environnement OAI.

---

### 🧱 Architecture OAI-MEP

L’architecture d’OAI-MEP est composée des éléments suivants :

- **OAI-MEP Gateway** : agit en tant que **point d’entrée**, redirigeant le trafic **MP1** vers les services MEC appropriés.
- **Kong** : joue le rôle de **proxy inverse** permettant de router le trafic.
- **DaRS (Discovery and Registration Service)** : cœur de la plateforme MEC, il expose une **API REST MP1** qui permet :
  - `POST` : **enregistrement d’un service MEC**
  - `GET` : **découverte de tous les services MEC**
  - `GET` avec filtre : **découverte par type de service**
  - `DELETE` : **suppression d’un service enregistré**

---

### 📦 Clonage du projet

Cloner le dépôt Git du projet :

```bash
git clone https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-mep.git
cd oai-mep
```

---

### 🚀 Déploiement avec Docker Compose

Utiliser la commande suivante pour lancer tous les services nécessaires :

```bash
docker-compose -f docker-compose/docker-compose-mep.yaml up -d
```

---

### ⚠️ Configuration du DNS ou `/etc/hosts`

Pour assurer le bon fonctionnement du routage MP1 via Kong, il est **obligatoire de configurer le FQDN** `oai-mep.org` dans votre système.

Ajouter la ligne suivante dans votre fichier `/etc/hosts` :

```bash
192.168.70.2 oai-mep.org
```

> ❗ Si cette configuration n’est pas faite, **OAI-MEP ne pourra pas router correctement le trafic MP1** vers les services MEC hébergés.

---

### 🌐 API MP1 – Fonctionnalités

L'interface **MP1** exposée par **DaRS** permet l’interaction avec les services MEC :

| Méthode | Fonction                                        | Exemple d'utilisation          |
|---------|-------------------------------------------------|--------------------------------|
| POST    | Enregistrement d’un service MEC                 | `curl -X POST <url>`           |
| GET     | Découverte de tous les services                 | `curl -X GET <url>`            |
| GET     | Découverte filtrée par type                     | `curl -X GET <url>?type=xyz`   |
| DELETE  | Suppression d’un service enregistré             | `curl -X DELETE <url>/<id>`    |

> Vous pouvez tester ces appels via `curl`, Postman ou toute autre interface REST.

---

### 📚 Référence

- [27] GitLab OAI-MEP : [https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-mep](https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-mep)

---

✅ Une fois le service lancé et le nom de domaine correctement configuré, la plateforme OAI-MEP est prête à accueillir et orchestrer vos services MEC via l’interface MP1.


## 1.3 Déploiement d’OAI-RNIS

OAI-RNIS (Radio Network Information Service) est un composant clé de l'architecture MEC, conçu pour fournir des **données radio spécifiques à un utilisateur**, en s’appuyant sur les spécifications **ETSI MEC GS 012** adaptées à une implémentation **5G Standalone (SA)**.

---

### 🧱 Architecture Fonctionnelle

OAI-RNIS repose sur plusieurs modules appelés **rnisApp** ou **rnisService**, dont les principaux rôles sont les suivants :

- **Northbound API** : Interface RESTful exposant les données RNIS.
- **Data Convergence Service** : Fusionne les données provenant des sources mp2 (réseau 3GPP).
- **Notification Service** : Envoie les événements RNIS aux services abonnés via HTTP.
- **Core Network Wrapper Service** : Consomme les événements du cœur de réseau 5G via **OAI-CM**.
- **KPIs-xApp Service** : Récupère les métriques radio depuis l’agent E2 de la xApp RAN.

---

### 📦 Clonage du projet

Le projet OAI-RNIS peut être cloné avec la commande suivante :

```bash
git clone https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-rnis.git
cd oai-rnis
```

---

### 🚀 Fonctionnement et Intégration

- RNIS extrait les données du réseau 3GPP via l’interface **mp2**.
- Ces données sont ensuite exposées via l’interface **mp1**.
- Au démarrage, **OAI-RNIS enregistre automatiquement** son service auprès de **OAI-MEP** via un appel `POST` mp1.

---

### 📡 Modules Clés

| Module                      | Fonction                                                                 |
|----------------------------|--------------------------------------------------------------------------|
| Core Network Wrapper       | Récupère les événements 5G exposés par OAI-CM                            |
| KPIs-xApp Service          | Collecte les statistiques RAN via xApp (ex : `rnis-xApp`)                |
| Data Convergence Service   | Agrège les informations collectées et déclenche les événements RNIS     |
| Notification Service       | Envoie les données aux abonnés par HTTP                                 |
| Northbound API             | Expose les informations RNIS en REST via mp1                            |

> ⚙️ L’architecture est **évolutive** en fonction du nombre de nœuds ou d’applications MEC à interfacer.

---

### 📊 Récupération des KPIs

Deux méthodes permettent à une application MEC d’accéder aux KPIs via MP1 :

#### ✅ Méthode 1 : Requête GET directe

Utiliser la commande suivante :

```bash
curl -X 'GET' 'http://oai-mep.org/rnis/v2/queries/layer2_meas' -H 'accept: application/json'
```

#### 📥 Méthode 2 : Souscription avec envoi automatique des KPIs

Une **application MEC** écrite avec **Python Flask** est fournie pour consommer automatiquement les KPIs :

```bash
python3 examples/example-mec-app.py
```

> ⚠️ Il est **nécessaire de démarrer un UE (User Equipment)** pour obtenir des données KPI valides.

Les KPIs peuvent ensuite être :
- Consultés via le **dashboard web de l’application MEC**.
- Ou récupérés manuellement par GET :

```bash
curl -X 'GET' 'http://oai-mep.org/rnis/v2/queries/layer2_meas' -H 'accept: application/json'
```

---

### 🌐 Interfaces RNIS

| Interface | Description                                                 |
|-----------|-------------------------------------------------------------|
| mp1       | Interface RESTful pour les consommateurs MEC                |
| mp2       | Interface d’intégration vers le réseau 3GPP (via OAI-CM / RAN) |

---

### 📚 Référence

- [32] GitLab OAI-RNIS : https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-rnis

---

✅ Une fois lancé, OAI-RNIS devient une brique MEC essentielle pour fournir des **indicateurs radio précis** à des applications de périphérie dynamiques et intelligentes.
