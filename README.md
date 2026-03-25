# Voting App : Déploiement avec Docker, Docker Compose et Docker Swarm

Projet pratique réalisé en binôme dans le cadre de la validation du module 3DOCK à SUPINFO.

L’objectif de ce projet est de conteneuriser puis déployer une application de vote distribuée reposant sur plusieurs technologies : Python, Node.js, .NET, Redis et PostgreSQL.  
Le travail couvre à la fois l’exécution locale avec Docker Compose et le déploiement distribué sur un cluster Docker Swarm.

1. Présentation générale du projet

L’application se compose de trois services métiers principaux, chacun ayant une responsabilité bien définie dans le traitement du vote :

- Vote
  Développé avec Python et Flask, ce service fournit l’interface de vote à l’utilisateur. Lorsqu’un choix est effectué, il est envoyé dans Redis.

- Worker  
  Développé en .NET, ce composant agit comme intermédiaire entre Redis et PostgreSQL. Il lit les votes stockés temporairement dans Redis puis les enregistre de manière persistante dans la base de données.

- Result  
  Développé avec Node.js, ce service récupère les données enregistrées et affiche les résultats en temps réel. La mise à jour dynamique est assurée grâce à Socket.io.

2. Organisation du projet

Le projet est structuré de manière à bien séparer les différents composants applicatifs et les fichiers d’orchestration :

voting-app/
│
├── result/
│   ├── node_modules/
│   ├── views/
│   │   ├── stylesheets/
│   │   ├── angular.min.js
│   │   ├── app.js
│   │   ├── index.html
│   │   └── socket.io.js
│   ├── .dockerignore
│   ├── .gitignore
│   ├── Dockerfile.result
│   ├── package-lock.json
│   ├── package.json
│   └── server.js
│
├── run/
│
├── vote/
│   ├── static/
│   ├── templates/
│   ├── .dockerignore
│   ├── .gitignore
│   ├── app.py
│   ├── Dockerfile.vote
│   └── requirements.txt
│
├── worker/
│   ├── bin/
│   ├── obj/
│   ├── .dockerignore
│   ├── .gitignore
│   ├── Dockerfile.worker
│   ├── Program.cs
│   └── Worker.csproj
│
├── .env
├── .env.swarm
├── .gitignore
├── docker-compose.swarm.yml
├── docker-compose.yml
├── README.md
├── Vagrantfile
├── voting-app.sln
└── worker-token

Cette structure facilite la maintenance, la compréhension du projet, ainsi que la séparation entre environnement local et environnement distribué.

3. Conteneurisation des services

Chaque service applicatif possède son propre Dockerfile, adapté à sa technologie et à ses besoins.

Service `vote` (Python)

Le service de vote repose sur :

- l’image de base `python:3.11-slim`
- un utilisateur non privilégié nommé `appuser`
- l’installation des dépendances via `requirements.txt`
- l’exposition du port 8080

Cette configuration permet d’obtenir une image légère, lisible et plus sécurisée.

Service `worker` (.NET)

Le composant worker utilise :

- un build multi-stage afin de séparer la phase de compilation et la phase d’exécution
- une exécution avec un utilisateur non-root
- des paramètres transmis par variables d’environnement

Cette approche réduit la taille finale de l’image et améliore les bonnes pratiques de déploiement.

Service `result` (Node.js)

Le service d’affichage des résultats repose sur :

- l’installation des dépendances avec `npm ci --omit=dev`
- l’intégration de Socket.io pour la mise à jour temps réel
- une logique de lecture des résultats depuis PostgreSQL

Ce composant permet d’observer immédiatement l’évolution des votes côté interface utilisateur.

4. Exécution avec Docker Compose en local

Pour un lancement en environnement local, nous utilisons Docker Compose.

Commande de démarrage :

Sur un terminal dans la racine du projet :

bash
docker compose up --build

Une fois les conteneurs lancés, les services sont accessibles comme suit :

- Vote: via l’adresse `http://localhost:8080`
- Result: via l’adresse `http://localhost:8081`
- PostgreSQL: via `localhost:5432`
- Redis: via `localhost:6379`

L’utilisation de volumes permet de conserver les données même après l’arrêt ou le redémarrage des conteneurs, ce qui garantit une meilleure continuité pendant les tests.

Resultats attendu sur docker

CONTAINER ID   NAME            CPU %     MEM USAGE / LIMIT     MEM %     NET I/O           BLOCK I/O        PIDS
ef91937a2e69   voting-vote     0.25%     77.59MiB / 7.684GiB   0.99%     2.31kB / 252B     34.5MB / 0B      3
0bc6eb7d9271   voting-result   0.07%     56.03MiB / 7.684GiB   0.71%     11.4kB / 10.8kB   53.6MB / 0B      11
e575cb70ad04   voting-worker   2.64%     30.47MiB / 7.684GiB   0.39%     133kB / 178kB     51.9MB / 0B      25
0b2edea1dbdd   voting-db       0.41%     36.63MiB / 7.684GiB   0.47%     110kB / 98.2kB    31.6MB / 422kB   8
79d7fad3421b   voting-redis    1.02%     6.598MiB / 7.684GiB   0.08%     81.7kB / 42.6kB   12.3MB / 0B      6

5. Déploiement avec Docker Swarm

Pour la partie orchestration distribuée, nous utilisons un fichier dédié nommé :

bash
docker-compose-swarm.yml

Dans cette version Swarm, plusieurs éléments ont été pensés pour correspondre à un déploiement sur cluster :

- 2 replicas pour les services vote et result
- contraintes de placement afin de réserver Redis et PostgreSQL au nœud manager
- répartition des services applicatifs sur les nœuds workers
- utilisation de réseaux overlay
- intégration de healthchecks
- ajout de restart policies
- publication des images sur Docker Hub afin que chaque nœud puisse les récupérer

Cette approche permet d’obtenir un déploiement plus robuste et plus proche d’un environnement réel.

6. Mise en place du cluster avec Vagrant et VirtualBox

Le cluster Docker Swarm utilisé dans ce projet est composé de trois machines virtuelles :

- manager1: `192.168.99.100`
- worker1: `192.168.99.101`
- worker2: `192.168.99.102`

Pour créer et démarrer l’ensemble des machines virtuelles :

bash
vagrant up

Pour vérifier l’état du cluster, il faut d’abord se connecter au manager1 :
docker stack services voting-app

bash
vagrant ssh manager1

Puis afficher la liste des nœuds :

bash
docker node ls

Le résultat attendu est de la forme suivante :

text
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
wtk6b19rmpnawn9a1f37neima *   manager1   Ready     Active         Leader           29.3.0
llptf5za88cbstktocgbq4x0h     worker1    Ready     Active                          29.3.0
vwetpprxvpgsussiej3rn174t     worker2    Ready     Active                          29.3.0


Cette étape confirme que le cluster est bien formé et que les nœuds workers ont correctement rejoint le manager.

7. Déploiement de la stack dans Swarm

I. Publication des images sur Docker Hub

Depuis la machine hôte, il faut d’abord taguer puis pousser les images afin qu’elles soient accessibles depuis les machines du cluster.

```bash
docker tag voting-app-vote:latest globoy12/votingapp_vote:latest
docker tag voting-app-result:latest globoy12/votingapp_result:latest
docker tag voting-app-worker:latest globoy12/votingapp_worker:latest
docker login
docker push globoy12/votingapp_vote:latest
docker push globoy12/votingapp_result:latest
docker push globoy12/votingapp_worker:latest
```

II. Déploiement depuis le manager

Connexion au nœud manager :

```bash
vagrant ssh manager1
```

Placement dans le dossier du projet partagé :

```bash
cd /vagrant
```

Déploiement de la stack :

```bash
docker stack deploy -c docker-compose-swarm.yml voting-app
```

III. Vérification des services

Pour contrôler que les services sont bien déployés :

```bash
docker stack services voting-app
```

Cette commande permet de voir le nombre de réplicas, les ports publiés, ainsi que l’état général des services.

8. Gestion de la haute disponibilité et simulation de panne

L’un des intérêts majeurs de Docker Swarm est sa capacité à maintenir les services disponibles même lorsqu’un nœud devient indisponible.

Dans notre cas, le routing mesh permet de continuer à accéder aux services même si un worker rencontre une panne.

Pour simuler l’indisponibilité d’un nœud, on peut utiliser la commande suivante :

```bash
docker node update --availability drain worker1
```

Avec cette opération, le nœud `worker1` est retiré de la planification active. Les tâches qui y étaient exécutées sont alors automatiquement redéployées sur les autres nœuds disponibles, ici notamment sur `worker2`.

Cela permet de démontrer concrètement le mécanisme de tolérance aux pannes fourni par Swarm.

9. Sécurité et bonnes pratiques Docker

Dans ce projet, plusieurs bonnes pratiques ont été appliquées afin d’améliorer la qualité du déploiement :

- utilisation d’utilisateurs non-root dans les conteneurs applicatifs
- séparation claire entre build et runtime pour le service .NET
- réduction de la taille des images grâce à des images de base légères
- usage de variables d’environnement pour la configuration
- persistance des données via des volumes dédiés
- séparation logique des flux réseau avec des réseaux distincts
- réplication des services critiques côté interface applicative
- déploiement distribué dans un cluster Swarm pour améliorer la disponibilité

Ces choix permettent d’obtenir une architecture plus propre, plus maintenable et plus proche des standards de déploiement modernes.

10. Conclusion

Ce projet nous a permis de mettre en pratique plusieurs concepts essentiels liés à la conteneurisation et à l’orchestration :

- création d’images Docker personnalisées
- exécution multi-services avec Docker Compose
- mise en place d’un cluster Docker Swarm
- déploiement distribué avec réplication
- simulation de panne et observation du redéploiement automatique

Au-delà du simple fonctionnement de l’application, ce travail illustre également une démarche d’industrialisation, avec une architecture distribuée, des services découplés et des outils adaptés au déploiement moderne.
