# TP4 : Docker et Docker Compose, application MERN

**Étudiant :** Aboubacar Sidiki Niaré
**Niveau :** 3ème année Licence en Informatique appliquée à la Gestion, parcours Business Intelligence (LIG)
**Groupe :** 5B
**Établissement :** ISG de Sousse, Département Informatique
**Matière :** Introduction au DevOps
**Enseignante TD :** Mme Wiem KBAIER
**Année universitaire :** 2026-2027

---

## 1. Introduction

Dans ce TP, j'ai containerisé une application MERN (MongoDB, Express, React, Node.js) avec Docker, puis j'ai orchestré les trois services avec Docker Compose. J'ai ensuite automatisé le build et le déploiement avec un pipeline Jenkins. L'objectif était de comprendre comment une application composée de plusieurs parties peut être lancée de façon identique sur n'importe quelle machine.

## 2. Structure du projet

```
MyAppReact/
├── Client/              Application React (Vite) + Dockerfile
├── Server/              API Express + Dockerfile
├── docker-compose.yml   Orchestration des 3 services
├── Jenkinsfile          Pipeline Jenkins
├── captures/            Captures d'écran
└── README.md
```

Dépôt GitHub (fork) : https://github.com/aboubacar240031-pixel/MyAppReact

## 3. Clonage du projet

J'ai cloné le dépôt fourni dans le sujet :

```bash
git clone https://github.com/wiem-kb/MyAppReact.git
```

Le dossier contient deux parties : `Client` (React) et `Server` (API Express, qui écoute sur le port 9000). Comme je n'ai pas les droits d'écriture sur le dépôt d'origine, j'ai créé un fork sur mon compte GitHub pour pouvoir y pousser mon travail et le relier à Jenkins.

## 4. Les Dockerfiles

### Serveur (`Server/Dockerfile`)

```dockerfile
FROM node:lts-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 9000
CMD ["npm", "start"]
```

J'utilise l'image légère `node:lts-alpine`. Je copie d'abord les `package*.json` pour que Docker garde en cache l'étape `npm install` tant que les dépendances ne changent pas. Puis je copie le code, j'expose le port 9000 et je démarre avec `npm start`. J'ai aussi ajouté un `.dockerignore` pour ne pas copier `node_modules` dans l'image.

```bash
docker build -t mern-server ./Server
```

### Client (`Client/Dockerfile`)

```dockerfile
FROM node:lts-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
RUN npm install -g serve
EXPOSE 3000
CMD ["serve", "-s", "dist", "-l", "3000"]
```

Le client est construit avec `npm run build`, puis servi par le serveur HTTP léger `serve` sur le port 3000. En regardant le `package.json`, j'ai vu que le projet utilise **Vite**, donc le dossier généré s'appelle `dist` et non `build` comme avec Create React App. C'est pourquoi la commande finale pointe vers `dist`.

```bash
docker build -t mern-client ./Client
```

## 5. Réseau Docker et conteneurs manuels

J'ai créé un réseau dédié pour que les conteneurs puissent communiquer par leur nom :

```bash
docker network create mern-network
docker run -d --name mongodb --network mern-network mongo
docker run -d --name server --network mern-network -p 9000:9000 -e MONGO_URI=mongodb://mongodb:27017/mern mern-server
docker run -d --name client --network mern-network -p 3000:3000 mern-client
```

L'option `-p 9000:9000` relie le port du conteneur à celui de ma machine, ce qui me permet d'ouvrir l'application dans le navigateur. La variable `MONGO_URI` utilise le nom `mongodb` comme adresse, car Docker résout le nom des conteneurs d'un même réseau.

Test : http://localhost:3000 affiche « API is running », et http://localhost:9000 affiche « Cannot GET / ». Ce second message est normal : Express répond, mais aucune route `/` n'est définie. Cela prouve que le serveur fonctionne.

## 6. Docker Compose

Le fichier `docker-compose.yml` est à la racine du projet et remplace toutes les commandes précédentes. Il définit :

- **mongodb** : image `mongo`, conteneur nommé `mongodb`
- **server** : build depuis `./Server`, port 9000, variable `MONGO_URI`, dépend de `mongodb`
- **client** : build depuis `./Client`, port 3000, dépend de `server`
- un réseau **mern-network** avec le driver `bridge`, utilisé par les trois services

```bash
docker compose up --build
```

Avant de lancer Compose, j'ai supprimé les conteneurs et le réseau créés à la main pour éviter les conflits de noms et de ports.

## 7. Pipeline Jenkins

Le fichier `Jenkinsfile` à la racine décrit un pipeline en 4 étapes :

| Étape                        | Rôle                                                                                            |
| ----------------------------- | ------------------------------------------------------------------------------------------------ |
| **Checkout**            | Récupère le code depuis mon dépôt GitHub (branche `main`)                                  |
| **Build images**        | Construit les images `mern-server` et `mern-client`                                          |
| **Deploy with Compose** | Arrête l'ancienne version (`docker compose down`) puis lance `docker compose up --build -d` |
| **Test**                | Attend le démarrage, affiche `docker compose ps` et teste le client avec `curl`             |

Le bloc `post { failure { ... } }` affiche les logs Docker Compose si le pipeline échoue.

Le job s'appelle **TP4-MERN**, de type *Pipeline*, avec la définition *Pipeline script from SCM* (Git, branche `*/main`, Script Path `Jenkinsfile`).

**Résultat :** après plusieurs essais, le build **#8** s'est terminé avec `Finished: SUCCESS`. Les conteneurs `mongodb`, `server` et `client` ont été créés par Jenkins.

## 8. Difficultés rencontrées

- **`no configuration file provided: not found`** : j'étais dans le dossier parent. Docker Compose cherche le `docker-compose.yml` dans le dossier courant, il fallait faire `cd MyAppReact`.
- **Docker Desktop éteint** : l'erreur `failed to connect to the docker API` venait du fait que le moteur Docker n'était pas démarré.
- **`build` ou `dist`** : mon premier Dockerfile client visait `build`, alors que Vite génère `dist`.
- **Erreur dans le sujet** : la commande du client est écrite `docker build -t mern-server ./client`. J'ai utilisé `mern-client` pour ne pas écraser l'image du serveur.
- **Jenkins ne trouvait pas `docker`** (builds #1 à #5) : Jenkins tourne comme service Windows avec un autre utilisateur. J'ai ajouté le chemin de `docker.exe` dans la variable `PATH` du Jenkinsfile, après l'avoir trouvé avec `where.exe docker`.
- **`docker compose` : unknown command** (build #6) : Jenkins ne trouvait pas le plugin Compose de Docker. J'ai ajouté la variable `DOCKER_CONFIG` pointant vers le dossier `.docker` de mon utilisateur, ce qui a permis le build #8 réussi.
- **Conflit de noms de conteneurs** : les conteneurs créés par Jenkins portent les mêmes noms que ceux de mon Compose local. Il faut faire `docker rm -f client server mongodb` avant de relancer l'un ou l'autre.

## 9. Captures d'écran

**Conteneurs créés par Jenkins** 

**Client sur http://localhost:3000**

**Serveur sur http://localhost:9000**

**Pipeline Jenkins TP4-MERN**

**Console Output du build #8 : SUCCESS**

**Historique des builds**

## 10. Conclusion

Ce TP m'a permis de comprendre l'intérêt de Docker pour empaqueter une application avec ses dépendances, de Docker Compose pour lancer plusieurs services ensemble, et de Jenkins pour automatiser tout le processus. Les erreurs rencontrées, surtout avec Jenkins, m'ont appris à lire les logs et à diagnostiquer un problème étape par étape.
