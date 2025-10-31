# Docker: `ENTRYPOINT` vs `CMD` — Guia Completo com Exemplos

> Uma explicação detalhada sobre as diferenças entre `ENTRYPOINT` e `CMD` no `Dockerfile`, com casos de uso reais, boas práticas e exemplos práticos.

---

## Sumário

- [1. Definições básicas](#1-definições-básicas)
- [2. Diferenças principais](#2-diferenças-principais)
- [3. Formatos de sintaxe](#3-formatos-de-sintaxe)
- [4. Casos de uso e exemplos](#4-casos-de-uso-e-exemplos)
- [5. Boas práticas](#5-boas-práticas)
- [6. Resumo rápido](#6-resumo-rápido)

---

## 1. Definições básicas

| Comando       | Descrição |
|---------------|---------|
| **`CMD`**     | Define o **comando padrão** a ser executado quando o contêiner inicia. Pode ser sobrescrito facilmente com `docker run <imagem> novo-comando`. |
| **`ENTRYPOINT`** | Define o **comando principal** que **sempre será executado**. Argumentos do `docker run` são passados como parâmetros. |

> **Ambos** podem usar dois formatos: **exec** (recomendado) ou **shell**.

---

## 2. Diferenças principais

| Característica         | `CMD`                                   | `ENTRYPOINT`                                   |
|------------------------|-----------------------------------------|------------------------------------------------|
| **Propósito**          | Comandos/argumentos padrão              | Comportamento fixo do contêiner                |
| **Sobrescrita**        | Fácil (`docker run imagem echo teste`)  | Só com `--entrypoint`                          |
| **Argumentos no `docker run`** | Substituem o `CMD`                   | São **anexados** ao `ENTRYPOINT`               |
| **Formato preferido**  | `["arg1", "arg2"]`                      | `["comando", "arg"]`                           |

---

## 3. Formatos de sintaxe

### Formato **exec** (recomendado)
```dockerfile
CMD ["nginx", "-g", "daemon off;"]
ENTRYPOINT ["python", "app.py"]
```
- Executa diretamente → melhor para sinais (`SIGTERM`, `SIGINT`).
- Processo principal = o comando.

### Formato **shell**
```dockerfile
CMD nginx -g "daemon off;"
ENTRYPOINT python app.py
```
- Executa via `/bin/sh -c` → shell é o PID 1.
- **Evite** em processos de longa execução.

---

## 4. Casos de uso e exemplos

### Caso 1: `CMD` sozinho (flexível)

```dockerfile
FROM nginx:alpine
CMD ["nginx", "-g", "daemon off;"]
```

```bash
docker run minha-nginx                # → nginx -g "daemon off;"
docker run minha-nginx echo "oi"      # → echo "oi" (CMD sobrescrito)
```

> **Uso**: Imagens genéricas, ferramentas CLI, testes.

---

### Caso 2: `ENTRYPOINT` sozinho (fixo)

```dockerfile
FROM python:3.9-slim
COPY script.py /script.py
ENTRYPOINT ["python", "/script.py"]
```

```bash
docker run meu-script --help          # → python /script.py --help
docker run meu-script                 # → python /script.py
```

> **Uso**: Aplicações com comando principal imutável.

---

### Caso 3: `ENTRYPOINT` + `CMD` (ideal)

```dockerfile
FROM node:18
COPY app.js /app.js
ENTRYPOINT ["node", "/app.js"]
CMD ["--port", "3000"]
```

```bash
docker run app-node                   # → node /app.js --port 3000
docker run app-node --port 8080       # → node /app.js --port 8080
```

> **Uso**: Aplicações com configuração padrão, mas personalizável.

---

### Caso 4: `ENTRYPOINT` com script de inicialização

```dockerfile
FROM ubuntu:22.04
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["start"]
```

**`entrypoint.sh`:**
```bash
#!/bin/bash
echo "Configurando ambiente..."
# Suas configurações aqui
exec "$@"
```

```bash
docker run minha-img test             # → entrypoint.sh test
```

> **Uso**: Configuração dinâmica, variáveis de ambiente, migrações.

---

### Caso 5: `CMD` shell vs exec

| Tipo | Dockerfile | Problema |
|------|------------|--------|
| **Shell** | `CMD echo oi && sleep 3600` | Shell é PID 1 → não recebe `SIGTERM` |
| **Exec**  | `CMD ["sleep", "3600"]`     | `sleep` é PID 1 → para com `CTRL+C` |

> **Sempre use `exec` para processos principais.**

---

## 5. Boas práticas

| Regra | Por quê? |
|------|--------|
| Use **formato exec** | Evita shell desnecessário, melhora sinais |
| Use `ENTRYPOINT` + `CMD` | Controle + flexibilidade |
| Use `ENTRYPOINT` para apps | Garante execução correta |
| Use `CMD` para ferramentas | Permite sobrescrita |
| Scripts? Use `exec "$@"` | Passa argumentos corretamente |

---

## 6. Resumo rápido

| Você quer... | Use |
|-------------|-----|
| Comando fixo | `ENTRYPOINT` |
| Argumentos padrão | `CMD` |
| Ambos (recomendado) | `ENTRYPOINT` + `CMD` |
| Flexibilidade total | Só `CMD` |
| Configuração antes | `ENTRYPOINT` com script |

---

## Exemplos no GitHub

- [nginx-cmd](https://github.com/exemplos/docker-nginx-cmd)
- [python-entrypoint](https://github.com/exemplos/python-app-entrypoint)
- [entrypoint-script](https://github.com/exemplos/docker-entrypoint-sh)

---

## Referências

- [Dockerfile reference - CMD](https://docs.docker.com/reference/dockerfile/#cmd)
- [Dockerfile reference - ENTRYPOINT](https://docs.docker.com/reference/dockerfile/#entrypoint)
- [Best practices for ENTRYPOINT/CMD](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

---

> **Dica final**: Pense no `ENTRYPOINT` como o **"verbo principal"** do seu contêiner e no `CMD` como os **"argumentos padrão"**.

---
