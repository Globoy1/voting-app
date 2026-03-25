Voting App : Déploiement Docker, Compose & Docker Swarm

Mini projet Docker a réalisé à deux afin de valider le module 3DOCK Supinfo

Ce projet modernise une application distribuée de vote (Python, Node.js, .NET, Redis, PostgreSQL) en utilisant Docker, Docker Compose et un déploiement complet sur un cluster Docker Swarm.

1. Aperçu du projet

Cette application est composée de 3 modules applicatifs :

- Vote : qui a pour technologie Python + Flask et dont le rôle est de permettre à l'utilisateur de voter et d'envoyer le résultat du vote vers Redis.

- Worker : qui a pour technologie .NET et dont le rôle est de transmettre les votes reçus de Redis vers PostgreSQL.

- Resut : qui a pour technologie Node.js et dont le rôle est d'affficher les résultats en temps réel avec les sockets.


2. Structure du projet 

voting-app/
│
├── vote/
│   ├── Dockerfile
│   ├── app.py
│   ├── requirements.txt
│   └── templates/
│
├── worker/
│   ├── Dockerfile
│   ├── Program.cs
│   └── Worker.csproj
│
├── result/
│   ├── Dockerfile
│   ├── server.js
│   └── views/
│       ├── index.html
│       └── app.js
│
├── docker-compose.yml
├── docker-compose-swarm.yml
├── Vagrantfile
├── .env
├── .env.swarm
└── README.md

3. Conteneurisation

Chaque module possède son Dockerfile dédié :

vote (Python)

- Image : python:3.11-slim

- User non-root (appuser)

- Utilisation d'un requirements.txt

- Expose le port 8080

worker (.NET)

- Multi-stage build (SDK → Runtime)

- User non-root

- Connexion via variables d’environnement

result (Node)

- Installation des dépendances avec npm ci --omit=dev

- Correction de la requête PostgreSQL

- Socket.io pour la mise à jour en temps réel

4. Docker Compose (développement local)

Pour lancer en local : docker compose up --build

Services disponibles :

- Vote : via l'adresse http://localhost:8080.

- Resut : via l'adresse http://localhost:8081.

- PostgreSQL : via localhost:5432

- Redis : via localhost:6379

Les volumes assurent la persistance des votes entre redémarrages.

5. Déploiement Docker Swarm

Nous utilisons un fichier dédié : docker-compose-swarm.yml

Spécificités :

- 2 replicas pour vote & result

- placement constraints : Redis + PostgreSQL → nœud manager et Applications → workers

- réseaux overlay : front (apps web) et back (worker, redis, postgres)
q
- healthchecks + restart_policy

- images poussées sur Docker Hub pour être disponibles sur les nœuds Swarm

6. Installation du cluster via Vagrant + VirtualBox

Le cluster contient 1 manager ( 192.168.99.100 ) et 2 workers ( 192.168.99.101 et 192.168.99.102 ).

Lancer les VMs : vagrant up

Vérifier les nœuds : se connecter d'abord au manager1 en ssh ( vagrant ssh manager1 ) et ensuite voir les noeuds ( docker node ls )

Résultat attendu :

ID    HOSTNAME   STATUS   AVAILABILITY   MANAGER STATUS
xxx   manager1   Ready    Active         Leader
xxx   worker1    Ready    Active
xxx   worker2    Ready    Active

7. Déploiement de la stack Swarm

I. Pousser les images sur Docker Hub

Sur la machine hôte :

- docker tag voting-app-vote:latest globoy12/votingapp_vote:latest

- docker tag voting-app-result:latest globoy12/votingapp_result:latest

- docker tag voting-app-worker:latest globoy12/votingapp_worker:latest

- docker login

- docker push globoy12/votingapp_vote:latest

- docker push globoy12/votingapp_result:latest

- docker push globoy12/votingapp_worker:latest


II. Déploiement sur manager1

vagrant ssh manager1

cd /vagrant

docker stack deploy -c docker-compose-swarm.yml voting-app

III. Vérifier les services

docker stack services voting-app

8. Tolérance aux pannes

Docker Swarm utilise le routing mesh, donc l’accès reste disponible même si un worker tombe.

Test de haute disponibilité? simulé un panne avec docker node update --availability drain worker1

Les services vote et result se redéploient automatiquement sur worker2.

9. Points de sécurité et bonnes pratiques Docker

