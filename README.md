# MEC
This repository provides a step-by-step guide for implementing OAI-MEC in a 5G network.

# Déploiement du 5G-MEC avec OpenAirInterface (OAI)

## 1.1 Déploiement des Composants Core et RAN

### 1.1.1 Déploiement du Réseau Central OAI 5G

Nous utilisons la branche `develop` du projet OAI-CN disponible ici :  
🔗 [https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed)

Chaque composant est fourni sous forme d’image Docker. Le fichier `docker-compose.yaml` doit être configuré pour permettre une communication fluide entre les composants.

Activez le routage IP sur l’hôte :
```bash
sudo sysctl net.ipv4.conf.all.forwarding=1
sudo iptables -P FORWARD ACCEPT
```

Ajoutez les informations SIM de l’UE OAI dans `oai_db.sql2`.

Lancez le cœur du réseau avec :
```bash
python3 core-network.py --type start-basic
```

---

### 1.1.2 Déploiement du Gestionnaire de Configuration OAI (OAI-CM)

OAI-CM permet de s'abonner aux événements d’AMF et SMF via :

- `/subscribe/notification/amf`
- `/subscribe/notification/smf`

🔗 Référentiel GitLab : [https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-config](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-config)

---

### 1.1.3 Déploiement des Composants RAN

🔗 Code source RAN : [https://gitlab.eurecom.fr/oai/openairinterface5g](https://gitlab.eurecom.fr/oai/openairinterface5g)

1. Clonez le dépôt :
```bash
git clone https://gitlab.eurecom.fr/oai/openairinterface5g gnbs
cd gnbs/cmake_targets/
```

2. Compilez avec :
```bash
./build_oai -I -w USRP --gNB --nrUE --build-e2 --ninja
```

- `-I` : installe les prérequis
- `-w USRP` : support USRP
- `--gNB`, `--nrUE` : construit les exécutables gNB et UE
- `--build-e2` : active l’agent E2
- `--ninja` : compilation rapide

3. Le dossier `flexric` est généré sous `gnbs/openair2/E2AP/flexric`. Plus d'infos ici :  
🔗 [https://gitlab.eurecom.fr/oai/flexric](https://gitlab.eurecom.fr/oai/flexric)

4. Lancez le nearRT-RIC :
```bash
cd flexric
./build/examples/ric/nearRT-RIC
```

5. Démarrez RabbitMQ :
```bash
docker-compose -f docker-compose/docker-compose-ran.yaml up -d rabbitmq
```

6. Exécutez le xApp RNIS :
```bash
python3 build/examples/xApp/python3/rnisxapp.py
```

7. Vérifiez la file RabbitMQ `rnis_xapp` :
- Accès : [http://192.168.70.166:15672/#/queues](http://192.168.70.166:15672/#/queues)
- Identifiants : `user | password`

---

## 1.2 Déploiement d’OAI-MEP

🔗 Dépôt GitLab : [https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-mep](https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-mep)

1. Clonez le dépôt :
```bash
git clone https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-mep.git
```

2. Lancez le MEP :
```bash
docker-compose -f docker-compose/docker-compose-mep.yaml up -d
```

3. Ajoutez cette ligne dans `/etc/hosts` :
```
192.168.70.2 oai-mep.org
```

Le Service DaRS fournit les opérations suivantes sur l’interface mp1 :

- `POST` : enregistrer un service
- `GET` : découvrir/filtrer des services
- `DELETE` : supprimer un service

---

## 1.3 Déploiement d’OAI-RNIS

🔗 Référentiel RNIS : [https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-rnis](https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-rnis)

1. Clonez le dépôt :
```bash
git clone https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-rnis.git
```

Le RNIS collecte les données radio via mp2 et les expose via mp1 à d'autres services MEC.

Modules clés :

- **Core Network Wrapper Service** : consomme les événements du Core via OAI-CM
- **KPIs-xApp Service** : collecte les métriques radio depuis une xApp O-RAN
- **Data Convergence Service** : fusionne et traite les données
- **Northbound API** et **Service de notification** : expose les KPIs aux applications clientes

---

### Exploitation des KPIs RAN

- **Récupération directe via GET** :
```bash
curl -X 'GET' 'http://oai-mep.org/rnis/v2/queries/layer2_meas' -H 'accept: application/json'
```

- **Souscription via une application MEC Flask** :
```bash
python3 examples/example-mec-app.py
```

> Assurez-vous que l’UE est actif pour obtenir des KPIs valides.

---

## Références GitLab

- OAI-CN5G : [https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed)  
- OAI-CM : [https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-config](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-config)  
- OAI-RAN : [https://gitlab.eurecom.fr/oai/openairinterface5g](https://gitlab.eurecom.fr/oai/openairinterface5g)  
- flexRIC : [https://gitlab.eurecom.fr/oai/flexric](https://gitlab.eurecom.fr/oai/flexric)  
- OAI-MEP : [https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-mep](https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-mep)  
- OAI-RNIS : [https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-rnis](https://gitlab.eurecom.fr/oai/orchestration/oai-mec/oai-rnis)

