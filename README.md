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

### 1.1.2 Déploiement du Gestionnaire de Configuration OAI (OAI-CM)

Le gestionnaire de configuration OAI-CM permet l’abonnement aux événements exposés par l'**AMF** et le **SMF**, les rendant disponibles à d'autres services.

- Référentiel initial en développement.
- Il s'intègre au RNIS via le **Core Network Wrapper Service** pour la capture des événements 5G Core.
- [GitLab OAI-CM](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-cm)

---

### 1.1.3 Déploiement des Composants RAN

Le projet **OAI-RAN** permet de compiler et déployer les composants gNB, CU-UP, UE et xApps pour l’interfaçage avec la plateforme MEC.

#### Étapes :

1. Cloner le dépôt :

```bash
git clone https://gitlab.eurecom.fr/oai/openairinterface5g gnbs
```

2. Compiler avec flexric et E2 support :

```bash
cd gnbs/cmake_targets/
./build_oai -I -w USRP --gNB --nrUE --build-e2 --ninja
```

- `-I` : installation des prérequis
- `-w USRP` : support de l’USRP
- `--build-e2` : agent E2 intégré pour xApps
- Un dossier `flexric` est généré dans `gnbs/openair2/E2AP/flexric`.

3. Lancer le nearRT-RIC :

```bash
cd flexric
./build/examples/ric/nearRT-RIC
```

4. Lancer RabbitMQ :

```bash
docker-compose -f docker-compose/docker-compose-ran.yaml up -d rabbitmq
```

5. Vérifier la file `rnis_xapp` dans l’interface RabbitMQ :

```
http://192.168.70.166:15672/#/queues
```

- Identifiants : `user` / `password`

6. Lancer le script Python de collecte des stats RAN :

```bash
python3 build/examples/xApp/python3/rnisxapp.py
```

---

## 1.2 Déploiement d’OAI-MEP

OAI-MEP permet l’enregistrement, la découverte et l’orchestration des services MEC via l’interface **MP1**.

### Composants :

- **Kong** : reverse proxy
- **DaRS** : API de découverte/enregistrement RESTful
- **oai-mep-gateway** : point d’entrée pour le trafic MP1

### Étapes :

1. Cloner le dépôt :

```bash
git clone https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-mep.git
cd oai-mep
```

2. Lancer le déploiement :

```bash
docker-compose -f docker-compose/docker-compose-mep.yaml up -d
```

3. Configurer `/etc/hosts` :

```bash
192.168.70.2 oai-mep.org
```

> Sinon, Kong ne pourra pas router correctement le trafic MP1 vers les services MEC hébergés.

### API MP1 :

| Méthode | Fonction                             |
|---------|--------------------------------------|
| POST    | Enregistrement d’un service MEC      |
| GET     | Découverte des services MEC          |
| GET     | Filtrage par type de service         |
| DELETE  | Suppression d’un service enregistré  |

---

## 1.3 Déploiement d’OAI-RNIS

OAI-RNIS fournit des **données RAN contextualisées** pour les services MEC, via une API REST conforme au standard ETSI MEC GS 012.

### Fonctionnement :

- Communication avec OAI-CM (core) et xApp (RAN) via mp2
- Exposition des données via MP1 REST (Northbound API et Notification Service)
- Auto-enregistrement au MEP au démarrage

### Modules internes :

| Module                    | Fonction                                                          |
|---------------------------|-------------------------------------------------------------------|
| Core Network Wrapper      | Événements réseau 5G via OAI-CM                                   |
| KPIs-xApp Service         | Métriques radio du RAN (via xApp E2)                              |
| Data Convergence Service  | Agrégation et traitement des données                             |
| Notification Service      | Envoi d’événements à tous les abonnés                            |
| Northbound API            | Interface RESTful pour la récupération directe                   |

### Déploiement :

1. Cloner le dépôt :

```bash
git clone https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-rnis.git
cd oai-rnis
```

2. Démarrer RNIS (voir documentation GitLab [32] pour détails supplémentaires).

### Utilisation des KPIs :

#### Méthode 1 : GET direct

```bash
curl -X 'GET' 'http://oai-mep.org/rnis/v2/queries/layer2_meas' -H 'accept: application/json'
```

#### Méthode 2 : Souscription automatique

1. Lancer l’application MEC Flask fournie :

```bash
python3 examples/example-mec-app.py
```

2. Démarrer un **UE OAI** pour générer des métriques.

3. Les KPIs sont alors visibles dans le **dashboard** ou récupérables manuellement par GET.

---

## 🔗 Références GitLab

- **OAI Core Network** : https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed
- **OAI-CM** : https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-cm
- **OAI-RAN** : https://gitlab.eurecom.fr/oai/openairinterface5g
- **OAI-MEP** : https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-mep
- **OAI-RNIS** : https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-rnis

---

✅ Une fois tous ces composants déployés et configurés correctement, vous disposez d’une **plateforme MEC 5G complète**, interopérable avec des **xApps O-RAN**, des **services de découverte dynamique**, et une **exposition des données radio en temps réel**.

