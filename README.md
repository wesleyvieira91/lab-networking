# LAB 4 — Networking no Docker (bridge, port mapping e DNS interno)

> **Objetivo do LAB:** fixar o básico do dia a dia:
1) diferença entre **porta pública** (host) e **porta interna** (container)  
2) comunicação entre containers em rede **bridge**  
3) **DNS interno** (resolver nome → IP dentro da rede do Docker)

---

## 0) Conceito rápido (para fixar)

- `-p 8080:80` = **host:8080 → container:80**  
  **Analogia:** “ramal público 8080 chama o ramal interno 80”.

- `docker network create ...` cria um “condomínio” (rede).  
  Containers na mesma rede se enxergam.

- DNS interno: em redes do Docker, **o nome do container** pode resolver para o IP dele.

---

## 1) Pré-requisitos

```bash
docker ps
docker network ls
```

---

## 2) Checkpoint A — Port mapping (host → container)

Suba um Nginx e publique a porta:

```bash
docker run -d --name lab-net-web -p 8080:80 nginx:alpine
```

Valide:

```bash
curl -I http://localhost:8080 | head
docker ps | grep lab-net-web
```

**Sucesso:** `curl -I` retorna `HTTP/1.1 200 OK` (ou similar).

**Evidência (print):** `curl -I http://localhost:8080 | head`

---

## 3) Checkpoint B — Rede bridge custom

Crie uma rede:

```bash
docker network create lab-net
docker network ls | grep lab-net
```

Conecte o nginx nessa rede:

```bash
docker network connect lab-net lab-net-web
```

Confirme:

```bash
docker network inspect lab-net | grep -n lab-net-web || true
```

**Sucesso:** o container aparece na inspeção da rede.

**Evidência (print):** trecho de `docker network inspect lab-net` mostrando `lab-net-web`.

---

## 4) Checkpoint C — Comunicação container → container (DNS interno)

Suba um Alpine dentro da rede e teste `curl` usando o nome do container:

```bash
docker run --rm -it --network lab-net alpine:3.20 sh
```

Dentro do Alpine:

```sh
apk add --no-cache curl
curl -I http://lab-net-web | head
exit
```

**Sucesso:** responde `200 OK`.

**Evidência (print):** saída do `curl -I http://lab-net-web | head`

---

## 5) Checkpoint D — Interno vs Público (explicação para sala)

Agora você tem:
- **Público (host):** `http://localhost:8080`
- **Interno (rede Docker):** `http://lab-net-web` (só dentro da rede)

---

## 6) Extra (rápido) — Sem publicar porta e ainda assim funcionar “interno”

Suba um nginx **sem** `-p` (apenas interno):

```bash
docker rm -f lab-net-web
docker run -d --name lab-net-web --network lab-net nginx:alpine
```

Teste do host (deve falhar):

```bash
curl -I http://localhost:8080 || echo "OK: sem port mapping não acessa pelo host"
```

Teste por dentro da rede (deve funcionar):

```bash
docker run --rm -it --network lab-net alpine:3.20 sh -c "apk add --no-cache curl >/dev/null && curl -I http://lab-net-web | head"
```

---

## 7) Limpeza

```bash
docker rm -f lab-net-web 2>/dev/null || true
docker network rm lab-net 2>/dev/null || true
```

---

## Troubleshooting rápido

### 1) `curl` não existe no Alpine
Dentro do Alpine:
```sh
apk add --no-cache curl
```

### 2) DNS não resolve
Garanta que:
- ambos estão na mesma rede (`--network lab-net`)
- o nome é `lab-net-web`

### 3) Porta já em uso
Troque `8080` por `8081`:
```bash
docker run -d --name lab-net-web -p 8081:80 nginx:alpine
curl -I http://localhost:8081 | head
```
