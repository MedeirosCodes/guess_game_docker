# Jogo de Adivinhação com Flask, React e PostgreSQL em Docker

Este projeto demonstra uma aplicação web completa, incluindo um backend Flask (Python), um banco de dados PostgreSQL e um frontend React, orquestrados usando Docker Compose e servidos por Nginx.

## Visão Geral

O projeto é composto por três serviços principais:

* **Backend (Flask)**: Uma aplicação Python Flask que contém a lógica do jogo de adivinhação e interage com o banco de dados PostgreSQL.

* **Banco de Dados (PostgreSQL)**: Um servidor de banco de dados PostgreSQL para armazenar os dados do jogo, como pontuações e informações de jogadores.

* **Frontend (React + Nginx)**: Uma aplicação React que fornece a interface do usuário do jogo. O Nginx atua como um proxy reverso para o backend Flask e serve os arquivos estáticos do frontend.

## Pré-requisitos

Certifique-se de ter as seguintes ferramentas instaladas em sua máquina:

* **Docker Desktop** (ou Docker Engine e Docker Compose)

  * [Instalação do Docker](https://docs.docker.com/get-docker/)

  * [Instalação do Docker Compose](https://docs.docker.com/compose/install/) (geralmente já vem com o Docker Desktop)

## Estrutura do Projeto

~~~Projeto
seu_projeto/
├── .env                  # Variáveis de ambiente para o Docker Compose
├── docker-compose.yml    # Orquestração dos serviços
├── db/
│   └──── Dockerfile        # Dockerfile para o PostgreSQL
├── game/
│   ├── Dockerfile        # Dockerfile para o backend Flask
│   ├── requirements.txt  # Dependências Python (ex: Flask, Gunicorn, psycopg2-binary)
│   └── run.py            # Seu aplicativo Flask (ou app.py, etc.)
│   └──── frontend/
│       ├── Dockerfile        # Dockerfile para build do React e Nginx
│       ├── package.json      # Dependências do React
│       ├── src/              # Código fonte do React
│       └── public/           # Arquivos públicos do React
└── nginx/
    └──── frontend/
        └── nginx.conf        # Configuração do Nginx para proxy reverso e servir estáticos
~~~

## Como Executar (Recomendado: Docker Compose)

Esta é a maneira mais fácil de levantar todos os serviços da aplicação.

### Configuração das Variáveis de Ambiente

Tem um arquivo `.env` na raiz do seu projeto (na mesma pasta do `docker-compose.yml`) com as seguintes variáveis:

Variáveis de ambiente para a aplicação Flask
FLASK_DB_USER=postgres_user    # Usuário do banco
FLASK_DB_NAME=postgres_db      # Nome do banco
FLASK_DB_PASSWORD=postgres123  # Senha de acesso ao banco (recomendavel alterar)


### Iniciando os Serviços

No terminal, navegue até a pasta raiz do seu projeto (onde está o `docker-compose.yml`) e execute o seguinte comando:

 `docker compose up --build -d`


* `--build`: Garante que as imagens dos serviços sejam construídas (ou reconstruídas) antes de iniciar os contêineres.

* `-d`: Inicia os contêineres em modo *detached* (em segundo plano).

Aguarde o Docker baixar as dependências e construir as imagens. Este processo pode levar alguns minutos na primeira vez.

### Acessando a Aplicação

Após todos os serviços estarem em execução, você pode acessar a aplicação frontend através do seu navegador:

[http://localhost:80](http://localhost:80)


## Como Executar (Manual: Sem Docker Compose)

Esta seção é para fins de compreensão ou depuração avançada, não sendo a forma recomendada para uso diário.

### Construindo as Imagens

Primeiro, construa as imagens para cada serviço individualmente: (obs: verifique que está dentro da pasta)

1. **Imagem do Banco de Dados PostgreSQL:**

 `docker build -t postgres_db -f database/Dockerfile .`


*Isso criará a imagem do banco de dados PostgreSQL.*

2. **Imagem do Backend Flask:**

`docker build -t game_backend -f game/Dockerfile .`


*Isso criará a imagem do backend Flask.*

3. **Imagem do Frontend React + Nginx:**

`docker build -t game_frontend_nginx -f game/frontend/Dockerfile .`


*Isso criará a imagem do frontend React servida pelo Nginx.*


### Executando os Contêineres

Após construir as imagens, você precisa executar os contêineres e conectá-los manualmente:

1. **Executar o Contêiner do Banco de Dados:**

~~~comando docker
docker run --name postgres_db

-e POSTGRES_DB=postgres_db

-e POSTGRES_USER=postgres_user

-e POSTGRES_PASSWORD=postgres123

-v pgdata:/var/lib/postgresql/data

-d postgres_db
~~~

2. **Executar o Contêiner do Backend Flask:**

~~~comando docker

docker run --name flask_backend

-e FLASK_APP=run.py

-e FLASK_DB_TYPE=postgres

-e FLASK_DB_USER=postgres_user

-e FLASK_DB_NAME=postgres_db

-e FLASK_DB_PASSWORD=postgres123

-e FLASK_DB_HOST=postgres_db

-e FLASK_DB_PORT=5432

--link postgres_db:db

-p 5000:5000

-d game_backend
~~~

*O `--link postgres_db:db` é usado para conectar o backend ao banco de dados pelo nome do contêiner.*

3. **Executar o Contêiner do Nginx (Frontend):**

~~~comando docker
docker run --name nginx_proxy

-p 80:80

-v $(pwd)/nginx/nginx.conf/nginx.conf:/etc/nginx/nginx.conf:ro

--link flask_backend:backend

-d game_frontend_nginx

~~~

*O `--link flask_backend:backend` é usado para conectar o Nginx ao backend pelo nome do contêiner.*

Após iniciar todos os contêineres, acesse a aplicação em: [http://localhost:80](http://localhost:80)

## Estrutura dos Containers

* **`db` (PostgreSQL)**: Contêiner do banco de dados. Os dados são persistidos em um volume Docker nomeado (`pgdata`).

* **`backend` (Flask)**: Contêiner da aplicação Flask. Configurado(s) com variáveis de ambiente para conexão com o banco de dados. Múltiplas instâncias podem ser configuradas para balanceamento de carga.

* **`nginx` (Nginx)**: Contêiner do Nginx que serve os arquivos estáticos do frontend React e atua como proxy reverso para o(s) contêiner(es) do backend.

## Resiliência e Manutenção

* **Reinício de Containers:** Todos os serviços no `docker-compose.yml` estão configurados com `restart: unless-stopped` ou `restart_policy: on_failure`, garantindo que eles sejam reiniciados automaticamente em caso de falha.

* **Balanceamento de Carga no Proxy Reverso:** O serviço `backend` no `docker-compose.yml` está configurado com `deploy: replicas: 2` (ou mais), e o `nginx.conf` usa `proxy_pass http://backend:5000;`. O Docker Compose, em conjunto com o Nginx, gerencia automaticamente o balanceamento de carga entre as instâncias do backend.

* **Volumes Separados para o Banco de Dados:** O volume nomeado `pgdata` garante que os dados do PostgreSQL não sejam perdidos se o contêiner do banco de dados for removido ou atualizado.

* **Facilidade de Atualização:** Para atualizar qualquer componente, basta alterar a tag da imagem no `Dockerfile` correspondente (se estiver usando uma imagem base específica) ou no `docker-compose.yml` (se estiver usando uma imagem pré-construída) e, em seguida, executar `docker compose up --build -d` novamente.

## Solução de Problemas Comuns

* **Contêineres não iniciam:**

* Verifique os logs dos contêineres: `docker compose logs <nome_do_servico>` (ex: `docker compose logs backend`).

* Certifique-se de que as portas não estão em uso por outros aplicativos na sua máquina.

* **Erro de conexão com o banco de dados:**

* Verifique se as variáveis de ambiente de conexão no `.env` e no `game/Dockerfile` estão corretas e correspondem às do serviço `db`.

* Confirme se o serviço `db` está rodando: `docker ps`.

* Verifique se o `FLASK_DB_HOST` no backend está apontando para `db` (o nome do serviço do banco de dados no `docker-compose.yml`).

* **Frontend não carrega ou API não responde:**

* Verifique os logs do Nginx: `docker compose logs nginx`.

* Confirme se o `nginx/nginx.conf` está montado corretamente e se as configurações de `proxy_pass` e `root` estão corretas.

* Verifique se o backend está rodando e acessível: `docker logs backend`.

* **Problemas de cache:** Se você fez alterações no código e elas não aparecem, tente reconstruir as imagens sem cache: `docker compose build --no-cache`.