
# TP4 : Docker et Docker Compose (application MERN)

## Objectif

Containeriser une application MERN avec Docker puis l'orchestrer avec Docker Compose.

## Structure du projet

- Client/ : application React (Vite) + Dockerfile
- Server/ : API Express + Dockerfile
- docker-compose.yml : orchestration des 3 services

## Étapes réalisées

### 1. Clonage du projet

git clone https://github.com/wiem-kb/MyAppReact.git

### 2. Dockerfile du serveur (Server/Dockerfile)

Image de base node:lts-alpine, répertoire de travail /app, copie de package*.json,
npm install, copie du code, port 9000, démarrage avec npm start.
Commande : docker build -t mern-server ./Server

### 3. Dockerfile du client (Client/Dockerfile)

Image de base node:lts-alpine, npm install, npm run build (Vite génère le dossier dist),
installation de "serve", port 3000, service du dossier dist.
Commande : docker build -t mern-client ./Client

### 4. Réseau Docker

docker network create mern-network

### 5. Lancement manuel des conteneurs

docker run -d --name mongodb --network mern-network mongo
docker run -d --name server --network mern-network -p 9000:9000 -e MONGO_URI=mongodb://mongodb:27017/mern mern-server
docker run -d --name client --network mern-network -p 3000:3000 mern-client

### 6. Docker Compose

Fichier docker-compose.yml avec les services mongodb, server (build ./Server, port 9000,
variable MONGO_URI, depends_on mongodb) et client (build ./Client, port 3000, depends_on server),
tous connectés au réseau mern-network (driver bridge).
Commande : docker compose up --build

## Variable d'environnement

MONGO_URI=mongodb://mongodb:27017/mern
Dans Docker Compose, le serveur joint MongoDB en utilisant le nom du service "mongodb" comme hostname.

## Tests effectués

- docker ps / docker compose ps : 3 conteneurs en cours d'exécution
- http://localhost:3000 : le client répond ("API is running")
- http://localhost:9000 : le serveur Express répond ("Cannot GET /" car aucune route / n'est définie, ce qui prouve que le serveur fonctionne)
- docker exec server ping -c 2 mongodb : communication entre les services validée

## Remarque

La commande de build du client dans l'énoncé (mern-server ./client) contient une erreur de nom ;
la commande correcte est : docker build -t mern-client ./Client

## Captures d'écran

Voir le dossier captures/ pipeline jenkins file avec builds; capture de creation de contenaire
