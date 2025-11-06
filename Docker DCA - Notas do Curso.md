# Docker DCA - Notas do Curso (Caio Delgado)

**Autor das Notas:** Valdir A Bussola Jr 

**Curso:** [Docker DCA no YouTube](https://www.youtube.com/playlist?list=PL4ESbIHXST_TJ4TvoXezA0UssP1hYbP9_)  
**Status:** Completo até a aula 15 (faltando Kubernetes).  
**Objetivo:** Guia **passo a passo** para revisar comandos, exemplos, conceitos e observações.  
**Dica Geral:** Sempre teste em ambiente lab (ex: VMs Vagrant). Use `docker system prune -a --volumes -f` para limpar tudo entre testes.

## 📋 Sumário por Aulas (Baseado na Playlist)
| Aula | Título (Aproximado) | Seção Principal |
|------|---------------------|-----------------|
| 1 | Arquitetura Docker| [Arquitetura](#Arquitetura)
| 2 | Introdução e Login | [Globais](#globais) |
| 3 | Containers Básicos | [Containers](#containers) |
| 4 | Imagens e Build | [Imagens](#imagens) |
| 5 | Volumes | [Volumes](#volumes) |
| 6 | Backup/Restore e Plugins | [Backup/Restore](#backup-e-restore) + [Plugins](#plugins-de-volume) |
| 7 | Networking | [Networking](#networking-docker) |
| 8 | Docker Compose | [Compose](#Compose)
| 9-10-11-12 | Swarm Mode (Init, Services, Scale) | [Swarm](#swarm) |
| 13 | Monitoring (Prometheus/Grafana) | [Monitoramento](#monitoramento) |
| 14 | Tools (Swarmpit, Portainer, Harbor e Docker Machine) | [Tools](#tools-docker) |
| 15 | Labels e Aplicações| [Labels](#Labels) |
| 16 | Kubernetes | [Kubernetes](#Kubernetes) |
| 17 | Docker MKE ou Docker EE | [MKE](#MKE) |

**Linha de Raciocínio:**  
1. **Fundamentos** (Login → Run → Inspect).  
2. **Gerenciamento** (Containers/Images/Volumes).  
3. **Persistência** (Volumes/Backup/Plugins).  
4. **Rede** (Drivers → Swarm).  
5. **Orquestração** (Swarm/Stacks/Secrets).  
6. **Produção** (Monitoring/Tools).  
**Regra de Ouro:** `docker run` 99% do tempo. Use `exec` > `attach`. Prefira volumes nomeados > bind mounts.

---
## 🗺️ Arquitetura 

![Arquitetura Docker - Visão Geral](https://docs.docker.com/get-started/images/docker-architecture.webp)

**Fonte:** Retirado de [https://docs.docker.com/engine/docker-overview/](https://docs.docker.com/engine/docker-overview/)

---
## 🔑 Globais

### Login/Logout
**Passo a Passo:**
1. **Login (várias formas):**
   ```bash
   # Com usuário e senha/PAT
   docker login -u 545648856789412  # Digite senha/PAT
   
   #genérica
   docker login -u user_ID --password digite-senha-aqui

   # Via stdin (recomendado para scripts)
   echo "dckr_pat_abc123def4567890xyz" | docker login -u meuusuario --password-stdin

   # Interativo (gera código)
   docker login  # Gera XKFC-GVHW → Cole no browser (página web docker hub)
   ```
2. **Logout:**
   ```bash
   docker logout
   ```

### Comandos Globais (Troubleshooting/Limpeza)
| Comando | Descrição | Exemplo |
|---------|-----------|---------|
| `docker system info` | Overview completo (configurações) | `docker info` (legado) |
| `docker system prune` | Limpa não-usados (containers, images, etc.) | `docker system prune -f` (força) |
| `docker system prune -a` | Inclui todas images | `docker system prune -a --volumes -f` (completo) |
| `docker system prune --filter "until=168h"` | Até X horas | - |
| `docker plugin install <plugin>` | Instala plugin | `docker plugin ls` (lista) |
| `docker search <imagem>` | Busca no Hub | `curl -s "https://registry.hub.docker.com/v2/repositories/library/rockylinux/tags/?page_size=100" \| jq -r '.results[].name'` (tags) |

**Observação:** Prune = **limpeza essencial** entre labs.

### Exemplos Gerais (Primeiros Testes)
```bash
# Hello World
docker container run --rm -it hello-world

# Nginx simples
docker run -d -p 8080:80 --name my_nginx nginx

# Com volume
docker run -d -p 8080:80 -v /path/on/host:/usr/share/nginx/html --name my_nginx nginx

# Shell interativo
docker run -it rockylinux:9 /bin/bash

# Container "imortal" (tail -f)
docker container run -dit --name rocky9 --hostname c1 rockylinux:9 tail -f /dev/null
```

---

## 🐳 Containers

### Listar/Inspecionar
| Comando | Descrição | Filtros |
|---------|-----------|---------|
| `docker container ls` | Running | `docker ps` (legado) |
| `docker container ls -a` | Todos | `-f "status=exited"` (parados) |
| `docker container ls -aq` | Só IDs | `-f "status=running\|paused"` |

### Iniciar/Parar/Remover
```bash
# Stop/Start
docker container stop <nome\|ID>  # Múltiplos OK
docker container start <nome\|ID>

# Remover
docker container rm <nome\|ID>  # Múltiplos
docker container rm -f $(docker container ls -aq)  # Todos
docker container prune -f  # Parados só
docker container ls -aq | xargs docker container rm #outra maneira de excluir
```

### Conectar/Sair (🔥 **Sempre use `exec`**)
| Método | Comando | Sair sem parar |
|--------|---------|----------------|
| **Recomendado** | `docker container exec -it <nome> bash` | `CTRL+D` (fica RUNNING) |
| Attach | `docker container attach <nome>` | `CTRL+P+Q` (Read Escape) |

**OBS Crucial:** `attach` + `exit` = STOP. `exec` + `exit` = RUNNING.

### Copiar/Executar/Logs/Inspect
```bash
# Copiar
docker container cp teste rocky8:/tmp

# Exec
docker container exec rocky8 ls -l /tmp  #executa comando sem attach 

# Logs
docker container logs <nome ou ID>

# Inspect (JSON)
docker container inspect almalinux
docker inspect almalinux | jq '.[] | .Config | {Cmd, Entrypoint}'
```

### `docker container create` vs `run`
| Comando | O que faz | Quando usar |
|---------|-----------|-------------|
| `docker container create <nome>` | **Só cria** (não start) | Config avançada |
| `docker run` | Create + Start | **99% dos casos** |

**Exemplo:**
```bash
docker container create --name meu-container nginx
docker container start meu-container
```

### Gambiarra: Commit → Imagem Custom (🚫 **Evite em Prod, use Dockerfile**)
**Passo a Passo:**
1. Rode container → Modifique:
   ```bash
   docker container run --name rocky -it rockylinux:9 /bin/bash
   dnf install -y nginx  # Dentro
   ```
2. Commit:
   ```bash
   docker container commit rocky rocky-nginx  #gera uma nova imagem
   ```
3. Salvar/Carregar:
   ```bash
   docker image save rocky-nginx:latest -o rocky-linux.tar  #gera um tar da nova imagem
   docker image rm rocky-nginx:latest; docker container rm -f rocky  #remove a imagem e o container
   docker image load -i rocky-linux.tar  #retorna a imagem a partir do .tar
   ```
**Diferença:** `<container-origem>` → `<nova-imagem>`.

---

## 🖼️ Imagens



**Objetivo:** Dominar o ciclo completo de imagens — **listar, remover, investigar, construir, otimizar e distribuir** — com foco em **performance, segurança e boas práticas**.

---

### 🔍 Listar e Remover Imagens

| Comando | Descrição | Exemplo |
|--------|-----------|--------|
| `docker image ls` | Lista todas | `docker images` (legado) |
| `docker image ls -a` | Inclui intermediárias | - |
| `docker image ls -f "dangling=true"` | **Imagens órfãs** (sem tag, sem uso) | - |
| `docker image ls -q` | Apenas IDs | Útil em scripts |
| `docker image rm <ID\|nome>` | Remove uma ou mais | `docker image rm nginx` |
| `docker image rm -f $(docker image ls -aq)` | **Remove TODAS** | Cuidado! |
| `docker image prune` | Remove **imagens não usadas** | - |
| `docker image prune -a` | Remove **todas não referenciadas** | - |
| `docker image prune -f` | Sem confirmação | - |
| `docker image prune -a -f` | **Limpeza total** | - |

**Dica:** Use `prune -a -f` entre labs para evitar bloat.

---

### ℹ️ Ajuda e Investigação

```bash
docker image --help
```

### **Inspeção Profunda**
```bash
# Histórico de camadas (deduzir Dockerfile)
docker image history prom/prometheus:main

# Metadados completos (JSON)
docker image inspect prom/prometheus:main

# Filtrar campos específicos
docker image inspect nginx | jq '.[0].Config.ExposedPorts'
```

---

### 🔎 Buscar e Baixar

```bash
# Busca no Docker Hub
docker search nginx
docker search nginx --filter stars=100 --limit 5
docker search --filter is-official=true nginx

# Pull (baixa)
docker image pull debian
docker image pull nginx:alpine
```

---

### 🏗️ Build (Construção)

### Sintaxe Básica
```bash
# Build no diretório atual
docker image build -t echo-container .

# Especificar Dockerfile e contexto
docker image build -t echo-test:latest -f echo-container/Dockerfile .

# Sem cache + tempo
time docker image build --no-cache -t exemplo:v2 -f path/Dockerfile context/
```

### **Contexto de Build**
> Tudo na pasta do `Dockerfile` é enviado ao daemon → **use `.dockerignore`**.

```bash
# .dockerignore (exemplo)
__pycache__
*.pyc
.git
node_modules
.env
```

---

### 🏷️ Tag e Push

```bash
# Tag (local)
docker image tag echo-container 545648841321245/echo-container:v1

# Push para registry
docker image push 545648841321245/echo-container:v1
```

---

### 🔥 Dockerfile: Exemplos do Curso (Otimizados)

### 1. **Echo Simples**
```dockerfile
FROM alpine
ENTRYPOINT ["echo"]
CMD ["--help"]
```
```bash
docker build -t meu-echo .
docker run meu-echo -n "Olá DCA!"
```

---

### 2. **Busybox + COPY**
```dockerfile
FROM busybox
COPY conteudo.txt /
RUN cat /conteudo.txt
```

---

### 3. **Apache (Debian)**
```dockerfile
FROM debian
RUN apt-get update && \
    apt-get install -y wget git apache2 && \
    rm -rf /var/lib/apt/lists/* /var/cache/apt/archives/*
EXPOSE 80
CMD ["apachectl", "-D", "FOREGROUND"]
```

---

### 4. **Java App (OpenJDK + Limpeza)**
```dockerfile
FROM openjdk:8-jre-alpine
COPY java-wc-app/target/app.jar /app/app.jar
COPY java-wc-app/samples /samples 
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

---

### 5. **Multi-stage Simples**
```dockerfile
FROM alpine:latest AS dicas
WORKDIR /samples
COPY 1.txt .

FROM alpine:latest
WORKDIR /root/
COPY --from=dicas /samples/1.txt .
CMD ["cat", "1.txt"]
```

---

### 6. **Go App (Sem Multi-stage)**
```dockerfile
FROM golang:1.7.3
WORKDIR /go/src/github.com/alexellis/href-counter/
RUN go get -d -v golang.org/x/net/html
COPY app.go .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .
CMD ["./app"]
```

---

### 7. **Go App (Multi-stage Otimizado)**
```dockerfile
FROM golang:1.19 AS builder
WORKDIR /go/src/github.com/alexellis/href-counter/
COPY go.mod go.sum ./
RUN go mod download
COPY app.go .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags "-s -w" -o app

FROM alpine:3.17.1
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /go/src/github.com/alexellis/href-counter/app .
CMD ["./app"]
```

> **Melhorias aplicadas:** `go mod`, `-ldflags`, `alpine`, `ca-certificates`.

---

### 8. **Go App (Versão Final - Curso)**
```dockerfile
FROM golang:1.19 AS builder
WORKDIR /go/src/github.com/alexellis/href-counter/
COPY go.mod go.sum app.go ./
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags "-s -w" -o app

FROM alpine
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /go/src/github.com/alexellis/href-counter/app .
CMD ["./app"]
```

---

## 🚀 Dockerfile Best Practices (Memorize!)

| # | Regra | Por quê? |
|---|-------|---------|
| 1 | **Ordem importa (cache)** | Mude o que varia por último |
| 2 | **`COPY` > `ADD`** | `ADD` aceita URL (risco) |
| 3 | **`.dockerignore`** | Evita bloat no contexto |
| 4 | **Agrupar `RUN`** | Menos camadas |
| 5 | **Limpar cache** | `rm -rf /var/lib/apt/lists/*` |
| 6 | **Imagens mínimas** | `alpine`, `slim`, `distroless` |
| 7 | **Tags específicas** | `openjdk:8-jre-alpine` |
| 8 | **Multi-stage** | Reduz tamanho final |
| 9 | **`USER` não-root** | Segurança |
| 10 | **`HEALTHCHECK`** | Monitoramento |

-----

### 🛠️ Troubleshooting e Logs Detalhados do Build - Debug em Dockerfile

O comando `docker build` (e o `docker compose build`) utiliza o BuildKit por padrão, que oferece uma saída limpa e otimizada (Modo TTY). No entanto, em caso de falhas, essa saída pode omitir logs cruciais.

Para depuração detalhada (Troubleshooting), utilize a flag **`--progress=plain`**.

### Uso: Logs Sequenciais e Completos

A flag `--progress=plain` força o BuildKit a reverter para um estilo de saída **linear e não interativo**, garantindo que você visualize:

1.  **Output Completo de `RUN`:** O log detalhado de cada comando `RUN` do seu `Dockerfile` é exibido por completo, o que é essencial para identificar pacotes que falharam na instalação ou erros de scripts.
2.  **Saída Consistente:** Garante um log fácil de analisar em ambientes de CI/CD (Integração Contínua) que não suportam o modo TTY.

### Exemplos Práticos:

```bash
# Para debug do build em um Dockerfile:
docker build --progress=plain .

# Para debug de um serviço específico no Docker Compose:
docker compose build --progress=plain <nome-do-serviço>
```

### 🎯 Por Que Isso é Importante em Troubleshooting

O principal desafio ao construir imagens Docker é depurar falhas que ocorrem durante o comando `RUN`. O modo padrão de progresso (`auto`/TTY) do BuildKit (o motor de build moderno do Docker) muitas vezes esconde o output detalhado de comandos que falharam ou foram cancelados, dificultando a identificação da linha exata que deu erro.


---

## 📌 `ENTRYPOINT` vs `CMD` — Guia Completo

> Explicação detalhada com casos reais e boas práticas.

---

### 1. **Definições Básicas**

| Comando | Descrição |
|--------|-----------|
| `CMD` | **Comando padrão** (sobrescrito facilmente) |
| `ENTRYPOINT` | **Comando principal** (sempre executa) |

---

### 2. **Diferenças Principais**

| Característica | `CMD` | `ENTRYPOINT` |
|----------------|-------|--------------|
| Sobrescrita | `docker run imagem novo-cmd` | `--entrypoint` |
| Argumentos | Substituem | São **anexados** |
| Uso comum | Ferramentas CLI | Aplicações |

---

### 3. **Formatos**

#### **Exec (Recomendado)**
```dockerfile
CMD ["nginx", "-g", "daemon off;"]
ENTRYPOINT ["python", "app.py"]
```

#### **Shell (Evite)**
```dockerfile
CMD nginx -g "daemon off;"
```
→ Shell é PID 1 → não recebe `SIGTERM`.

---

### 4. **Casos de Uso**

#### **Caso 1: `CMD` Flexível**
```dockerfile
FROM nginx:alpine
CMD ["nginx", "-g", "daemon off;"]
```
```bash
docker run img echo "teste"  # → echo "teste"
```

#### **Caso 2: `ENTRYPOINT` Fixo**
```dockerfile
FROM python:3.9
ENTRYPOINT ["python", "script.py"]
```
```bash
docker run img --help  # → python script.py --help
```

#### **Caso 3: `ENTRYPOINT` + `CMD` (Ideal)**
```dockerfile
ENTRYPOINT ["node", "app.js"]
CMD ["--port", "3000"]
```
```bash
docker run app --port 8080  # → node app.js --port 8080
```

#### **Caso 4: Script de Entrada**
```dockerfile
COPY entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["start"]
```
```bash
#!/bin/bash
echo "Iniciando..."
exec "$@"
```

---

### 5. **Boas Práticas**

| Regra | Exemplo |
|------|--------|
| Use **exec** | `["cmd", "arg"]` |
| Combine `ENTRYPOINT` + `CMD` | Controle + flexibilidade |
| Scripts? Use `exec "$@"` | Passa args corretamente |
| Evite shell em processos longos | Use exec |

---

### 6. **Resumo Rápido**

| Quer... | Use |
|--------|-----|
| Comando fixo | `ENTRYPOINT` |
| Args padrão | `CMD` |
| Ambos | `ENTRYPOINT` + `CMD` |
| Flexibilidade | Só `CMD` |

---

**Dica Final:**  
> `ENTRYPOINT` = **verbo principal**  
> `CMD` = **argumentos padrão**

---

## 🧹 Limpeza Final

```bash
# Remove tudo não usado
docker system prune -a --volumes -f
```

---

**Próximos Passos:**
- [ ] Testar todos os Dockerfiles
- [ ] Adicionar `HEALTHCHECK`
- [ ] Usar `distroless` em produção
- [ ] Automatizar com CI/CD

**Referências:**
- [Docker Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/)


---

## 💾 Volumes

**Conceitos:**
- **Storage:** OverlayFS (COW/Layers). Config: `/etc/docker/daemon.json`.
- **Tipos:** Host (bind), Anônimo/Nomeado (Docker), tmpfs (RAM).
- **Prefira:** Volumes nomeados (backup fácil, share entre containers).
- **Flags:** `-v` ou `--volume` (simples) vs `--mount` (poderoso, cluster-only).

**OBS:** O arquivo de configuração não existe por default na instalação do docker, precisa ser criado manualmente `/etc/docker/daemon.json`

### Ajuda/Help:
```bash
docker volume --help
```

### 1. **Bind Mount (Host)**
```bash
docker run -dit --name servidor -v /srv:/srv debian  # Cria /srv se não existe
docker container exec servidor ls -l /srv   #confirma o volume no container
docker container inspect servidor | grep vol
```

### 2. **Volume Anônimo**
```bash
docker run -dit --name servidor -v /vol debian  # Sem source → /var/lib/docker/volumes/<hash>/_data
docker volume ls  # Hash 64 chars
docker volume inspect <hash>
```

### 3. **Volume Nomeado** (✅ **Melhor**)
```bash
docker volume create volume1
docker run -dit --name servidor -v volume1:/vol debian
docker volume ls  # Nome visível
```

### 4. **tmpfs (RAM/Volátil)**
```bash
# Moderna
docker container run -dit --name tmpfs --mount type=tmpfs,destination=/app,tmpfs-size=100M debian

# Legada
docker container run -dit --name tmpfs --tmpfs /app,size=100M debian
docker container exec tmpfs df -hT /app    #confirmar a pasta /app
```

### **--mount Sintaxe Completa**
```bash
# Anônimo
--mount type=volume,target=/vol

# Nomeado
--mount type=volume,source=meu-vol,target=/vol

# Bind (readonly)
--mount type=bind,source=/host/path,target=/container,readonly

# Inspect
docker container inspect <nome> --format '{{json .Mounts}}' | jq
```

### Limpeza
```bash
docker volume prune -f -a
docker volume rm $(docker volume ls -q)
```

---

## 💼 Backup e Restore (🔥 **Sem Downtime**)

**Passo a Passo (Anônimo → Tar):**
1. **Criar/Preencher:**
   ```bash
   docker run -dit -v /webdata --name webserver debian
   docker cp dockerfiles/ webserver:/webdata
   docker container exec webserver ls -l /webdata   #opcional, confirma se de criou /webdata 
   ```

2. **Backup (Container Auxiliar Alpine):**
   ```bash
   docker run --rm --volumes-from webserver -v $(pwd):/backup alpine tar cvf /backup/backup.tar /webdata
   ```

3. **Restore (Novo Container):**
   ```bash
   docker run -dit -v /webdata --name webserver2 debian
   docker run --rm --volumes-from webserver2 -v $(pwd):/backup alpine ash -c "cd /webdata && tar xf /backup/backup.tar --strip 1"
   docker exec webserver2 ls -l /webdata  # ✅
   ```

**OBS:** `--volumes-from` **legado**. Use nome do volume diretamente.

**Share Volume (2 Containers):**
```bash
docker run -dit -v <hash>:/webdata --name server2 debian
# Ou --volumes-from (legado)
```

---

## 🔌 Plugins de Volume

### 1. **SSHFS (Vieux)**
```bash
docker plugin install vieux/sshfs DEBUG=1
docker plugin ls

# Criar (VM remota com SSH)
docker volume create -d vieux/sshfs -o sshcmd=vagrant@10.2.0.15:/vagrant -o password=vagrant sshvolume

# Test
docker run --rm -v sshvolume:/data alpine ls /data
```

### 2. **NFS (Trajano) - **Global/Swarm**"
```bash
docker plugin install --alias nfsvolplug trajano/nfs-volume-plugin --grant-all-permissions

# Criar (Client aponta Server)
docker volume create -d trajano/nfs-volume-plugin -o device=192.168.15.4:/home/vagrant/storage volume_nfs

# Test Nginx
docker run -d -v volume_nfs:/usr/share/nginx/html -p 80:80 nginx
```

**OBS:** NFS Server em VM separada.

---

## 🌐 Networking Docker

**Conceitos:**
- **veth:** Placa virtual.
- **Bridge:** Default (docker0 + proxy).
- **Drivers:**
| Driver | Escopo | Uso |
|--------|--------|-----|
| **bridge** | Local | Default |
| **host** | Local | Mesmo IP host (sem -p) |
| **overlay** | **Global/Swarm** | Cluster |
| **none** | Isolado | Builds |
| macvlan | VLANs | Avançado |
| **bridge** | Local | user defined (--create) |

## Bridge Default

![Bridge Default](https://rgw.cloudpoint.tcpro.cz/swift/v1/KEY_0efe203c42c0402f9402a570302dc066/blog_new/Introduction-to-Container-Networking-in-Docker/Introduction-to-Container-Networking-in-Docker-01.webp)

**Fonte:** Retirado de [https://taikun.cloud/introduction-to-container-networking-in-docker/](https://taikun.cloud/introduction-to-container-networking-in-docker/)

## Bridge User-Defined

![Bridge Default](https://www.docker.com/app/uploads/2022/12/networking-drivers-use-cases-3.png)

**Fonte:** Retirado de [https://www.docker.com/blog/understanding-docker-networking-drivers-use-cases/](https://www.docker.com/app/uploads/2022/12/networking-drivers-use-cases-3.png)

## Host

![Host](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcQ5p23HsrfHe007XvZ3CVJds1CujZ1tobfA9g&s)

**Fonte:** Retirado de [https://www.oneclickitsolution.com/centerofexcellence/devops/docker-networking-explained](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcQ5p23HsrfHe007XvZ3CVJds1CujZ1tobfA9g&s)

## Overlay

![Overlay](https://www.docker.com/app/uploads/2022/12/networking-drivers-use-cases-4.png)

**Fonte:** Retirado de [https://www.docker.com/blog/understanding-docker-networking-drivers-use-cases/](https://www.docker.com/app/uploads/2022/12/networking-drivers-use-cases-4.png)

## Macvlan

![Macvlan](https://www.docker.com/app/uploads/2022/12/networking-drivers-use-cases-2.png)

**Fonte:** Retirado de [https://www.docker.com/blog/understanding-docker-networking-drivers-use-cases/](https://www.docker.com/app/uploads/2022/12/networking-drivers-use-cases-2.png)

## None

![None](https://miro.medium.com/v2/resize:fit:720/format:webp/1*CeXXSLkZ0GfML3iMdhmpaw.png)

**Fonte:** Retirado de [https://meghasharmaa704.medium.com/none-network-driver-6811873e83af](https://miro.medium.com/v2/resize:fit:720/format:webp/1*CeXXSLkZ0GfML3iMdhmpaw.png)


**Referência:**  
- [Docker Blog - Understanding Networking Drivers](https://www.docker.com/blog/understanding-docker-networking-drivers-use-cases/)  
- [Docker Docs - Networking](https://docs.docker.com/engine/network/)
- [Docker Docs - Networking Containers](https://docker-docs.uclv.cu/engine/tutorials/networkingcontainers/) 


### Importante sobre drivers de rede bridge

| Recurso                | Rede bridge Padrão (bridge)             | Rede bridge Definida pelo Usuário       |
|-----------------------|-----------------------------------------|-----------------------------------------|
| **Criação**            | Automática (existe por padrão)          | Manual (`docker network create ...`)    |
| **Descoberta de Serviços (DNS)** | Não suportada. Contêineres só se comunicam por endereço IP (ou usando a flag `--link`). | Suportada. Contêineres podem se comunicar usando o nome do outro contêiner (ex.: `ping meu-banco`). |
| **Isolamento**         | Ruin. Todos os contêineres sem rede especificada se conectam a ela, permitindo comunicação indesejada. | Excelente. Apenas os contêineres que você anexar podem se comunicar entre si, garantindo melhor isolamento. |
| **Anexar/Desanexar**   | Impossível. Para sair, você precisa parar e recriar o contêiner. | Flexível. Você pode anexar ou desanexar contêineres em tempo de execução (`docker network connect/disconnect`). |
| **Recomendação**       | Não recomendada para ambientes de produção ou multi-contêiner. | Altamente recomendada para qualquer aplicação com dois ou mais contêineres. |


### Relação de Rede `host` com Portas no Docker

Esta seção explora como o driver de rede `host` remove o isolamento de rede entre o contêiner e o host (máquina hospedeira), com foco no fim do uso de NAT. A tabela abaixo compara os mecanismos e cenários de uso.

| **Driver de Rede**                | **Por que é usado `--publish`?**         | **Por que `--publish` NÃO é usado?**    | **Exemplo Prático**                     |
|----------------------------|------------------------------------------|------------------------------------------|-----------------------------------------|
| **Bridge (Padrão ou User-Defined)**         | Para redirecionar uma porta do host para a porta do contêiner (ex.: `-p 8080:80`). | O contêiner tem um IP interno isolado.   | `docker run -d -p 8080:80 --name web nginx` (Acesso via Host:8080) |
| **host** | Não aplicável.                           | O contêiner compartilha o stack de rede do host. Se um serviço estiver rodando na porta 80 do contêiner, ele será redirecionado como porta 80 do host. | `docker run -d --network host --name web nginx` (Acesso via Host:80) |

### Observações
- O driver `host` elimina o isolamento de rede, fazendo o contêiner usar a rede do host diretamente.
- Ao usar `--network host`, não é necessário mapear portas (ex.: `-p`), pois o contêiner herda as portas do host.


### Exemplos
```bash
# Bridge (default)
docker run -d --name web -h server -p 80:80 --network bridge nginx # -h = --hostname
docker container run -dit --name webserver -p 80:80 nginx #implícito o driver - idem comando anterior
docker run -dit --name webserver2 -p 8080:8080 ubuntu

docker container exec webserver2 ping -c4 webserver  # não resolve dns
docker container exec webserver2 ping -c4 172.17.0.3  # funciona

docker network ls/prune/inspect bridge
docker network prune -f 
docker network prune --filter until=24h
docker network inspect bridge | jq
docker network inspect <id-de-rede>

# Host
docker run -d --name webhost --network host nginx  # Sem -p!

# Bridge user-defined
docker network create minha-rede-app
docker run -d \
  --network minha-rede-app \
  --name banco-de-dados \
  -e POSTGRES_PASSWORD=minhasenha \
  postgres:latest
docker run -d \
  --network minha-rede-app \
  --name servidor-web \
  -p 8080:80 \
  nginx

docker container exec servidor-web ping -c4 banco-de-dados  # dns funciona 
docker network disconnect minha-rede-app servidor-web  # on the fly - desconectei da rede
docker network connect --ip 172.17.0.200 minha-rede-app servidor-web  #reconectei com IP novo

#Macvlan

docker network create -d macvlan \
  --subnet=192.168.10.0/24 \
  --gateway=192.168.10.1 \
  -o parent=eth0 \
  --aux-address="host=192.168.10.254" \
  vlan10_network

docker network create -d macvlan \
  --subnet=192.168.20.0/24 \
  --gateway=192.168.20.1 \
  -o parent=eth0 \
  --aux-address="host=192.168.20.254" \
  vlan20_network

docker container run -dit --name webserver_vlan10 \
  --network vlan10_network \
  nginx
docker container run -dit --name webserver_vlan20 \
  --network vlan20_network \
  nginx

# None
docker run -d --network none nginx
```

**Fluxo Bridge:** Host (enp0s3) → docker0 (interface do docker-proxy) → veth (eth0 container).

**Atenção:** Plugins de rede e rede Overlay tem escopo global no Docker.

---

## Importante diferenciar o uso do termo Overlay no ecossistema Docker

💡 **Overlay: Rede (Network Driver) vs. Armazenamento (Storage Driver)**

| Termo            | Categoria             | Função                                                                 | Uso Principal              |
|-------------------|------------------------|-----------------------------------------------------------------------|----------------------------|
| overlay           | Driver de Rede (Network Driver) | Permite que contêineres em diferentes Hosts Docker (em um cluster, como Docker Swarm) se comuniquem como se estivessem na mesma rede local. | Comunicação Multi-Host (Clusters) |
| overlay2 (ou overlay) | Driver de Armazenamento (Storage Driver) | Gerencia como as camadas de imagens (Image Layers) são empilhadas e como os contêineres adicionam uma camada gravável (Copy-on-Write). | Eficiência de Disco (Build e Runtime) |

### 1. Overlay (Rede)
- **O que faz**: Cria uma rede virtual distribuída que se estende por vários hosts.
- **Comando**: Usado com `docker network create -d overlay ...`
- **Foco**: Conectividade entre máquinas.

### 2. Overlay2 (Armazenamento)
- **O que faz**: É o driver de armazenamento mais comum e recomendado em sistemas Linux modernos. Ele usa o sistema de arquivos subjacente para implementar a funcionalidade Copy-on-Write (COW), que é o que permite que uma imagem seja composta por múltiplas camadas somente leitura e uma única camada gravável (o contêiner).
- **Comando**: Definido na configuração do Docker Daemon (via `/etc/docker/daemon.json`).
- **Foco**: Estrutura de Imagens e Contêineres no disco.

### Conclusão
- Apesar de compartilharem a raiz "Overlay", eles operam em domínios completamente diferentes do Docker: um é sobre como os contêineres falam entre si em um cluster, e o outro é sobre como os arquivos de imagens e contêineres são armazenados no disco.

---
Com certeza\! Aqui está o resumo da nossa conversa sobre a relação de Boas Práticas, Imutabilidade e configuração de contêineres Docker, formatado em Markdown do início ao fim.

-----

## 🛡️ Boas Práticas no Docker: Imutabilidade e Configuração Declarativa

Sua observação sobre evitar a edição manual de arquivos internos em contêineres está **totalmente correta** e alinhada com as melhores práticas e a filosofia de **Imutabilidade de Contêineres** (Immutable Containers).

A regra de ouro é: **Trate o contêiner como Imutável.** Qualquer alteração feita manualmente dentro de um contêiner será perdida se ele for recriado, quebrando a reprodutibilidade.

-----

## 1\. O Princípio da Imutabilidade

O Docker oferece ferramentas apropriadas (flags, `Dockerfile`, `daemon.json`) para modificar o comportamento do contêiner de forma declarativa e persistente.

O exemplo de DNS demonstra a diferença entre as práticas:

| Prática | Comando | Status | Porquê? |
| :--- | :--- | :--- | :--- |
| **Má Prática (Evitar)** | `docker exec container1 nano /etc/resolv.conf` | **Perigoso** | A alteração é efêmera, não faz parte da definição do contêiner e é perdida ao recriá-lo. |
| **Boa Prática (Correto)** | `docker run --dns 8.8.8.8 ...` | **Recomendado** | A configuração é declarativa, faz parte do comando de execução, e é facilmente replicável. |

-----

## 2\. O Caso Específico do DNS

A configuração DNS no Docker pode ser controlada em três níveis:

### A. Padrão: DNS do Host

Por padrão, o Docker tenta **herdar a configuração DNS do sistema operacional Host** ou utiliza um servidor DNS interno que encaminha as requisições para o DNS do Host.

### B. Configuração Global (Persistente) via Daemon

Para configurar o DNS de forma persistente para **todos os contêineres** no sistema (a menos que seja sobrescrito):

```json
# Arquivo /etc/docker/daemon.json
{
  "dns": ["8.8.8.8", "8.8.4.4"]
}
```

  * **Status:** **Boa Prática**. Define o padrão do sistema.
  * **Ação:** Requer reiniciar o Docker Daemon (`sudo systemctl restart docker`).

### C. Configuração por Contêiner (Sobrescrita)

Para aplicar uma configuração DNS apenas a um contêiner específico:

```bash
docker run -dit --name container2 --dns 8.8.8.8 debian
```

  * **Status:** **Boa Prática**. Configuração declarativa e específica para a execução.

-----

## 3\. Generalizando: Usando a Ferramenta Certa

O princípio de evitar a edição manual aplica-se a quase todos os aspectos do contêiner, forçando o uso de ferramentas declarativas do Docker:

| O que você quer fazer? | Má Prática (Evitar Edição Manual) | Boa Prática (Ferramenta Docker) |
| :--- | :--- | :--- |
| **Instalar Pacotes** | `docker exec ... apt install vim` | **`Dockerfile`**: Use a instrução `RUN` (pacote faz parte da imagem). |
| **Alterar Configurações** | `docker exec ... nano /etc/app/config.yml` | **Volumes de Configuração**: Use a *flag* `-v` (montar arquivo do Host para o Contêiner). |
| **Mudar Usuário** | `docker exec ... chown user:group ...` | **`Dockerfile`**: Use a instrução `USER`. Ou `docker run --user ...`. |
| **Ambiente** | `docker exec ... export VAR=value` | **`docker run -e VAR=value`** ou use arquivos `.env`. |

-----

## ✅ Conclusão

A chave para um ambiente Docker saudável e robusto é a **Reprodutibilidade**. Priorize o uso das *flags*, *drivers* e arquivos de configuração do Docker (`Dockerfile`, `docker run`, `daemon.json`) em vez de intervenções manuais via `docker exec`.

---
## ⚓ Docker Compose: Orquestração Local

**Objetivo:** Orquestrar múltiplos containers localmente com `docker compose`, definindo serviços, redes e volumes em um arquivo YAML. Esta seção organiza os comandos, conceitos e exemplos do curso Docker DCA de Caio Delgado, complementando com boas práticas, sintaxe técnica aprimorada e exemplos claros para facilitar a revisão e aplicação.

**Conceito Principal:** Docker Compose é uma ferramenta para definir e gerenciar aplicações multi-container localmente (single host), usando um arquivo YAML (`docker-compose.yml` ou outro nome). Ele simplifica a orquestração, criando automaticamente uma rede **bridge user-defined** por padrão e permitindo a configuração de serviços, volumes e redes em um único comando.

**Pré-requisitos:** Docker Engine instalado (Compose já incluso). Verifique: `docker compose --version`.

**Referência Oficial:** [Docker Compose CLI](https://docs.docker.com/reference/cli/docker/compose/)

---

## 📋 Sumário da Seção
- [Visão Geral](#visão-geral)
- [Passo a Passo](#passo-a-passo)
- [Comandos Principais](#comandos-principais)
- [Boas Práticas](#boas-práticas)
- [Exemplo Completo com Variáveis](#exemplo-completo-com-variáveis)
- [Observações](#observações)

---

## Visão Geral

**O que é Docker Compose?**
- Ferramenta para orquestração local de múltiplos containers.
- Define serviços (containers), redes e volumes em um arquivo YAML.
- **Não é para produção** (use Swarm/Kubernetes). Ideal para desenvolvimento/testes.
- **Diferença vs Swarm:** Ignora `deploy` (Swarm usa). Suporta `build` (Swarm exige imagens pré-construídas).

**Componentes do YAML:**
- **services**: Containers e suas configs (imagem, portas, volumes, etc.).
- **networks**: Redes personalizadas (default: bridge user-defined).
- **volumes**: Persistência (nomeados ou bind).

**Estrutura Típica:**
```yaml
version: '3.9'
services:
  app:
    image: minha-imagem
    ports:
      - "8080:80"
    volumes:
      - volume1:/data
networks:
  minha-rede:
    driver: bridge
volumes:
  volume1:
```

**Rede Padrão:** Todo `docker compose up` cria uma rede **bridge user-defined** automaticamente (`<nome-projeto>_default`), visível em `docker network ls`.

---

## Passo a Passo

1. **Definir o Ambiente (Opcional):**
   - Crie um `Dockerfile` para construir imagens customizadas.
   - Exemplo:
     ```dockerfile
     FROM nginx:latest
     COPY ./index.html /usr/share/nginx/html/
     ```

2. **Criar o Arquivo `docker-compose.yml`:**
   - Defina serviços, redes e volumes.
   - Exemplo mínimo:
     ```yaml
     version: '3.9'
     services:
       web:
         image: nginx
         ports:
           - "8080:80"
     ```

3. **Executar:**
   ```bash
   docker compose up -d  # Detached (background)
   ```
4. **Ajuda:**
   ```bash
   docker compose up --help
docker compose --help
docker compose down --help
docker compose pull --help

   ```

**OBS:** O comportamento padrão do docker compose up é anexado (simulando a necessidade de `-it` ou `attach` em um contexto de servidor ou "de baixo dos panos") e que você usa o -d para desatachar.

---

## Comandos Principais

### Inicializar/Parar/Remover
| Comando | Descrição | Exemplo |
|---------|-----------|---------|
| `docker compose up` | Cria e inicia serviços, redes, volumes | `docker compose up -d` (detached) |
| `docker compose -f <arquivo>.yml up` | Usa arquivo YAML customizado | `docker compose -f meu-projeto.yml up -d` |
| `docker compose down` | Para e remove serviços/redes | `docker compose down` |
| `docker compose down -v` | Remove também volumes nomeados | `docker compose -f meu-projeto.yml down -v` |
| `docker compose down --rmi all` | Remove imagens também | `docker compose down -v --rmi all` |
| `docker compose stop/start` | Para/inicia serviços sem remover | `docker compose stop` |
| `docker compose stop/start` | Para/inicia serviços sem remover - projeto específico | `docker compose -f meu-projeto.yml start` |

**OBS:** Volumes **bind** (host) não são removidos com `-v`.

### Build e Pull
| Comando | Descrição | Exemplo |
|---------|-----------|---------|
| `docker compose build` | Constrói imagens (se `build` no YAML) | `docker compose build` |
| `docker compose up --build` | Força rebuild antes de iniciar | `docker compose up -d --build` |
| `docker compose up -f <arquivo>.yml --build`| Usa YAML, constrõe a(s) imagem(ns) e sobe os containers - rebuild | `docker compose up -f meu-projeto.yaml --build` |
| `docker compose pull` | Baixa imagens do registry | `docker compose -f meu-projeto.yml pull` |

**OBS:** Se não houver `build` no YAML, `docker compose build` é ignorado.

### Monitoramento e Inspeção
| Comando | Descrição | Exemplo |
|---------|-----------|---------|
| `docker compose ps` | Lista serviços (containers) | `docker compose -f meu-projeto.yml ps -a` ou `docker compose -f meu-projeto.yml ps` (ou ls) |
| `docker compose ls` | Lista projetos Compose | `docker compose ls -a` |
| `docker compose logs` | Logs de todos os serviços | `docker compose logs -f` (real-time) |
| `docker container logs <nome>` | Logs de um container específico e contínuo | `docker container logs meu-webserver -f` |
| `docker compose -f <arquivo>.yml logs -f`| Usa YAML, logs de um container específico e contínuo |`docker compose -f docker-compose-v2.yml logs -f` |
| `docker container ls` | Container específico para status | `docker container ls`

### Escalar Serviços
```bash
# Escala para 3 réplicas - método legado
docker compose scale web=3
# Ou via up - método recomendado e atual
docker compose up -d --scale web=3
# padrão
docker compose up -d --scale <nome-serviço>=x
#com arquivo específico
docker compose -f meu-projeto.yml up -d --scale webserver=5
# Verificar
docker compose ps
```

⚠️ **ATENÇÃO:**
- **Portas no YAML causam conflito ao escalar** (ex: `8080:80`). Use load balancer (ex: Nginx/Traefik) ou remova portas do YAML.
- `EXPOSE` no Dockerfile não causa conflito, pois não faz bind de portas.

### Redes
- **Padrão:** Rede `bridge` criada automaticamente (`<projeto>_default`).
- **Listar:**
  ```bash
  docker network ls
  # Exemplo saída:
  NETWORK ID     NAME                 DRIVER    SCOPE
  c7b4264b1dc2   bridge               bridge    local
  e805e489ebaa   meu-projeto_default  bridge    local
  ```

---

## Boas Práticas

1. **Use Versão Específica no YAML**:
   - Ex: `version: '3.9'` (ver [Compose Specification](https://docs.docker.com/compose/compose-file/)).
2. **Nomenclatura de Imagens**:
   - Sem `build`, nome da imagem é `<pasta>_<serviço>` (ex: `meu-projeto_web`).
3. **Evite Portas no YAML para Escala**:
   - Use load balancer ou redes overlay em produção.
4. **.env para Variáveis**:
   - Mantenha senhas/segredos fora do YAML.
   - Exemplo `.env`:
     ```
     DB_PASS=12345678
     DB_USER=wpuser
     ```
5. **Limpe Recursos**:
   ```bash
   docker compose down -v --rmi all
   ```
6. **Use `depends_on` com Healthchecks**:
   - Garante ordem de inicialização.
   - Exemplo:
     ```yaml
     depends_on:
       db:
         condition: service_healthy
     ```
7. **Evite `docker-compose` (Legado)**:
   - Use `docker compose` (sem hífen, nova CLI).

---

## Exemplo Completo com Variáveis

### Cenário: WordPress + MySQL
- **Objetivo:** Deploy de WordPress com MySQL, usando variáveis em `.env` para configuração segura.
- **Estrutura de Arquivos:**
  ```
  meu-projeto/
  ├── .env
  ├── docker-compose.yml
  ├── Dockerfile
  └── html/index.html
  ```

### Arquivos

**.env:**
```env
DB_USER=wpuser
DB_PASS=12345678
DB_NAME=wordpress
DB_HOST=db
WP_PORT=8080
```

**Dockerfile (WordPress customizado):**
```dockerfile
FROM wordpress:latest
COPY ./html /var/www/html
```

**docker-compose.yml:**
```yaml
version: '3.9'
services:
  wordpress:
    build: .  # Usa Dockerfile local
    image: meu-projeto_wordpress
    ports:
      - "${WP_PORT}:80"  # Porta via .env
    environment:
      WORDPRESS_DB_HOST: ${DB_HOST}
      WORDPRESS_DB_USER: ${DB_USER}
      WORDPRESS_DB_PASSWORD: ${DB_PASS}
      WORDPRESS_DB_NAME: ${DB_NAME}
    depends_on:
      db:
        condition: service_healthy
    networks:
      - wp-net
  db:
    image: mysql:8.0
    volumes:
      - wp-data:/var/lib/mysql
    environment:
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASS}
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      retries: 5
    networks:
      - wp-net
volumes:
  wp-data:
    driver: local
networks:
  wp-net:
    driver: bridge
```

**docker-compose.yml: Versão Testada/Funcional/Comentada**
```yaml
#version: '3.8'     
name: ProjWordpress  

#docker volume create mysql_db (cria uma volume nomeado)
volumes:
  mysql_db:
  wordpress:  

# docker network create wordpress_net (crio uma rede bridge user defined - com resolucao nomes (DNS), on-fly desatacha ou atacha em running, etc)
networks:
  wordpress_net:

services: 

  #docker container run -it --name wordpress -p 80:80 -e WORDPRESS_DB_HOST=db -e WORDPRESS_DB_USER=wpuser -e WORDPRESS_DB_PASSWORD=12345678 -e WORDPRESS_DB_NAME=wordpress wordpress
  wordpress:  
    image: wordpress    
    hostname: webserver 
    ports: 
      - 8080:80
    restart: always 
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wpuser 
      WORDPRESS_DB_PASSWORD: 12345678
      WORDPRESS_DB_NAME: wordpress 
    
    networks:
      - wordpress_net
    depends_on:
      - db

    volumes:
      - wordpress:/var/www/html    

  #docker container run -it --hostname db -p 3306:3306 -v mysql_db:/var/lib/mysql -e MYSQL_DATABASE=wordpress -e MYSQL_USER=wpuser -e MYSQL_PASSWORD=cursdodockerdca -e MYSQL_RANDOM_ROOT_PASSWORD='1' mysql
  db:
    image: mysql:8.0
    restart: always
    volumes:
      - mysql_db:/var/lib/mysql
    #ports:
    # - 3306:3306   #comentei o ports, pois wordpress e db estao na mesma rede - somente nome ja basta
    environment:
      MYSQL_DATABASE: wordpress 
      MYSQL_USER: wpuser 
      MYSQL_PASSWORD: 12345678 
      MYSQL_RANDOM_ROOT_PASSWORD: '1' 

    networks:
      - wordpress_net
```

### Comandos
1. **Iniciar:**
   ```bash
   docker compose up -d
   ```
2. **Verificar:**
   ```bash
   docker compose ps
   docker compose logs -f
   # Acesse: http://localhost:8080
   ```
3. **Escalar WordPress (sem portas no YAML):**
   - Remova `ports` do YAML e use um proxy (ex: Nginx, traefik).
   ```bash
   docker compose up -d --scale wordpress=3
   ```
4. **Parar/Remover:**
   ```bash
   docker compose down -v
   ```

---

## Observações

1. **Rede Bridge Automática:**
   - Cada `docker compose up` cria uma rede `<projeto>_default` (ex: `meu-projeto_default`).
   - Containers se comunicam via nome do serviço (ex: `wordpress` acessa `db`).

2. **Escalabilidade:**
   - Evite portas fixas no YAML para escalar.
   - Exemplo com proxy:
     ```yaml
     services:
       proxy:
         image: nginx
         ports:
           - "80:80"
       app:
         image: meu-app
         # Sem ports!
     ```

3. **Variáveis de Ambiente:**
   - Use `.env` ou passe diretamente:
     ```bash
     DB_PASS=12345678 docker compose up -d
     ```

4. **Compose vs Swarm:**
   - **Compose**: Local, suporta `build`, ignora `deploy`.
   - **Swarm**: Cluster, exige `image`, usa `deploy`.

5. **Logs e Debugging:**
   - Use `docker compose logs -f` para todos os serviços.
   - Para container específico: `docker container logs <nome> -f`.

6. **Healthchecks:**
   - Essenciais para `depends_on` e inicialização confiável.
   - Exemplo MySQL:
     ```yaml
     healthcheck:
       test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
       interval: 10s
       retries: 5
     ```

7. **Limpeza Completa:**
   ```bash
   docker compose down -v --rmi all
   docker system prune -a --volumes -f
   ```

---

**Dica Final:** Sempre valide o YAML com `docker compose config` antes de executar. Teste em lab (ex: Vagrant) e use `docker stats` para monitorar recursos. 🚀  
**Próximo Passo:** Integre com Swarm para orquestração em cluster (veja [Seção Swarm](#06---swarm)).

---
## 🐝 Swarm

**Objetivo:** Orquestrar containers em um cluster com Docker Swarm, gerenciando nodes, serviços, tasks, redes overlay e stacks. Esta seção organiza os comandos, conceitos e exemplos do curso Docker DCA de Caio Delgado, complementando com boas práticas, sintaxe técnica aprimorada e exemplos claros para facilitar a revisão e aplicação prática.

**Conceito Principal:** Docker Swarm é o orquestrador nativo do Docker para gerenciar clusters de containers em múltiplos nós (máquinas físicas ou VMs). Ele utiliza o algoritmo **Raft Consensus** para garantir consistência entre nodes managers e suporta serviços **replicados** (escala definida) e **globais** (um por nó). Swarm é ideal para ambientes de produção simples, mas não oferece autoscaling nativo como Kubernetes.

**Pré-requisitos:** Docker Engine instalado (Swarm incluso). Setup de lab com 1 manager e 2+ workers (ex: VMs Vagrant). Verifique: `docker swarm init`.

**Referência Oficial:** [Docker Swarm](https://docs.docker.com/engine/swarm/)


## 📋 Sumário da Seção
- [Visão Geral](#visão-geral)
- [Conceitos Fundamentais](#conceitos-fundamentais)
- [Passo a Passo](#passo-a-passo)
- [Comandos Principais](#comandos-principais)
- [Boas Práticas](#boas-práticas)
- [Exemplo Completo com Stacks e Variáveis](#exemplo-completo-com-stacks-e-variáveis)
- [Observações](#observações)


## Visão Geral

**O que é Docker Swarm?**
- Orquestrador para gerenciar clusters de containers em múltiplos **nodes** (máquinas físicas/VMs).
- **Nodes**:
  - **Manager**: Gerencia o cluster (Raft Consensus, orquestração).
  - **Worker**: Executa containers (tasks).
- **Serviços**: Estado desejado (ex: 5 réplicas de Nginx).
- **Tasks**: Unidades atômicas (containers) agendadas (scheduler) para atingir o serviço.
- **Redes Overlay**: Comunicação global entre nodes.
- **Stacks**: Arquivos YAML (semelhantes ao Docker Compose) para deploy em cluster.

**Raft Consensus**:
- Garante consistência entre managers.
- Tolerância a falhas: `(N-1)/2` (N = número de managers).
- Quorum mínimo: `(N/2)+1`.
- **Recomendação**: Use 3 ou 5 managers (nunca pares para evitar **split-brain**). Exemplos Possíveis: 1, 3, 5, 7, 9,...
- [Simulação Raft](https://raft.github.io/) | [Explicação Visual](https://thesecretlivesofdata.com/raft/).

**Cenário do Curso**:
- 1 manager (`master.docker-dca.example`) + 2 workers (`node01`, `node02`) + 1 registry (`registry.docker-dca.example:5000`).
- Managers podem rodar containers, mas focam em gerenciamento.

---

## Conceitos Fundamentais

### Nodes
- **Definição**: Máquinas (físicas/VMs) que fornecem CPU, RAM, disco e rede.
- **Tipos**:
  - **Manager**: Lidera o cluster (Raft). Executa comandos `docker service/stack`.
  - **Worker**: Executa tasks (containers).
- **Estados**: Active (operacional) ou Drain (manutenção, sem novas tasks).

### Serviços
- **Definição**: Estado desejado (ex: "quero 3 Nginx rodando").
- **Tipos**:
  - **Replicado** (padrão): Define número exato de réplicas.
    - Ex: 5 instâncias de uma API.
    - Uso: Aplicações web, filas.
  - **Global**: Uma task por nó.
    - Ex: Agentes de monitoramento (Prometheus Node Exporter).
    - Uso: Logs, segurança, manutenção.
- **Scheduler:** Situado no nó `manager`, ele decide em qual nó `worker` a `task` deve ser executada garantido seu estado.
- **Tasks**: Containers agendados para cumprir o serviço.
  - Estados: `NEW`, `PENDING`, `RUNNING`, `FAILED`, `COMPLETE`, `ORPHANED`, `STARTING`, `REMOVE`, `REJECTED`, etc. ([Docs](https://docs.docker.com/engine/swarm/how-swarm-mode-works/swarm-task-states/)).
  - Ciclo comum: `ASSIGNED → PREPARED → RUNNING`.

  **Serviço Replicado vs Serviço Global:**

 ![Service Replicated](https://docs.docker.com/engine/swarm/images/replicated-vs-global.webp)

**Fonte:** [https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/](https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/)

### Redes
| Tipo | Driver | Escopo | Propósito |
|------|--------|--------|-----------|
| ingress | overlay | Global | Balanceamento de carga (Swarm routing mesh). |
| custom overlay | overlay | Global | Comunicação interna entre serviços. |
| bridge | bridge | Local | Containers no mesmo nó. (single host)|
| host | host | Local | Raramente ou nunca usada em ambiente de cluster (sem propósito) |

- **Routing Mesh**: Todas as portas publicadas (`-p`) são acessíveis em qualquer nó via rede `ingress`.

⚠️ **ATENÇÃO:** No single host (docker run/compose), a colisão de portas é uma limitação devido ao vínculo direto com o sistema operacional do host (redes bridge default ou bridge user-defined). No Docker Swarm, redes overlay com routing mesh superam isso, permitindo que múltiplas réplicas compartilhem a mesma porta publicada em qualquer nó do cluster, sem conflitos. Para escalabilidade, use sempre o modo padrão (overlay) e evite mode=host (único cenário para conflito de portas no Swarm).

> 💡 **Dica Importante:** Para escalar serviços no Docker Compose (rede bridge local) sem colisão de portas, você tem um opção fora a rede overlay, portas `Aleatórias/Efemerais`. Não especifique a porta externa. O Docker Compose irá escolher uma porta aleatória e livre no Host. Exemplo:
```yaml
ports:
  - "80" # O Docker escolhe uma porta livre no Host e a mapeia para a porta 80 do contêiner.
```  

### Stacks
- Arquivos YAML (como Docker Compose, mas com `deploy`).
- **Diferença vs Compose**:
  - Swarm: Usa `image` (registry), ignora `build`.
  - Compose: Suporta `build`, ignora `deploy`.

---

## Passo a Passo

1. **Inicializar Cluster:**
   - Em um manager:
     ```bash
     docker swarm init --advertise-addr <IP_MANAGER>
     # Exemplo:
     docker swarm init --advertise-addr 192.168.15.100  # Evite DNS
     ```
   - Gera tokens para workers/managers.

2. **Adicionar Nodes:**
   - Workers:
     ```bash
     docker swarm join --token <TOKEN_WORKER> 192.168.15.100:2377
     ```
   - Managers (se necessário):
     ```bash
     docker swarm join --token <TOKEN_MANAGER> 192.168.15.100:2377
     ```

3. **Criar Serviço:**
   ```bash
   docker service create --name webserver --replicas 3 registry.docker-dca.example:5000/nginx
   docker service create --name hostapp --publish published=80,target=80,mode=host --replicas 3 nginx # rede modo host (single host - stack network host)
   docker service create --name webserver --replicas 3 --publish 80:80 --network dca-overlay --mount source=volume_nfs,target=/usr/share/nginx/html regitry.docker-dca.example:5000/nginx
   ```

4. **Escalar e Monitorar:**
   ```bash
   docker service scale webserver=5
   docker service ps webserver
   docker service logs -f webserver
   docker node ls  # watch docker node ls (real time terminal)
   ```

5. **Promover e Despromover:**
   ```bash
   #promover um node worker a manager
   docker node promote <nome-do-hostname-node ou ID>
   #despromover
   docker node demote <nome-do-hostname-node ou ID>
   ```


6. **Deploy Stack:**
   ```bash
   docker stack deploy -c webserver.yml minha-stack
   ```

---

## Comandos Principais

### Cluster e Nodes
| Comando | Descrição | Exemplo |
|---------|-----------|---------|
| `docker swarm init --advertise-addr <IP>` | Inicia cluster | `docker swarm init --advertise-addr 192.168.15.100` |
| `docker swarm join-token manager/worker` | Gera token para novos nodes | `docker swarm join-token worker` |
| `docker swarm join --token <TOKEN> <IP>:2377` | Adiciona node | `docker swarm join --token SWMTKN-12232j4h3h43kjh4... 192.168.15.100:2377` |
| `docker node ls` | Lista nodes (só em manager) | `watch docker node ls` (real-time) |
| `docker node promote/demote <ID>`|HOSTNAME>` | Promove/rebaixa node | `docker node promote node01` |
| `docker node update --availability drain/active` | Drena/ativa node (manutenção) | `docker node update node01 --availability drain` |
| `docker node inspect <ID ou HOSTNAME>` | Detalhes do node | `docker node inspect node01 --pretty` |

### Serviços
| Comando | Descrição | Exemplo |
|---------|-----------|---------|
| `docker service create` | Cria serviço | `docker service create --name pingtest --replicas 1 alpine ping google.com` ou `docker service create --name agente-log --mode global minha/log-collector` |
| `docker service ls/ps/logs/inspect` | Lista/info/logs | `docker service ps pingtest --no-trunc` ou `docker service inspect webserver --pretty`|
| `docker service scale <nome>=N` | Escala réplicas | `docker service scale pingtest=3` |
| `docker service update` | Atualiza configs | `docker service update --replicas 5 webserver` |
| `docker service rm <nome>` | Remove serviço | `docker service rm pingtest` |
| `docker service <help>` | Ajuda | `docker service --help` |

### Secrets
```bash
# Criar
echo "senha" | docker secret create senha_db -
# Listar/Inspecionar
docker secret ls
docker secret inspect --pretty senha_db
# Usar em serviço
docker service create --name mysql --secret senha_db -e MYSQL_ROOT_PASSWORD_FILE=/run/secrets/senha_db mysql:5.7   # /run/secrets/ é tmpfs no Linux - Systemd
```

### Redes Overlay
```bash
docker network create -d overlay dca-overlay
docker network inspect dca-overlay
docker service create --name webserver --network dca-overlay --publish 80:80 nginx
```

### Stacks
```bash
docker stack deploy -c webserver.yml minha-stack
docker stack ls/ps/services minha-stack
docker stack rm minha-stack
```

### Monitoramento
```bash
docker stats  # Recursos
docker service logs -f <nome>  # Logs
docker node ls
docker stack ls
docker stack ps <nome-service>
docker service ls/ps
docker service inspect <nome-service> | jq
docker secret inspect --pretty <nome-secret>
docker container ls/ps
docker image ls
docker network ls
docker network inspect <nome ou ID>
```

---

## Boas Práticas

1. **Use 3 ou 5 Managers**:
   - Evite números pares para prevenir **split-brain**.
   - Exemplo: 3 managers toleram 1 falha (`(3-1)/2`).

2. **Prefira IPs a Nomes**:
   - Evite falhas de DNS em `docker swarm init --advertise-addr <IP>`.

3. **Configuração Consistente**:
   - Replique `/etc/docker/daemon.json` em todos os nodes.
   - Exemplo (registry inseguro):
     ```json
     {
       "insecure-registries": ["registry.docker-dca.example:5000"]
     }
     ```
   - Reinicie: `systemctl restart docker`.

4. **Use Redes Overlay**:
   - Rede `ingress` para balanceamento.
   - Custom overlay para comunicação interna.

5. **Secrets para Dados Sensíveis**:
   - Armazene em `/run/secrets` (tmpfs).
   - Evite variáveis de ambiente claras.

6. **Manutenção sem Downtime**:
   - Use `docker node update --availability drain` para realocar tasks.

7. **Escalabilidade**:
   - Evite `--publish mode=host` (limita réplicas por nó).
   - Use modo padrão (`overlay`) para evitar conflitos de porta.

8. **Labels para Controle**:
   ```bash
   docker node update --label-add location=us-east-1 node01
   ```
   - No YAML:
     ```yaml
     deploy:
       placement:
         constraints:
           - node.labels.location==us-east-1
     ```

9. **Comandos Executados Somente em Nós Managers:**

   - Os comandos `docker swarm`, `docker node`, `docker service`, `docker config` e `docker stack` (entre outros) só podem ser executados em nós managers, pois eles gerenciam o estado do cluster, que é exclusivo dos managers via Raft Consensus. Workers apenas executam tasks (containers) e não têm acesso à API de gerenciamento (segurança). Comandos locais (docker container, docker image, docker stats, etc.) podem ser usados em qualquer nó para interagir com recursos locais. 
   Se tentarmos executa-los em nós workers, recebemos o seguinte erro:

   ```bash
   Error response from daemon: This node is not a swarm manager
   ```

> 💡 Dica Final: Sempre verifique o papel do nó com `docker node inspect self --pretty` antes de executar comandos. Use `watch docker node ls` em um manager para monitoramento em tempo real.


**OBS:** Em managers como works (managers também podem conter tasks) também podemos utilizar comandos locais (container ls, logs, exec)

### Exemplo Prático
**Cenário**: Cluster com 1 manager (`master`) e 2 workers (`node01`, `node02`).

1. **Em Manager**:
   ```bash
   docker swarm init --advertise-addr 192.168.15.100
   docker node ls  # Lista master, node01, node02
   docker service create --name webserver --replicas 3 nginx
   docker stack deploy -c webserver.yml minha-stack
   ```

2. **Em Worker**:
   ```bash
   docker node ls  # Erro: "This node is not a swarm manager"
   docker service ls  # Erro: "This node is not a swarm manager"
   docker container ls  # Lista containers locais (tasks)
   ```

3. **Promover Worker a Manager** (em manager):
   ```bash
   docker node promote node01
   docker node ls  # node01 agora é manager
   ```
### Resumo dos Comandos e Onde São Executados
| Comando | Executado em | Propósito |
|---------|--------------|-----------|
| `docker swarm` | Managers (exceto `join`) | Gerencia cluster (init, join-token) |
| `docker node` | Managers | Gerencia nós (ls, promote, update) |
| `docker service` | Managers | Gerencia serviços (create, scale, rm) |
| `docker stack` | Managers | Gerencia stacks (deploy, rm) |
| `docker secret` | Managers | Gerencia secrets (create, ls) |
| `docker config` | Managers | Gerencia configs (create, ls) |
| `docker network` | Managers (para overlay) | Gerencia redes (create -d overlay) |
| `docker system` | Qualquer nó | Informações e limpeza (info, prune) |
| `docker info` | Qualquer nó | Detalhes do Swarm/nó |
| `docker container` | Qualquer nó | Interage com containers locais |
| `docker events` | Qualquer nó | Monitora eventos |
| `docker inspect` | Managers (serviços/nodes), qualquer nó (containers) | Inspeciona objetos |
| `docker volume` | Qualquer nó (locais), managers (serviços) | Gerencia volumes |


---

## Exemplo Completo com Stacks e Variáveis

### Cenário: WordPress + MySQL em Swarm
- **Objetivo:** Deploy de uma stack WordPress com MySQL, usando registry privado, rede overlay, secrets e constraints.
- **Estrutura**:
  ```
  stack/
  ├── .env
  ├── wordpress.yml
  ```

### Arquivos

**.env:**
```env
DB_USER=wpuser
DB_PASS=12345678
DB_NAME=wordpress
WP_PORT=8080
```

**wordpress.yml:**
```yaml
version: '3.9'
name: wordpress-stack
services:
  wordpress:
    image: registry.docker-dca.example:5000/wordpress
    ports:
      - "${WP_PORT}:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: ${DB_USER}
      WORDPRESS_DB_PASSWORD_FILE: /run/secrets/db_pass
      WORDPRESS_DB_NAME: ${DB_NAME}
    secrets:
      - db_pass
    networks:
      - wp-overlay
    deploy:
      mode: replicated
      replicas: 2
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: "1"
          memory: 60M
        reservations:
          cpus: "0.5"
          memory: 30M
  db:
    image: registry.docker-dca.example:5000/mysql
    volumes:
      - wp-data:/var/lib/mysql
    environment:
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD_FILE: /run/secrets/db_pass
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    secrets:
      - db_pass
    networks:
      - wp-overlay
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role==manager
      restart_policy:
        condition: on-failure
secrets:
  db_pass:
    external: true
volumes:
  wp-data:
networks:
  wp-overlay:
    driver: overlay
```

### Comandos
1. **Configurar Registry Inseguro (todos os nodes):**
   ```bash
   echo '{"insecure-registries": ["registry.docker-dca.example:5000"]}' > /etc/docker/daemon.json
   systemctl restart docker
   ```

2. **Setup Registry:**
   ```bash
   docker container run -d --name registry -p 5000:5000 registry:2
   docker image pull wordpress
   docker image tag wordpress registry.docker-dca.example:5000/wordpress
   docker image push registry.docker-dca.example:5000/wordpress
   # Verificar as APIs do registry
   curl -s http://registry.docker-dca.example:5000/v2/_catalog | jq
   curl -s http://registry.docker-dca.example:5000/v2/wordpress/tags/list | jq
   ```

3. **Criar Secret:**
   ```bash
   echo "${DB_PASS}" | docker secret create db_pass -
   ```

4. **Inicializar Swarm:**
   ```bash
   docker swarm init --advertise-addr 192.168.15.100
   docker swarm join --token <TOKEN> 192.168.15.100:2377  # Em workers
   ```

5. **Deploy Stack:**
   ```bash
   docker stack deploy -c wordpress.yml wordpress-stack # c = --compose-file
   docker stack services wordpress-stack
   docker stack ps wordpress-stack
   ```

6. **Teste de Carga:**
   ```bash
   apt-get install apache2-utils -y
   ab -n 10000 -c 100 http://master.docker-dca.example:8080/
   ```

7. **Escalar:**
   ```bash
   docker service scale wordpress-stack_wordpress=6
   ```

8. **Remover:**
   ```bash
   docker stack rm wordpress-stack
   ```
---

## Observações

1. **Swarm vs Kubernetes**:
   - Swarm: Mais simples, nativo, sem autoscaling.
   - Kubernetes: Complexo, com HPA (Horizontal Pod Autoscaling).

2. **Routing Mesh**:
   - Portas publicadas (`-p`) são acessíveis em qualquer nó via rede `ingress`.
   - Exemplo: `http://node01:8080` acessa WordPress mesmo se o container está em `node02`.

3. **Manutenção sem Downtime**:
   - Use `drain` para realocar tasks:
     ```bash
     docker node update node01 --availability drain
     docker node update node01 --availability active
     ```

4. **Modo Host**:
   - Limita réplicas por nó devido a conflitos de porta.
   - Exemplo:
     ```bash
     docker service create --publish mode=host,target=80,published=80 nginx
     ```
   - **Evite** em produção; use overlay.

5. **Registry Privado**:
   - Configure `insecure-registries` em todos os nodes.
   - Use API para inspeção:
     ```bash
     curl -s http://registry.docker-dca.example:5000/v2/_catalog | jq
     ```

6. **Tasks e Estados**:
   - Verifique com `docker service ps --no-trunc` para erros detalhados.
   - Estados comuns: `PENDING`, `RUNNING`, `FAILED`.

7. **Labels para CI/CD**:
   - Controle onde services rodam:
     ```yaml
     constraints:
       - node.labels.os==ubuntu18.04
     ```
8. **Labels Exemplos comandos:**
   ```bash
     docker node update --label-add disk=ssd node01.docker-dca.example
     docker node update --label-add chave=valor node01.docker-dca.example
     docker node update --label-add location=us-east-1 node01.docker-dca.example
   ```


**Dica Final:** Sempre valide stacks com `docker stack deploy --dry-run`. Use `docker stats` e `docker service logs -f` para monitoramento. Teste em lab antes de produção! 🚀  

---
## Tutorial: Teste de Estresse e Escalonamento Manual no Docker Swarm com Apache Benchmark
 
**Objetivo:** Demonstrar como usar o `apache benchmark` (AB) para simular carga em um serviço Swarm e entender o escalonamento manual, já que o Swarm não suporta autoscaling nativo como o HPA do Kubernetes.

**Conceito Principal:**  
O Docker Swarm permite escalonar serviços manualmente com `docker service scale`, mas não ajusta réplicas automaticamente com base em métricas (diferente do Horizontal Pod Autoscaler - HPA - do Kubernetes). Este tutorial usa o AB para gerar carga e observar o comportamento do cluster, ajustando réplicas à mão para avaliar desempenho.

### Pré-requisitos
- Cluster Swarm configurado (1 manager + 2+ workers, ex.: `master.docker-dca.example`, `node01`, `node02`).
- Serviço rodando (ex.: Nginx ou WordPress, como no seu rascunho).
- Acesso SSH aos nós para instalação de ferramentas e monitoramento.

### Passo a Passo

#### 1. Instalar Ferramentas Apache
O Apache fornece o pacote `apache2-utils`, que inclui o `ab` (Apache Benchmark), útil para testes de carga. Ferramentas como `k6` e `Apache JMeter` são mais avançadas, mas `ab` é suficiente para este tutorial.

- **Comando de Instalação** (execute em qualquer nó, ex.: manager):
  ```bash
  sudo apt-get update
  sudo apt-get install apache2-utils -y
  ```
- **Verificação**:
  ```bash
  ab -V  # Confirma versão (ex.: ab 2.4.41)
  ```

#### 2. Configurar um Serviço de Teste no Swarm
Crie um serviço simples (ex.: Nginx) para testar a carga.

- **Criar Rede Overlay** (em manager):
  ```bash
  docker network create -d overlay test-overlay
  ```

- **Criar Serviço com 1 Réplica Inicial** (em manager):
  ```bash
  docker service create --name webserver \
    --replicas 1 \
    --publish 8080:80 \
    --network test-overlay \
    nginx
  ```

- **Verificar**:
  ```bash
  docker service ps webserver  # Confirma réplica em um nó
  curl http://master.docker-dca.example:8080  # Testa acesso (página padrão Nginx)
  ```

#### 3. Realizar Teste de Carga com `ab`
Use o `ab` para simular tráfego e observar o desempenho.

- **Comando de Teste**:
  ```bash
  ab -n 10000 -c 100 http://master.docker-dca.example:8080/
  ```
  - `-n 10000`: Total de requisições (10.000).
  - `-c 100`: Concorrência (100 requisições simultâneas).
  - **Saída Esperada**: Estatísticas como tempo por requisição, taxa de transferência e erros (se houver).

- **Interpretação da Saída**:
  - `Time per request`: Média por requisição (em ms).
  - `Failed requests`: Indica falhas (pode sugerir sobrecarga).
  - Exemplo:
    ```
    Requests per second:    150.50 [#/sec] (mean)
    Time per request:       664.523 [ms] (mean)
    Failed requests:        0
    ```

#### 4. Monitorar Recursos
Use `docker stats` para verificar o impacto da carga nos containers.

- **Em Manager ou Worker (onde a réplica está)**:
  ```bash
  docker stats  # Mostra CPU/memória do container webserver
  ```
- **Observação**: Com 1 réplica, o desempenho pode degradar sob alta carga (ex.: latência alta ou falhas).

#### 5. Escalonar Manualmente
Como o Swarm não tem autoscaling, ajuste as réplicas manualmente com base nos resultados do teste.

- **Aumentar Réplicas** (em manager):
  ```bash
  docker service scale webserver=3  # Adiciona 2 réplicas
  docker service ps webserver  # Verifica distribuição
  ```

- **Repetir Teste de Carga**:
  ```bash
  ab -n 10000 -c 100 http://master.docker-dca.example:8080/
  ```
  - **Esperado**: Menor tempo por requisição e maior taxa de transferência, devido ao balanceamento via routing mesh.

#### 6. Analisar Resultados
Compare os resultados antes e depois do escalonamento:
- **1 Réplica**: Alta latência, possível falhas.
- **3 Réplicas**: Melhor desempenho, distribuição de carga entre nós.

- **Exemplo de Comparação**:
  - 1 Réplica: 664 ms/requisição, 150 req/s.
  - 3 Réplicas: 220 ms/requisição, 450 req/s.

### 7. Limpeza
Remova o serviço e a rede após os testes.

- **Comandos** (em manager):
  ```bash
  docker service rm webserver
  docker network rm test-overlay
  docker system prune -f  # Opcional: limpa recursos não usados
  ```

### Observações e Limitações
- **Swarm vs. Kubernetes**: O Swarm requer escalonamento manual (`docker service scale`), enquanto o Kubernetes usa HPA (Horizontal Pod Autoscaler) para ajustar réplicas automaticamente com base em métricas (CPU/memória).
- **Routing Mesh**: A porta `8080` é acessível em qualquer nó devido ao routing mesh, mesmo que a réplica esteja em outro nó (ex.: `node01` ou `node02`).
- **Ferramentas Alternativas**: Para testes mais robustos, considere `k6` (moderno, scriptável) ou `Apache JMeter` (interface gráfica).
- **Monitoramento**: Use `docker stats` ou integre Prometheus/Grafana (mencionado no seu rascunho) para métricas detalhadas.


### Boas Práticas
- **Teste Incremental**: Comece com 1 réplica, escalone gradualmente (ex.: 2, 3, 5) e monitore.
- **Validação**: Use `docker service ps --no-trunc` para verificar o estado das tasks.
- **Segurança**: Execute testes em um ambiente de lab (ex.: Vagrant/VMs) para evitar impacto em produção.
- **Documentação**: Registre os resultados do `ab` (tempo, falhas) para análise futura.


### Exemplo de Saída `ab` (Referência)
```plaintext
This is ApacheBench, Version 2.4.41 ...
Server Software:        nginx/1.21.6
Server Hostname:        master.docker-dca.example
Server Port:            8080

Document Path:          /
Document Length:        612 bytes

Concurrency Level:      100
Time taken for tests:   66.452 seconds
Complete requests:      10000
Failed requests:        0
Total transferred:      6120000 bytes
Requests per second:    150.50 [#/sec] (mean)
Time per request:       664.523 [ms] (mean)
```

### Próximos Passos
- Teste com uma stack (ex.: WordPress + MySQL) e adicione secrets/volumes.
- Integre monitoramento com Prometheus para análise em tempo real.
- Explore `k6` ou `JMeter` para cenários mais complexos.


---
### 🧠 Analogia da Hierarquia de Comandos no Ecossistema Docker


**Objetivo:** Explicar a progressão de abstração nos comandos Docker, comparando ambientes **locais (single host)** com **distribuídos (cluster multi-host)**. Essa didática usa a metáfora do **"comando guarda-chuva"** para ilustrar como arquivos de configuração centralizados (YAML) substituem comandos manuais individuais, facilitando a orquestração.

**Conceito Principal:**  
No Docker, comandos de **baixo nível** (simples/manuais) criam recursos isolados, enquanto comandos de **alto nível** (guarda-chuva) usam um arquivo YAML para orquestrar múltiplos recursos com um único comando. Essa substituição é análoga ao passar de `docker run` para `docker compose up` (local) e de `docker service create` para `docker stack deploy` (Swarm/cluster).

**Por Que Essa Analogia?**  
- **Baixo Nível**: Controle granular, mas repetitivo e propenso a erros (ex: configurar rede/volumes manualmente).
- **Alto Nível (Guarda-Chuva)**: Centraliza a configuração em um arquivo YAML, gerenciando ciclo de vida, redes e volumes automaticamente.
- **Transição Local → Cluster**: O mesmo padrão de "substituição" aplica-se, mas com escalabilidade: single host → multi-host (overlay, routing mesh).


#### Hierarquia de Comandos

| Nível de Abstração      | Escopo              | Comando Simples (Baixo Nível) | Comando Guarda-Chuva (Alto Nível) |
|--------------------------|---------------------|-------------------------------|-----------------------------------|
| Contêineres Individuais | Local (Single Host) | `docker container run`       | `docker compose up`              |
| Serviços Distribuídos   | Cluster (Multi-Host)| `docker service create`      | `docker stack deploy`            |


### Explicação Detalhada da Analogia

#### Relacionamento: Docker Compose (Local) vs. Docker Swarm/Stack (Cluster)

| Aspecto              | Docker Compose (Local)                                                                 | Docker Swarm/Stack (Cluster)                                                                   |
|----------------------|----------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| **Comando Simples** | `docker run ...` cria e configura **um único contêiner** de cada vez, sem gerenciamento automático de rede/volumes por nome. | `docker service create ...` cria e configura **um único serviço** no cluster, definindo réplicas, redes overlay e volumes (manual por serviço). |
| **Comando Guarda-Chuva** | `docker compose up` lê o arquivo `docker-compose.yml` e cria **todos os contêineres, redes e volumes** necessários com um único comando, gerenciando o ciclo de vida local. | `docker stack deploy` lê o arquivo `docker-compose.yml` (ou `stack.yml`) e cria **todos os serviços, redes e volumes** no cluster Swarm, gerenciando a distribuição de tarefas (tasks) entre nós. |
| **Rede Padrão**     | Bridge user-defined (local ao host).                                                   | Overlay (global, com routing mesh para balanceamento e sem colisão de portas).                 |
| **Escalabilidade**  | Limitada a um host; colisão de portas ao escalar (`--scale`).                          | Ilimitada no cluster; routing mesh resolve colisões de portas em múltiplos nós.                |
| **Uso Típico**      | Desenvolvimento/testes locais (ex: WordPress + MySQL em um host).                      | Produção com alta disponibilidade (ex: réplicas distribuídas, secrets, constraints).           |

**Fluxo de Substituição (Guarda-Chuva):**  
1. **Baixo Nível → Alto Nível (Local)**: Em vez de múltiplos `docker run` (um por contêiner), use `docker compose up` com YAML para orquestrar tudo.  
2. **Local → Cluster**: Em vez de múltiplos `docker service create` (um por serviço), use `docker stack deploy` com YAML para orquestrar no Swarm.  
   - **Diferenças Chave no YAML**:  
     - Compose: Suporta `build` (construção local).  
     - Stack: Exige `image` (do registry), usa `deploy` (réplicas, placement, resources).


### Exemplos Práticos

#### 1. **Local (Compose) - Baixo Nível**
```bash
docker container run -d -p 8080:80 --name web1 nginx
docker container run -d -p 3306:3306 --name db1 mysql
# Manual: Rede/volumes separados, risco de colisão de portas.
```

#### 2. **Local (Compose) - Guarda-Chuva**
**docker-compose.yml:**
```yaml
version: '3.9'
services:
  web:
    image: nginx
    ports:
      - "8080:80"
  db:
    image: mysql
    volumes:
      - db-data:/var/lib/mysql
volumes:
  db-data:
```
```bash
docker compose up -d  # Cria web, db, rede bridge, volume.
```

#### 3. **Cluster (Swarm) - Baixo Nível**
```bash
docker service create --name web --replicas 3 --publish 8080:80 nginx
docker service create --name db --replicas 1 mysql
# Manual: Rede overlay implícita, mas sem centralização.
```

#### 4. **Cluster (Swarm) - Guarda-Chuva**
**stack.yml:**
```yaml
version: '3.9'
services:
  web:
    image: nginx
    ports:
      - "8080:80"
    deploy:
      replicas: 3
  db:
    image: mysql
    volumes:
      - db-data:/var/lib/mysql
    deploy:
      replicas: 1
volumes:
  db-data:
networks:
  overlay-net:
    driver: overlay
```
```bash
docker stack deploy -c stack.yml minha-stack  # Distribui no cluster.
```

### Boas Práticas e Observações
- **YAML Centralizado**: Sempre valide com `docker compose config` (local) ou `docker stack deploy --dry-run` (Swarm).
- **Limitações**:
  - Compose: Colisão de portas em escala (`--scale` falha com portas fixas).
  - Stack: Sem `build` (use registry); autoscaling manual.
- **Transição**: Use a mesma lógica de YAML para migrar de local para cluster (adicione `deploy` para Swarm).
- **Extensão**: Integre secrets (`docker secret create`) no YAML do Stack para produção.


---


## 📊 Monitoramento (Prometheus/Grafana/cAdvisor)

### Configuração da Stack de Monitoramento

### Arquivo YAML (`monitoring.yml`)
Este arquivo define uma stack com serviços de monitoramento, usando volumes persistentes, redes overlay e constraints de placement.

```yaml
version: '3.9'
volumes:
  prometheus_data:
  grafana_data:

networks:
  monitoring:
    driver: overlay

services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
    ports:
      - "8080:8080"
    networks:
      - monitoring
    deploy:
      mode: global
      restart_policy:
        condition: on-failure
    # cAdvisor coleta métricas de containers em cada nó

  prometheus:
    image: prom/prometheus
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    depends_on:
      - cadvisor
    networks:
      - monitoring
    deploy:
      placement:
        constraints:
          - node.role == manager
        restart_policy:
          condition: on-failure
    # Prometheus armazena e consulta métricas

  node-exporter:
    image: prom/node-exporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    ports:
      - "9100:9100"
    networks:
      - monitoring
    deploy:
      mode: global
      restart_policy:
        condition: on-failure
    # Node Exporter coleta métricas de sistema (CPU, memória, disco)

  grafana:
    image: grafana/grafana
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    networks:
      - monitoring
    user: "472"  # ID do usuário grafana (segurança)
    deploy:
      placement:
        constraints:
          - node.role == manager
        restart_policy:
          condition: on-failure
    # Grafana visualiza dados do Prometheus
```

- **Explicação**:
  - `cadvisor` e `node-exporter` rodam em modo `global` (uma instância por nó).
  - `prometheus` e `grafana` rodam no manager (via `node.role == manager`) para centralizar dados e interface.
  - `volumes` persistem dados entre reinícios.
  - `networks: monitoring` usa overlay para comunicação no cluster.

### Configuração do Prometheus (`prometheus.yml`)
Crie um diretório `config` com o arquivo `prometheus.yml` para definir os alvos de scraping:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
```

- Coloque este arquivo em `./config/prometheus.yml` no diretório onde executará `docker stack deploy`.


### Implantação da Stack

### Comandos
Execute os seguintes comandos em um **nó manager**:

1. **Deploy da Stack**:
   ```bash
   docker stack deploy -c monitoring.yml monitoring
   ```

2. **Verificar Nós**:
   ```bash
   docker node ls  # Confirma manager e workers
   ```

3. **Listar Stacks**:
   ```bash
   docker stack ls  # Mostra a stack 'monitoring'
   ```

4. **Verificar Serviços e Tasks**:
   ```bash
   docker stack ps monitoring  # Lista todas as tasks (cAdvisor, Prometheus, etc.)
   ```

- **Saída Esperada**:
  - `docker stack ps monitoring` mostrará tasks em cada nó (cAdvisor e Node Exporter em todos, Prometheus e Grafana no manager).


## Acesso e Validação

### Endpoints
- **cAdvisor**: `http://<qualquer-nó>:8080` (métricas de containers por nó).
- **Node Exporter**: `http://<qualquer-nó>:9100` (métricas de sistema por nó).
- **Prometheus**: `http://<manager>:9090` (interface de consulta).
- **Grafana**: `http://<manager>:3000` (painéis, login padrão: `admin/admin`).

### Teste de Funcionamento
1. **Acesse Prometheus**:
   - Abra `http://master.docker-dca.example:9090` no navegador.
   - Use a consulta `up` para verificar se os alvos (cAdvisor, Node Exporter) estão ativos.

2. **Configure Grafana**:
   - Acesse `http://master.docker-dca.example:3000`.
   - Adicione uma fonte de dados apontando para `http://prometheus:9090`.
   - Importe dashboards (ex.: ID 1860 para Node Exporter, 8919 para cAdvisor).


### Monitoramento em Ação

### Exemplo de Uso
1. **Simule Carga** (usando o tutorial anterior):
   ```bash
   ab -n 10000 -c 100 http://master.docker-dca.example:8080/
   ```
   - Execute em um serviço de teste (ex.: Nginx na porta 8080).

2. **Monitore Métricas**:
   - Em Prometheus: Consulte `rate(container_cpu_usage_seconds_total[5m])` ou `node_memory_MemAvailable_bytes`.
   - Em Grafana: Veja gráficos de CPU/memória por nó/container.

3. **Escalone Manualmente** (exemplo):
   ```bash
   docker service scale monitoring_prometheus=2  # Adiciona réplica (se desejar)
   docker stack ps monitoring
   ```

---

### Boas Práticas
- **Segurança**: Altere a senha padrão do Grafana e use `user: 472` para evitar privilégios elevados.
- **Persistência**: Use volumes (`prometheus_data`, `grafana_data`) para evitar perda de dados.
- **Rede**: Certifique-se de que a rede `monitoring` está acessível entre nós.
- **Limpeza**: Após testes, remova com `docker stack rm monitoring` e limpe com `docker system prune -a --volumes -f`.
- **Escalabilidade**: Monitore com `docker stats` em workers para validar carga.


### Observações
- **Sem Autoscaling**: O Swarm não ajusta réplicas automaticamente; use scripts ou CI/CD para automação.
- **Compatibilidade**: O YAML segue o padrão Compose v3.9, compatível com Swarm.
- **Alternativas**: Considere Loki para logs ou Alertmanager para alertas no Prometheus.
- **Contexto DCA**: Alinha-se com seu rascunho (WordPress, `docker stats`) e amplia para monitoramento completo.

---

### Próximos Passos
- Integre alertas com Alertmanager.
- Adicione monitoramento de aplicações específicas (ex.: WordPress).
- Documente os dashboards do Grafana no GitHub.

---

## 🛠️ Tools Docker

**Objetivo:** Explorar ferramentas complementares ao Docker (Swarmpit, Portainer, Harbor e Docker Machine) para gerenciamento, monitoramento e provisionamento de ambientes containerizados.

**Conceito Principal:**  
Ferramentas como Swarmpit, Portainer e Harbor extendem as capacidades do Docker Swarm e standalone, oferecendo GUIs e registros privados. Docker Machine facilita a criação e gerenciamento de hosts Docker em VMs ou remotamente, sendo especialmente útil em setups legados.


### 1. Swarmpit: Interface Gráfica para Docker Swarm

**Descrição:**  
Swarmpit é uma ferramenta leve e amigável com interface gráfica (GUI) para gerenciar e monitorar clusters Docker Swarm. Funciona como um "Lens para Kubernetes", mas focado exclusivamente em Swarm.

**Referência:**
[https://github.com/swarmpit/swarmpit/blob/master/docker-compose.yml](https://github.com/swarmpit/swarmpit/blob/master/docker-compose.yml)

### Configuração da Stack (`docker-compose.yml`)
```yaml
version: '3.3'
services:
  app:
    image: swarmpit/swarmpit:latest
    environment:
      - SWARMPIT_DB=http://db:5984
      - SWARMPIT_INFLUXDB=http://influxdb:8086
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ports:
      - "888:8080"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080"]
      interval: 60s
      timeout: 10s
      retries: 3
    networks:
      - net
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 1024M
        reservations:
          cpus: '0.25'
          memory: 512M
      placement:
        constraints:
          - node.role == manager

  db:
    image: couchdb:2.3.0
    volumes:
      - db-data:/opt/couchdb/data
    networks:
      - net
    deploy:
      resources:
        limits:
          cpus: '0.30'
          memory: 256M
        reservations:
          cpus: '0.15'
          memory: 128M

  influxdb:
    image: influxdb:1.8
    volumes:
      - influx-data:/var/lib/influxdb
    networks:
      - net
    deploy:
      resources:
        limits:
          cpus: '0.60'
          memory: 512M
        reservations:
          cpus: '0.30'
          memory: 128M

  agent:
    image: swarmpit/agent:latest
    environment:
      - DOCKER_API_VERSION=1.35
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - net
    deploy:
      mode: global
      labels:
        swarmpit.agent: 'true'
      resources:
        limits:
          cpus: '0.10'
          memory: 64M
        reservations:
          cpus: '0.05'
          memory: 32M

networks:
  net:
    driver: overlay

volumes:
  db-data:
    driver: local
  influx-data:
    driver: local
```

### Implantação
1. **Deploy da Stack** (em manager):
   ```bash
   docker stack deploy -c docker-compose.yml swarmpit
   ```

2. **Verificar**:
   ```bash
   docker stack ls
   docker stack ps swarmpit
   docker service ls
   ```

3. **Acesso**:
   - Acesse `http://<manager>:888` (padrão: `admin/swarmpit`).

### Observações
- Focado em Swarm; não suporta standalone ou Kubernetes.
- Usa InfluxDB para métricas e CouchDB para dados.


### 2. Portainer: Painel de Controle Universal

**Descrição:**  
Portainer é uma GUI intuitiva e leve para gerenciar contêineres em Docker Standalone, Swarm, Kubernetes e Podman. Oferece um painel centralizado para implantação e monitoramento.

### Configuração da Stack
1. **Baixar YAML**:
   ```bash
   curl -L https://downloads.portainer.io/portainer-agent-stack.yml -o portainer-agent-stack.yml
   ```

2. **Deploy da Stack** (em manager):
   ```bash
   docker stack deploy -c portainer-agent-stack.yml portainer
   ```

3. **Verificar**:
   ```bash
   docker stack ls
   docker stack ps portainer
   ```

4. **Acesso**:
   - Acesse `http://<manager>:9000` (padrão: `admin/<gerado>`).

### Observações
- Suporta múltiplos orquestradores (Swarm, Kubernetes, standalone).
- Difere do Swarmpit por ser uma solução mais ampla.


### 3. Harbor: Registro Privado

**Descrição:**  
Harbor é um registry privado open-source para armazenar e gerenciar imagens Docker, com recursos como segurança, replicação e UI.

### Configuração
- **Referência Oficial**: [Harbor Docs](https://goharbor.io/docs/2.14.0/)
- **Passos Gerais**:
  1. Baixe o instalador do GitHub.
  2. Configure via `harbor.yml` (ex.: hostname, certificados).
  3. Execute o script de instalação.
- **Exemplo Básico**:
  ```bash
  ./install.sh
  ```
- **Acesso**: `http://<harbor-host>` (padrão: `admin/Harbor12345`).

### Observações
- Ideal para ambientes corporativos (ex.: registry.docker-dca.example no seu rascunho).
- Requer configuração manual; veja a documentação para detalhes.


### 4. Docker Machine: Provisionamento de Hosts

**Descrição:**  
Docker Machine é uma ferramenta de linha de comando para criar e gerenciar máquinas virtuais (VMs) com Docker Engine instalado, útil em setups legados ou remotos.

### Funcionalidades
- Cria VMs com Docker em hypervisores (VirtualBox, Hyper-V).
- Configura variáveis de ambiente para comunicação com o Docker remoto.
- Gerencia hosts remotos (ex.: AWS, Azure).

### Comandos Principais
1. **Listar VMs**:
   ```bash
   docker-machine ls
   ```

2. **Criar VM**:
   ```bash
   docker-machine create --driver virtualbox minha-vm
   ```

3. **Conectar à VM**:
   ```bash
   docker-machine env minha-vm
   eval $(docker-machine env minha-vm)
   docker container ls  # Executa na VM
   ```

4. **Obter IP**:
   ```bash
   docker-machine ip minha-vm
   ```

5. **Gerenciar VM**:
   ```bash
   docker-machine stop minha-vm
   docker-machine start minha-vm
   docker-machine env -u  # Desconectar
   eval $(docker-machine env -u)  # Limpar variáveis
   docker-machine rm -y minha-vm  # Remover
   ```

6. **Inspecionar**:
   ```bash
   docker-machine inspect minha-vm | jq
   ```

### Observações
- Útil antes do Docker Desktop; hoje menos comum, mas válido para VMs remotas.
- Requer hypervisor instalado.

## Boas Práticas
- **Swarmpit/Portainer**: Use em labs; em produção, restrinja acesso via firewall.
- **Harbor**: Configure HTTPS e backups regulares.
- **Docker Machine**: Documente VMs criadas; limpe com `docker-machine rm` após uso.
- **Segurança**: Evite expor portas (ex.: 888, 9000) sem autenticação.
- **Teste**: Valide com `docker stack ps` e monitore com `docker stats`.

---

## Comparação Rápida

| Ferramenta    | Foco Principal         | Suporte Swarm | Suporte Standalone | GUI | Autoscaling |
|----------------|-------------------------|---------------|---------------------|-----|-------------|
| Swarmpit      | Gerenciamento Swarm    | Sim           | Não                | Sim | Não         |
| Portainer     | Gerenciamento Geral    | Sim           | Sim                | Sim | Não         |
| Harbor        | Registro Privado       | Indireto      | Sim                | Sim | Não         |
| Docker Machine| Provisionamento de VMs | Indireto      | Sim                | Não | Não         |

---

## Próximos Passos
- Integre Swarmpit/Portainer com a stack de monitoramento (Capítulo 07).
- Configure Harbor como registry no seu lab (ex.: registry.docker-dca.example).
- Explore Docker Machine para hosts remotos em produção.

**Dica Final:** Adicione capturas de tela das GUIs (Swarmpit/Portainer) ao seu GitHub para documentação visual! 🚀

---

## 🏷️ Labels

## A Importância de Labels para Containers e Suas Aplicações

**Objetivo:** Explorar o uso de labels no Docker para organização, rastreabilidade e integração com ferramentas externas, como Traefik e Portainer, com foco em CI/CD e orquestração.

**Conceito Principal:**  
Labels (rótulos) são pares chave-valor que adicionam metadados a objetos Docker (containers, imagens, serviços, nós). Eles não alteram o comportamento interno, mas são fundamentais para organização, automação e integração com ferramentas de terceiros, como proxies reversos, CI/CD e orquestração no Swarm.

### O que São Labels e Quando Usá-los?

Labels são metadados aplicados a objetos Docker, incluindo containers, imagens, serviços e nós no Swarm. A diferença principal está no propósito e no momento de aplicação:
- **Containers e Imagens**: Focados em identificação, rastreabilidade e integração com ferramentas externas.
- **Serviços e Nós (Swarm)**: Usados para orquestração, como placement constraints e políticas de deploy.

### Como Aplicar Labels
Labels podem ser aplicados em diferentes estágios:

| Objeto           | Onde Aplicar                | Comando/Sintaxe                                  |
|-------------------|-----------------------------|-------------------------------------------------|
| Imagem (Comum)   | No `Dockerfile`             | `LABEL com.minhaempresa.projeto="frontend"`     |
| Container (Runtime) | Durante `docker run`       | `docker run -d --label "ambiente=dev" nginx`    |
| Service (Swarm)  | No `docker-compose.yml` ou `docker service create` | `labels: ["traefik.enable=true"]`              |


### Casos de Uso Comuns para Labels em Containers e Imagens

Labels em containers e imagens são essenciais para organização e integração, enquanto em serviços/nós ajudam na orquestração interna do Swarm.

| Caso de Uso            | Explicação                                      | Exemplo de Label                     |
|-------------------------|-------------------------------------------------|--------------------------------------|
| **Rastreabilidade (Supply Chain)** | Documenta origem e build para auditoria e segurança. | `org.opencontainers.image.revision="git-sha1234"` |
| **Filtragem e Organização** | Categoriza para gestão e comandos em lote.      | `projeto="ecommerce", equipe="devops"` |
| **Integração com Ferramentas Externas** | Configura proxies reversos ou CI/CD automaticamente. | `traefik.enable="true", traefik.http.routers.web.rule="Host(\exemplo.com)"` |


### Exemplo Prático: Integração com Traefik (Proxy Reverso)

O Traefik é um proxy reverso moderno que utiliza labels para configurar roteamento dinâmico. O blog de Caio Delgado (https://caiodomingos.com.br) tem um tutorial antigo, mas os conceitos básicos ainda são válidos (atualizado para Traefik v2.x em 2025).

### Passo a Passo (Atualizado)
1. **Pré-requisitos**:
   - Rede overlay criada: `docker network create web-proxy`.
   - Traefik instalado como serviço Swarm (exemplo abaixo).

2. **Stack Traefik (`traefik.yml`)**:
   ```yaml
   version: '3.9'
   services:
     traefik:
       image: traefik:v2.10  # Versão atualizada (verifique no site oficial)
       command:
         - "--api.insecure=true"  # Habilita dashboard (use HTTPS em prod)
         - "--providers.docker=true"
         - "--providers.docker.swarmmode=true"
         - "--entrypoints.web.address=:80"
         - "--entrypoints.websecure.address=:443"
       ports:
         - "80:80"
         - "443:443"
         - "8080:8080"  # Dashboard
       volumes:
         - /var/run/docker.sock:/var/run/docker.sock:ro
       networks:
         - web-proxy
       deploy:
         placement:
           constraints:
             - node.role == manager
   networks:
     web-proxy:
       driver: overlay
   ```

3. **Deploy Traefik**:
   ```bash
   docker stack deploy -c traefik.yml traefik
   ```

4. **Criar Container com Labels**:
   ```bash
   docker run -d \
     --name meu-webserver \
     --network web-proxy \
     --label "traefik.enable=true" \
     --label "traefik.http.routers.meu-webserver.rule=Host(`myapp.meudominio.com`)" \
     --label "traefik.http.routers.meu-webserver.entrypoints=web" \
     nginx
   ```

5. **Verificar**:
   - Acesse `http://myapp.meudominio.com` (configure DNS ou `/etc/hosts` para apontar ao manager).
   - Veja o dashboard em `http://<manager>:8080`.

### Correções e Atualizações
- **Versão**: O blog de Caio pode usar Traefik v1.x. Atualizei para v2.10 (verifique a versão mais recente em https://traefik.io).
- **Segurança**: Remova `--api.insecure=true` em produção; use certificados TLS.
- **Swarm**: Adicionei `swarmmode=true` para compatibilidade total.


### Alternativa: Integração com Portainer

Portainer é uma alternativa mais versátil que o Traefik para demonstrar labels, pois permite gerenciar serviços Swarm via GUI e usa labels para configuração avançada (ex.: constraints, tags).

### Passo-a-Passo
1. **Deploy Portainer** (usando o YAML do Capítulo 08):
   ```bash
   curl -L https://downloads.portainer.io/portainer-agent-stack.yml -o portainer-agent-stack.yml
   docker stack deploy -c portainer-agent-stack.yml portainer
   ```

2. **Adicionar Label a um Serviço**:
   - Edite o YAML ou crie um serviço:
     ```bash
     docker service create --name app \
       --label "portainer.tags=frontend" \
       --network portainer_agent_network \
       --replicas 2 \
       nginx
     ```

3. **Gerenciar via Portainer**:
   - Acesse `http://<manager>:9000`.
   - Filtre serviços por label `frontend` na GUI.

### Por que Portainer Pode Ser Melhor?
- **Versatilidade**: Suporta Swarm, standalone e Kubernetes, ao contrário do Traefik (focado em proxy).
- **Simplicidade**: Labels como `portainer.tags` são intuitivos para organização e filtragem.
- **Contexto DCA**: Alinha-se com seu uso de GUIs (Swarmpit, Portainer) e monitoramento.


### Aplicações em CI/CD

Labels são cruciais em pipelines CI/CD (ex.: Jenkins, GitHub Actions) para:
- **Identificação**: `build.number=123` rastreia builds.
- **Deploy Automático**: Labels como `deploy.env=prod` acionam deploys condicionais.
- **Integração**: Ferramentas como Traefik ou Portainer leem labels para configurar roteamento ou gerenciamento.

### Exemplo CI/CD (GitHub Actions)
```yaml
name: Build and Deploy
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: |
          docker build -t myapp:${{ github.sha }} --label "build.number=${{ github.run_number }}" .
          docker push myapp:${{ github.sha }}
      - name: Deploy to Swarm
        run: |
          ssh user@manager "docker service update --label-add 'deploy.env=prod' myapp"
```

### Boas Práticas
- **Consistência**: Use prefixos (ex.: `com.minhaempresa.`) para evitar conflitos.
- **Documentação**: Registre labels em um README ou wiki.
- **Segurança**: Evite expor labels sensíveis (ex.: credenciais).
- **Teste**: Valide com `docker inspect --format '{{.Config.Labels}}' <container>`.


### Observações
- **Swarm vs. Containers**: Labels em serviços/nós controlam orquestração (ex.: `node.labels.location`), enquanto em containers/imagens focam em metadados e integração.
- **Traefik vs. Portainer**: Traefik é ideal para roteamento dinâmico; Portainer para gestão geral.
- **Contexto DCA**: Alinha-se com seu uso de stacks e redes overlay.

---

### Próximos Passos
- Experimente labels com Harbor para rastrear imagens.
- Integre com CI/CD para automação.
- Compare Traefik e Portainer em um lab.

**Dica Final:** Adicione exemplos de saída (`docker inspect`) ao seu GitHub! 🚀

---

## ☸️ Kubernetes (K8s)

**Objetivo:** Introdução ao Kubernetes via Minikube (K8s in Docker). Foco em conceitos básicos: **pods (unidade mínima)**, redes, deployments, secrets/configmaps e persistência (PV/PVC) para o DCA (Docker Certified Associate).

**Pré-requisitos:** Docker instalado. Teste em lab (ex: VM).  
**Dica Geral:** Sempre use `kubectl apply` para criar/atualizar. `kubectl get all` para overview.  
**Conceito Chave:** Docker → **Container** (mínimo deploy). K8s → **Pod** (1+ containers).  

![Conceito de Pod](https://kubernetes.io/docs/tutorials/kubernetes-basics/public/images/module_03_pods.svg)  

**Referência Inicial:** [Minikube Docs](https://minikube.sigs.k8s.io/docs/start/?arch=%2Flinux%2Fx86-64%2Fstable%2Fbinary+download).

---

## 🚀 Instalação e Início com Minikube

### Instalar Minikube
```bash
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
minikube version  # Verifique
```

### Iniciar Cluster (Driver Docker)
```bash
minikube start --driver=docker
docker container ls  # Confirma container Minikube
```

### Kubectl Básico
> **Sintaxe:** `kubectl + verbo + recurso + opções`.  
> Verbos: `get`, `describe`, `create`, `apply`, `delete`, `patch`.  
> Recursos: `nodes`, `pods`, `services`, `deployments`, `namespaces`.  

```bash
kubectl get nodes  # Nós do cluster
kubectl get all  # Tudo (pods, services, etc.)
```

**Finalizar Lab:** `minikube delete` (limpa tudo).

---

## 🛡️ Pods (Simples e Multi-Containers)

### YAML Essencial (Cláusulas Obrigatórias)
| Cláusula | Função | Exemplo (Pod) |
|----------|--------|---------------|
| `apiVersion` | Versão da API | `v1` |
| `kind` | Tipo de objeto | `Pod` |
| `metadata` | Identificação (nome, labels) | `name: meu-pod` |
| `spec` | Estado desejado (containers, volumes) | `containers: [...]` |

### Exemplo 1: Pod Simples (`pod.yml`)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```
Equivalente Docker: `docker run -d --name nginx -p 80:80 nginx:1.14.2`.

```bash
kubectl apply -f pod.yml
kubectl get pods  # Status
kubectl logs nginx  # Logs
kubectl logs -f nginx  # Follow
kubectl delete -f pod.yml  # Ou `pod/nginx`
```

### Exemplo 2: Pod com Comando (`demo.yml`)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo
spec:
  containers:
  - name: testpod
    image: alpine:3.5
    command: ["ping", "8.8.8.8"]
```
Equivalente Docker: `docker run -dit --name demo alpine:3.5 ping 8.8.8.8`.

### Exemplo 3: Pod Multi-Containers (`multi-container.yml`)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  restartPolicy: Never
  volumes:
  - name: shared-data
    emptyDir: {}
  containers:
  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html && sleep 3600"]
```
> **OBS:** `command` sobrescreve `ENTRYPOINT` (Docker). `args` sobrescreve `CMD`.  
> Conceito: **Sidecar** = Container auxiliar (ex: debian aqui).

| Cenário | O que Executa | Uso |
|---------|---------------|-----|
| Sem `command/args` | ENTRYPOINT + CMD (Dockerfile) | Padrão |
| `command` só | `command` + CMD (Dockerfile) | Sobrescreve entrypoint |
| `args` só | ENTRYPOINT (Dockerfile) + `args` | Sobrescreve args |
| Ambos | `command` + `args` | Sobrescreve tudo |

```bash
kubectl apply -f multi-container.yml
kubectl get pods/all
kubectl describe pod multi-container  # Eventos
kubectl get pod multi-container -o yaml  # YAML completo
kubectl logs multi-container -c nginx-container  # Especifica container
kubectl exec -it multi-container -c nginx-container -- /bin/bash  # Entre
kubectl delete -f multi-container.yml
```

![Pod Single vs Pod Multi - Containers](https://devopscube.com/content/images/2025/03/kubernetes-pod-1.png) 


📝 **Alternativa Entrar Minikube:** `minikube ssh` (bash do host K8s).

---

## 🌐 Redes (Services: ClusterIP e NodePort)

**Conceitos:**
- **ClusterIP:** IP interno ao cluster (default). Acesso só dentro.
- **NodePort:** Expõe em todos nodes (porta 30000-32767). Acesso externo.

### Exemplo 4: Pod + Service ClusterIP (`nginx-pod.yml`)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: hello-world
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

`nginx-svc.yml` (ClusterIP implícito):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-dca
spec:
  selector:
    app: hello-world
  ports:
  - port: 80
    protocol: TCP
```

```bash
kubectl apply -f nginx-pod.yml -f nginx-svc.yml
kubectl get pods/all/services
kubectl describe service nginx-dca
# Teste interno (via container Minikube)
docker exec -it minikube bash
curl <IP_Service>:80  # Sucesso
kubectl delete -f nginx-svc.yml
```

### Exemplo 5: NodePort (`nginx-svc-nodeport.yml`)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-dca
spec:
  type: NodePort
  selector:
    app: hello-world
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 80
    nodePort: 30033  # Aleatório se omitido
```

```bash
kubectl apply -f nginx-svc-nodeport.yml
kubectl get services
kubectl describe service nginx-dca
# Teste interno (pod Alpine)
kubectl run --rm -it alpine --image=alpine --restart=Never -- ash
apk update && apk add curl  # Dentro
curl nginx-dca:8080  # Sucesso
exit
# Teste externo
minikube ip  # IP Minikube
curl 192.168.XX.XX:30033  # Sucesso
curl $(minikube ip):30033
minikube service --url nginx-dca  # URL completa
curl $(minikube service --url nginx-dca)
kubectl delete service/nginx-dca pod/nginx
```

---

## 💡 Observação: Diferença entre Portas em Kubernetes vs. Docker

Ao migrar do Docker para o Kubernetes, a forma como as portas são expostas é a principal fonte de confusão. Lembre-se que o Kubernetes é **agnóstico** (não se importa) com o `EXPOSE` do Dockerfile.

| Porta/Campo | Onde é Definida | Finalidade e Escopo |
| :--- | :--- | :--- |
| **Porta da Aplicação** | Código Fonte (`app.py`, `server.js`) | É a porta real que o processo da aplicação está ouvindo dentro do Container (ex: 3000, 8080). |
| **`targetPort`** | Service YAML | **CHAVE K8S:** É a porta que o Service aponta no **Container**. **DEVE ser igual** à Porta da Aplicação. |
| **`EXPOSE`** | Dockerfile | **Docker:** É apenas **documentação/metadado**. Não afeta a conectividade no Kubernetes. |
| **`port`** | Service YAML | **Porta do Service (LAN/Interna):** É o endereço virtual que o Service expõe. Outros Pods se conectam usando este número (ex: `meu-service:80`). |
| **`nodePort`** | Service YAML (Tipo NodePort) | **Porta de Acesso Externo (WAN/Node):** A porta estática aberta em **todos os Nodes** do cluster (intervalo padrão 30000-32767). É a forma mais básica de acesso externo. |

### Fluxo de Tráfego Completo (NodePort)

O caminho do tráfego demonstra a hierarquia:


1.  **Acesso Externo (WAN/Internet)**:
    * **Porta:** `nodePort` (30000-32767)
    * **Caminho:** `IP do Node : NodePort`

2.  **Endereço Interno (LAN/Service)**:
    * **Porta:** `port` (A porta que o Service expõe internamente, ex: 80)
    * **Caminho:** `ClusterIP : Port`

3.  **Destino Final (Aplicação)**:
    * **Porta:** `targetPort` (A porta da aplicação real no Container, ex: 8080)
    * **Caminho:** `Container : TargetPort`

**Caminho Completo (Markdown Simples):**
`WAN` ➔ `[IP do Node : NodePort]` ➔ `[ClusterIP : Port]` ➔ `[Container : TargetPort]`


### 🌐 Comparativo de Controladores: Kubernetes vs. Docker Swarm

| Objeto Kubernetes | Função Principal (K8s) | Equivalente Docker Swarm | Observações Chave (Analogia) |
| :--- | :--- | :--- | :--- |
| **Deployment** | Gerencia réplicas e o ciclo de vida (rollouts) de Pods **Stateless** (sem estado). | **Service (Modo Replicated)** | É o controlador mais comum para **escalabilidade** de aplicações web e microsserviços. |
| **ReplicaSet** | Garante o número exato (`N`) de Pods rodando. Geralmente é gerenciado pelo Deployment. | **Service (Modo Replicated)** | O ReplicaSet é a "ferramenta de contagem" que o Deployment usa para manter o número de réplicas. |
| **DaemonSet** | Garante que **exatamente um Pod** seja executado em **todos os Nodes** elegíveis do cluster. | **Service (Modo Global)** | Usado para agentes de logs, monitoramento e segurança. O Pod não tem nome persistente. |
| **StatefulSet** | Gerencia réplicas com **identidade estável** (nome de rede e volume/armazenamento persistente) e ordem garantida. | **NÃO há Equivalente Direto** | Usado para bancos de dados e sistemas distribuídos. É o controlador para aplicações **Stateful** (com estado). |


---

## 📦 Deployments (Escala e Resiliência)

### Exemplo 6: Deployment Nginx (`nginx-deploy.yml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx-dca
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-dca
  template:
    metadata:
      labels:
        app: nginx-dca
    spec:
      containers:
      - name: nginx-dca
        image: nginx
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f nginx-deploy.yml
kubectl get deployments/all
kubectl describe deployment nginx-deployment
# Teste resiliência (2 terminais)
watch kubectl get all -l app=nginx-dca  # Terminal 1
kubectl delete pod -l app=nginx-dca  # Terminal 2 (recria!)
kubectl delete -f nginx-deploy.yml
```

Com certeza! Essa regra é fundamental e merece ser destacada.

Aqui está um bloco de resumo e boas práticas em Markdown sobre a relação entre `Deployment`, `Spec`, `Selector` e `Template`:

---

## 📌 Regra de Ouro: Deployment como Unidade Lógica

A melhor prática no Kubernetes é tratar o `Deployment` e seus componentes internos como uma **unidade indivisível** que representa um único componente (ou Microsserviço) da sua aplicação.

### 1. Um Deployment para Cada Componente Lógico

| Objeto | Regra Prática | Consequência |
| :--- | :--- | :--- |
| **Deployment YAML** | Deve ser criado **um arquivo de Deployment** para **cada unidade funcional** da sua aplicação (ex: um para o Frontend, um para o Serviço de Usuários, etc.). | Garante o isolamento do **rollout** (a atualização de um serviço não afeta o outro). |

### 2. A Tríade Essencial (`Selector` ↔ `Template`)

A relação entre o seletor e o template é o **vínculo vital** para que o Deployment funcione:

* **`spec.selector.matchLabels`**: O **critério de busca** do Deployment (O QUE ele procura gerenciar).
* **`template.metadata.labels`**: O **crachá de identidade** do Pod (O QUE ele é).

> ✅ **Regra:** O `selector` **DEVE** corresponder ao `template.labels`. Se não corresponderem, o Deployment não consegue gerenciar os Pods que ele mesmo cria.

### 3. O `Template` é o Contrato

O bloco `template` é o contrato que define o Pod. Qualquer alteração neste bloco (imagem, variáveis de ambiente, volumes) é interpretada pelo Deployment como uma necessidade de **rollout** (atualização gradual) para substituir os Pods antigos pelos novos.

| Campo | Função | Ação do Deployment |
| :--- | :--- | :--- |
| `spec.replicas` | Controla o número de Pods. | Causa **Escala** (aumentar/diminuir). |
| `spec.template` | Controla o conteúdo do Pod. | Causa **Rollout** (substituição gradual). |

---

Este resumo deve ser um ótimo ponto de referência para a sua seção de redes e arquitetura K8s!

---

Com certeza! O conceito de **Rollout** é fundamental para entender a alta disponibilidade no Kubernetes.

Aqui está um resumo focado no Gerenciamento de Rollout, perfeito para a sua documentação:

---

## 🚀 Gerenciamento de Rollout: Atualização sem Downtime

O **Rollout** é o processo de atualização de uma aplicação em execução para uma nova versão, gerenciado pelo **Deployment**. O objetivo é realizar a transição com a máxima disponibilidade e segurança.

### 1. O Gatilho do Rollout

Um rollout é acionado quando há uma **alteração funcional** no bloco **`spec.template`** de um Deployment. O Kubernetes percebe que o "blueprint" (o modelo do Pod) mudou e inicia a substituição.

| Gatilho do Rollout | Ação |
| :--- | :--- |
| **Imagem do Container** | Mudar de `app:v1` para `app:v2`. |
| **Variáveis de Ambiente** | Adicionar, remover ou alterar uma variável. |
| **Volumes/ConfigMaps** | Alterar a montagem de um volume ou ConfigMap/Secret referenciado. |

> ⚠️ **Atenção:** Mudar apenas o campo `replicas` causa uma **escala**, mas **NÃO** um rollout.

### 2. Estratégia Padrão: RollingUpdate

A estratégia padrão e mais segura é o `RollingUpdate` (Atualização Contínua), que garante que o serviço permaneça acessível durante a transição.

| Parâmetro | Descrição | Exemplo Padrão |
| :--- | :--- | :--- |
| **`maxUnavailable`** | Número (ou porcentagem) máximo de Pods que podem estar **indisponíveis** (desligados ou não prontos) durante a atualização. | `25%` |
| **`maxSurge`** | Número (ou porcentagem) máximo de Pods **além** da contagem desejada que o Kubernetes pode criar para acomodar os novos. | `25%` |

> **Exemplo:** Em um Deployment com 4 réplicas, o `RollingUpdate` fará:
> 1. Cria 1 Pod novo (`maxSurge` 25% → 1 Pod). Total: 5 Pods.
> 2. Derruba 1 Pod antigo (`maxUnavailable` 25% → 1 Pod). Total: 4 Pods.
> 3. Repete até que todos os Pods sejam da nova versão.

### 3. Gerenciamento e Controle

Os comandos `kubectl rollout` são essenciais para monitorar e intervir no processo:

| Comando | Função | Exemplo |
| :--- | :--- | :--- |
| `kubectl rollout status` | Acompanha o progresso da atualização em tempo real. | `kubectl rollout status deployment/meu-app` |
| `kubectl rollout history` | Lista todas as revisões (versões) do Deployment, incluindo o que mudou em cada uma. | `kubectl rollout history deployment/meu-app` |
| `kubectl rollout undo` | **Reverte instantaneamente** a aplicação para a versão anterior (o rollback). | `kubectl rollout undo deployment/meu-app` |
| `kubectl rollout pause` | Interrompe um rollout em andamento para inspeção. | `kubectl rollout pause deployment/meu-app` |

---
## 🔑 Secrets e ConfigMaps

**Conceitos:**
- **ConfigMap:** Dados não-sensíveis (configs, envs).
- **Secret:** Dados sensíveis (base64, senhas). Uso: Env ou volumes.

### Exemplo 7: ConfigMap (`configmap.yml`)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap-app1
data:
  initial_refresh_value: "4"
  ui_properties_file_name: "user-interface.properties"
  user-interface.properties: |
    color.good=green
    color.bad=red
```

`pod-configmap.yml` (consumir):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app1
spec:
  containers:
  - name: app1
    image: alpine
    command: ["ping", "8.8.8.8"]
    volumeMounts:
    - name: configs
      mountPath: "/etc/configs"
      readOnly: true
  volumes:
    - name: configs
      configMap:
        name: configmap-app1
```

```bash
kubectl apply -f configmap.yml -f pod-configmap.yml
kubectl get configmap/pods
kubectl describe configmap configmap-app1
kubectl exec -it app1 -- ash
ls /etc/configs; cat /etc/configs/initial_refresh_value  # Dentro
kubectl delete pod/app1
```

### Exemplo 8: Secret (`secret.yml`)
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: senha-mysql
type: kubernetes.io/basic-auth
stringData:
  username: root
  password: 123mudar
```

`pod-secret.yml` (consumir):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-db
spec:
  containers:
  - name: mysql-db
    image: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: senha-mysql
          key: password
```

```bash
kubectl apply -f secret.yml -f pod-secret.yml
kubectl get secrets/pods
kubectl describe secret senha-mysql pod/mysql-db
kubectl exec -it mysql-db -- mysql -u root -p123mudar  # Teste
kubectl delete pod/mysql-db
```

---

## 💽 Persistência (PV e PVC)

**Conceitos:**
- **PV (PersistentVolume):** Disco virtual ligado ao storage físico.
- **PVC (PersistentVolumeClaim):** Requisição de storage (liga ao PV menor que atende).
- **Modos:** RWO (1 node RW), ROX (muitos RO), RWX (muitos RW), RWOP (1 pod RW).  
> **OBS:** PVC escolhe PV mínimo satisfatório.

![PV vs PVC](https://support.huaweicloud.com/intl/en-us/basics-cce/en-us_image_0261235726.png) 

![POD](https://www.cloudzero.com/wp-content/uploads/2023/10/kubernetes-nodes.webp) 

### Exemplo 9: PVs (`pv.yml`)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv10m
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Mi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/dados1"
---
# pv200m e pv1g semelhantes...
```

```bash
kubectl apply -f pv.yml
kubectl get pv
```

### Exemplo 10: PVCs (`pvc.yml`)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc100m
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
---
# pvc700m semelhante...
```

```bash
kubectl apply -f pvc.yml
kubectl get pvc  # Bound: pvc100m → pv200m, pvc700m → pv1g
```

### Exemplo 11: Pod com PVC (`webserver.yml`)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webserver
spec:
  volumes:
    - name: webdata
      persistentVolumeClaim: 
        claimName: pvc100m
  containers:
    - name: webserver
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: webdata
```

> Fluxo: Pod → PVC → PV.

```bash
kubectl apply -f webserver.yml
kubectl get pods
kubectl exec -it webserver -- bash  # curl localhost
kubectl delete pod/webserver
```

---

## 🛑 Finalizando e Próximos Passos

```bash
minikube delete  # Limpa cluster
```

**OBS:** Para enterprise: Docker EE (legado) ou MKE (Mirantis Kubernetes Engine, moderno). Estude Helm para charts reutilizáveis.  
**Próximos:** HPA (autoscaling), Ingress (exposição externa), RBAC (segurança).  

**Capítulo K8s: Completo com foco em basics. Teste iterativamente!**


## ☸️ Mirantis Kubernetes Engine (MKE) - O Legado Docker EE


### 🛠️ Requisitos de Hardware para Docker Swarm

A arquitetura do Docker Swarm exige que os nós Manager e Worker tenham especificações distintas para garantir a estabilidade do plano de controle e a performance da execução das tarefas.

* **Requisitos Mínimos (Testes e Desenvolvimento):**
    * **Manager Node:** Mínimo de **8GB de RAM** e **2 vCPUs**. Esta configuração é crucial, pois o Manager lida com o plano de controle, o consenso Raft e a orquestração.
    * **Worker Nodes:** Mínimo de **4GB de RAM** e **2 vCPUs** (para cargas de trabalho leves).
    * **Disco:** Pelo menos **25 GB** de espaço em disco no Manager para armazenamento de dados do estado do Swarm (Raft log).

* **Requisitos de Produção (Recomendados):**
    * Para ambientes de produção, a escalabilidade e a redundância são prioridades.
    * **Manager Node:** Recomenda-se um aumento para **16GB de RAM** e **4 vCPUs** para lidar com picos de orquestração e maior tráfego do plano de controle.
    * **Disco:** Utilize **SSD** (Solid State Drive) com espaço entre **25 GB a 100 GB** para garantir baixa latência na leitura e escrita do estado do cluster. O desempenho do disco no Manager é vital para a saúde do Swarm.

### Precisa terminar ...