# Documentação do Projeto Unidade 1 Docker

## Pré-requisitos

- 🎮 **Docker**: [Instalação do Docker](https://docs.docker.com/engine/install/)
- ⚙️ **Docker Compose**: [Instalação do Docker Compose](https://docs.docker.com/compose/install/)
- 🛠️ **Git**: [Instalação do Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

---

## **Git**:
Clone

  ```bash
  git clone https://github.com/fellipedemoraes/puc-unidade-1-docker.git
  ```
  
Navegue até a pasta correspondente:

  ```bash
  cd puc-unidade-1-docker
  ```
Execute:

```bash
docker-compose up
```

Você pode acessar a aplicação diretamente do seu computador utilizando o endereço do localhost na porta 80: http://localhost.

<img src="print-ok.png">

### Observação:
 
A porta do host 80 deve estar disponível. Caso haja um servidor web (como Apache ou Nginx) em execução localmente, pode ocorrer um conflito de porta. 
Nesse caso, modifique o arquivo docker-compose.yml localizado na raiz do projeto, no bloco de configuração do Nginx, e mapeie uma porta diferente.

---

## Estrutura do Repositório

- **Arquivo Docker Compose:** Localizado na raiz do projeto. Nome: `docker-compose.yml`
- **Dockerfile do Backend (Python):** Localizado na raiz do projeto. Nome: `Dockerfile`
- **Dockerfile do Frontend (React):** Localizado na pasta `frontend`. Nome: `frontend/Dockerfile`
- **Configuração do NGINX:** Localizado na raiz do projeto. Nome: `nginx.conf`

---

## Configuração e Estrutura

### 1. Arquivo `docker-compose.yml`

O arquivo `docker-compose.yml` Contém todos os serviços necessários para iniciar o sistema.

#### 1.1. Frontend

Foi criado um **Dockerfile** para o frontend dentro da pasta `frontend`, separado do backend para facilitar atualizações. No arquivo `docker-compose.yml`, o serviço frontend é configurado para realizar o build utilizando o contexto da pasta `frontend`:

```yaml
frontend:
  build:
    context: ./frontend
    dockerfile: Dockerfile
  restart: always
  environment:
    - REACT_APP_BACKEND_URL=http://localhost
  depends_on:
    - guess
```

#### 1.2. Backend (Serviço Guess)

O backend é representado pelo bloco `guess`, que contém um `Dockerfile` localizado na raiz do projeto. As variáveis de ambiente, que anteriormente estavam codificadas diretamente no código, foram movidas para o arquivo `docker-compose.yml`. 

Adicionalmente, foi adicionado um **healthcheck** para garantir que o serviço do backend esteja pronto antes que o NGINX e o balanceamento de carga sejam ativados. 

Para isso foi necessário adicionar uma linha para instalar o curl dentro do Dockerfile para poder realizar **healthcheck**. 

Abaixo está a configuração do backend no arquivo `docker-compose.yml`:

```yaml
guess:
  build: .
  restart: always
  environment:
    - FLASK_APP=run.py
    - FLASK_DB_TYPE=postgres
    - FLASK_DB_USER=myuser
    - FLASK_DB_PASSWORD=mypassword
    - FLASK_DB_NAME=mydatabase
    - FLASK_DB_HOST=postgres
    - FLASK_DB_PORT=5432
  depends_on:
    - postgres
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
    interval: 10s
    timeout: 5s
    retries: 3
    start_period: 20s
```
Abaixo está a configuração do Dockerfile da raiz do projeto alterado com a instalação do curl na 3ª linha:

```yaml
FROM python:3.8-slim
WORKDIR /app
RUN apt-get update && apt-get install curl -y
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
EXPOSE 5000
CMD ["sh", "./start-backend.sh"]
```

#### 1.3. Banco de Dados (PostgreSQL)

Foi adicionada uma configuração para o serviço PostgreSQL, onde as credenciais e o nome do banco de dados são fornecidos por meio de variáveis de ambiente. Além disso, um volume nomeado "postgres-data" foi mapeado para garantir a persistência dos dados, assegurando que o banco de dados seja preservado mesmo após a reinicialização do container.

```yaml
postgres:
  image: postgres:latest
  restart: always
  environment:
    POSTGRES_USER: myuser
    POSTGRES_PASSWORD: mypassword
    POSTGRES_DB: mydatabase
  volumes:
    - postgres-data:/var/lib/postgresql/data
```
#### 1.4. Load Balancer (NGINX)

Foi configurado um serviço NGINX para funcionar como balanceador de carga, distribuindo as requisições entre o frontend e o backend (guess). O arquivo de configuração nginx.conf, que está localizado na raiz do repositório, é mapeado para dentro do container, onde atua como o arquivo de configuração principal do NGINX.

```yaml
nginx:
  image: nginx:latest
  restart: always
  ports:
    - "80:80"
  volumes:
    - ./nginx.conf:/etc/nginx/nginx.conf
  depends_on:
    - frontend
    - guess
```
#### 1.5. Volume para Persistência do PostgreSQL:

Foi criado um volume chamado "postgres-data" para garantir a persistência dos dados do banco de dados PostgreSQL. Esse volume é montado no bloco de configuração do container do PostgreSQL, permitindo que os dados sejam preservados mesmo após a reinicialização ou interrupção do container.

```yaml
volumes:
  postgres-data:
```
---

### 2. Balanceamento de Carga com NGINX:

Foi criada uma configuração no NGINX onde a rota raiz (/) é direcionada para o frontend. As demais rotas, como /create, /breaker e /guess, são encaminhadas para o backend. Para realizar essa divisão de rotas, foi utilizada uma expressão regular que filtra as rotas de maneira adequada.

```yaml
events { }

http {
    server {
        listen 80;

        location / {
            proxy_pass http://frontend:3000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location ~ ^/(create|breaker|guess) {
            proxy_pass http://guess:5000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```
---

### 3. Guia de Atualização: Docker Compose, Backend, Frontend e Load Balancer

#### 3.1. Balanceamento de Carga com NGINX:
Para modificar rotas, portas ou outras configurações, edite o arquivo nginx.conf localizado na raiz do repositório.
Se for necessário alterar a versão da imagem do NGINX, faça a mudança no arquivo docker-compose.yml.

Depois, execute:

```bash
docker-compose down
docker-compose up
```

#### 3.2. Atualize o Frontend:

Para atualizar o frontend, modifique o Dockerfile localizado na pasta frontend da aplicação ou altere o arquivo docker-compose.yml, conforme necessário.

Depois, execute:

```bash
docker-compose down
docker-compose up
```

#### 3.3. Atualize o Backend:

Para atualizar o backend, edite as variáveis de ambiente no arquivo docker-compose.yml ou modifique o Dockerfile que está na raiz do repositório da aplicação, conforme necessário.
Depois, execute:

```bash
docker-compose down
docker-compose up
```

#### 3.4. Atualize o docker-compose.yml:

Faça as alterações necessárias (como troca de portas, variáveis de ambiente, healthcheck, etc.) no arquivo docker-compose.yml. Após concluir, execute os comandos abaixo para aplicar as mudanças:

Depois, execute:

```bash
docker-compose down
docker-compose up
```

## Referências

- [Documentação do Docker](https://docs.docker.com/engine/install/)
- [Documentação do Docker Compose](https://docs.docker.com/compose/install/)
- [Documentação do Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
- [Docker Hub - Repositório de Imagens](https://hub.docker.com/)
- [Markdown Guide - Sintaxe Markdown](https://www.markdownguide.org/basic-syntax/)
- [Documentação do PostgreSQL](https://www.postgresql.org/docs/)
- [Documentação do NGINX](https://nginx.org/en/docs/)
- [PUC Minas](https://www.pucminas.br)
