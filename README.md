**Passo a Passo: Deploy Automatizado**

---

### 1. Preparar o Projeto Quarkus

- Buildar localmente:

```bash
./mvnw package -DskipTests
```

- Gerar imagem com Docker (opcional para testes locais):

```bash
docker build -f src/main/docker/Dockerfile.jvm -t ghcr.io/USUARIO/quarkus-example:v1 .
```

---

### 2. Configurar Docker Compose

**docker-compose.yml**:

```yaml
version: '3.8'

services:
  app:
    image: ghcr.io/USUARIO/quarkus-example:${TAG}
    ports:
      - "8080:8080"
    environment:
      - QUARKUS_DATASOURCE_JDBC_URL=${QUARKUS_DATASOURCE_JDBC_URL}
      - QUARKUS_DATASOURCE_USERNAME=${QUARKUS_DATASOURCE_USERNAME}
      - QUARKUS_DATASOURCE_PASSWORD=${QUARKUS_DATASOURCE_PASSWORD}
```

**.env**:

```env
TAG=v1
QUARKUS_DATASOURCE_JDBC_URL=jdbc:postgresql://seu-host:5432/seubanco
QUARKUS_DATASOURCE_USERNAME=seuusuario
QUARKUS_DATASOURCE_PASSWORD=suasenha
```

---

### 3. Configurar o Servidor

- Instalar o Docker:

```bash
curl -fsSL https://get.docker.com | sh
```

- Instalar o plugin Docker Compose (compose v2):

```bash
apt install docker-compose-plugin -y
```

- Criar o diretório da aplicação:

```bash
mkdir -p /opt/quarkus-example
```

- Enviar `docker-compose.yml` e `.env` para o servidor:

```bash
scp -i ~/.ssh/id_github_deploy docker-compose.yml root@SEU_IP:/opt/quarkus-example/
scp -i ~/.ssh/id_github_deploy .env root@SEU_IP:/opt/quarkus-example/
```

---

### 4. Gerar chave SSH sem senha (localmente)

```bash
ssh-keygen -t ed25519 -C "github-deploy" -f ~/.ssh/id_github_deploy
```

- Adicionar a chave **pública** ao arquivo `~/.ssh/authorized_keys` do servidor on-premise
- Adicionar a chave **privada** como `SERVER_SSH_KEY` no repositório GitHub

---

### 5. Configurar o GitHub Container Registry (GHCR) e Secrets

- Gerar um **Personal Access Token (PAT)** em [https://github.com/settings/tokens](https://github.com/settings/tokens)
  - Escopos: `write:packages`, `read:packages`, e `repo` (se o repo for privado)
- Criar os seguintes **secrets** no repositório GitHub (Settings > Secrets and variables > Actions):

| Nome              | Descrição                                       |
|-------------------|---------------------------------------------------|
| `GHCR_PAT`        | Token pessoal com permissão de publicar imagens   |
| `SERVER_IP`       | IP do servidor                      |
| `SERVER_USER`     | Usuário SSH (ex: `root`)                          |
| `SERVER_SSH_KEY`  | Chave privada SSH (sem senha)                     |

---

### 6. Criar Workflow no GitHub Actions

**.github/workflows/deploy.yml**:

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    env:
      IMAGE_NAME: ghcr.io/USUARIO/quarkus-example
      TAG: v${{ github.run_number }}

    steps:
      - uses: actions/checkout@v3

      - uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Make mvnw executable
        run: chmod +x ./mvnw

      - name: Build Quarkus
        run: ./mvnw package -DskipTests

      - name: Login GHCR
        run: echo "${{ secrets.GHCR_PAT }}" | docker login ghcr.io -u USUARIO --password-stdin

      - name: Build Docker image
        run: docker build -f src/main/docker/Dockerfile.jvm -t $IMAGE_NAME:${TAG} .

      - name: Push image
        run: docker push $IMAGE_NAME:${TAG}

      - name: SSH and Deploy
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /opt/quarkus-example
            sed -i 's/^TAG=.*/TAG=${{ env.TAG }}/' .env
            docker compose pull
            docker compose up -d
```
