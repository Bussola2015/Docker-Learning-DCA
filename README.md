# Docker Learning – Minha jornada rumo ao DCA

[![Estudando Docker](https://img.shields.io/badge/Estudando-Docker-2496ED?logo=docker&logoColor=white)](https://github.com/Bussola2015/Docker-Learning-DCA)
[![Última atualização](https://img.shields.io/github/last-commit/Bussola2015/Docker-Learning-DCA?color=green)](https://github.com/Bussola2015/Docker-Learning-DCA)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Olá, marujo!  
Esse é o meu **caderno vivo de estudos** para a certificação **Docker Certified Associate (DCA)**.  
Comecei em novembro de 2025 e vou atualizando conforme avanço nos tópicos oficiais, faço labs e quebro a cabeça (muito).

Aqui tem só o que realmente cai na prova + comandos que uso no dia a dia. Perfeito pra quem tá estudando pra DCA ou quer um cheat sheet Docker direto ao ponto.

---
## Índice rápido
- [Comandos essenciais](#comandos-essenciais)
- [Dockerfile na prática](#dockerfile-na-prática)
- [Docker Compose](#docker-compose)
- [Volumes & Networks](#volumes--networks)
- [Dicas que salvam na prova DCA](#dicas-que-salvam-na-prova-dca)
- [Como usar essas notas](#como-usar-essas-notas)

---
## Comandos essenciais

| Comando                           | O que faz                                      | Exemplo comum                              |
|-----------------------------------|------------------------------------------------|--------------------------------------------|
| `docker run`                      | Cria e inicia container                        | `docker run -d -p 8080:80 nginx`           |
| `docker ps -a`                    | Lista todos os containers                      | `docker ps -a`                             |
| `docker exec -it <id> /bin/bash`  | Entra no container rodando                     | `docker exec -it meu-mysql bash`           |
| `docker logs -f <id>`             | Segue os logs em tempo real                    | `docker logs -f app-web`                   |
| `docker build -t nome:tag .`      | Constrói imagem a partir do Dockerfile         | `docker build -t minha-app:v1 .`           |
| `docker images`                   | Lista imagens locais                           | `docker images`                            |
| `docker rmi <imagem>`             | Remove imagem                                  | `docker rmi nginx:latest`                  |
| `docker rm <container>`           | Remove container parado                        | `docker rm meu-container`                  |
| `docker system prune -a`          | Limpa tudo que não está sendo usado (cuidado!)| `docker system prune -a --volumes`         |

---
## Dockerfile na prática

```dockerfile
# Exemplo real que uso em produção
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
