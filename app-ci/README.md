# Dockerisation — satisfaction-evaluation

Documentation de la conteneurisation des deux modules du projet.

---

## Module 1 — Backend 
### Dockerfile

```dockerfile
FROM maven:3.9-eclipse-temurin-8 AS projet-compiler
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:8-jre
WORKDIR /app
COPY --from=projet-compiler /app/target/*.jar backend.jar
EXPOSE 8050
CMD ["java", "-jar", "backend.jar"]
```

**Intérêt du multi-stage build :** l'image finale ne conserve que le fichier `.jar`, exécuté sur un simple JRE, sans Maven ni code source ni compilateur — une image jusqu'à 5 fois plus légère et plus sécurisée.

### Build et exécution

```bash
docker build -t satisfaction-core evaluation-clinisys-core/

docker container run -d \
  --name=backend_satisfaction_core \
  -p 8085:8050 \
  --restart=always \
  --network=satisfaction-network \
  satisfaction-core
```
---

## Module 2 — Frontend 

### Dockerfile

```dockerfile
FROM node:20-alpine AS builder-pro
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
RUN npm run build-dist

FROM nginx:1.25.2-alpine
WORKDIR /usr/share/nginx/html
COPY --from=builder-pro /app/dist .
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Même principe : Node.js sert uniquement à générer les fichiers statiques (`dist/`). L'image finale ne contient que Nginx et ces fichiers, sans Node ni `node_modules`.

### Build et exécution

```bash
docker build -t satisfaction-web evaluation-clinisys-web/

docker container run -d \
  --name=web_satisfaction_web \
  -p 8084:80 \
  --restart=always \
  --network=satisfaction-network \
  satisfaction-web
```
## Équivalent avec Docker Compose

Plutôt que de lancer chaque `docker build` / `docker run` manuellement, le fichier `docker-compose.yml` réalise la même opération en une seule commande : build des deux images, création du réseau, démarrage des deux conteneurs liés.

```bash
docker compose up -d --build
```