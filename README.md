<div align="center">

# MySQL Server — Container Guide

[![MySQL](https://img.shields.io/badge/MySQL-8.0-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](https://hub.docker.com/_/mysql)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://docs.docker.com/compose/)
[![Beekeeper Studio](https://img.shields.io/badge/Beekeeper_Studio-SQL_Client-F8C149?style=for-the-badge&logo=beekeeper-studio&logoColor=black)](https://www.beekeeperstudio.io/)

_Documentação de uso e inicialização do container MySQL para ambientes de desenvolvimento local._

</div>

---

## Autor

<div align="center">

[![GitHub](https://img.shields.io/badge/GitHub-Cardosofiles-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/Cardosofiles)
[![Website](https://img.shields.io/badge/Website-cardosofiles.com.br-0A66C2?style=for-the-badge&logo=google-chrome&logoColor=white)](https://www.cardosofiles.com.br/pt)

</div>

---

## Ambiente de Desenvolvimento Recomendado

Para uma experiência de desenvolvimento consistente, produtiva e bem configurada no terminal Linux, utilize o setup disponível em:

<div align="center">

[![linux-terminal-settings](https://img.shields.io/badge/Cardosofiles-linux--terminal--settings-1DB954?style=for-the-badge&logo=linux&logoColor=white)](https://github.com/Cardosofiles/linux-terminal-settings)

</div>

> Terminal moderno de altíssima performance e nível Sênior. Inclui Zsh + Oh My Zsh, tema Powerlevel10k, Docker Engine Nativo integrado ao `systemd`, aliases, plugins e ferramentas nas versões mais recentes — tudo sem o peso do Docker Desktop.

---

## Pré-requisitos

Certifique-se de ter instalado em sua máquina:

| Ferramenta           | Versão mínima | Link                                                                |
| -------------------- | ------------- | ------------------------------------------------------------------- |
| **Docker**           | `>= 24`       | [docs.docker.com](https://docs.docker.com/engine/install/)          |
| **Docker Compose**   | `>= 2.x`      | [docs.docker.com/compose](https://docs.docker.com/compose/install/) |
| **Beekeeper Studio** | qualquer      | [beekeeperstudio.io](https://www.beekeeperstudio.io/)               |

---

## Variáveis de Ambiente

Na raiz do projeto, crie um arquivo `.env` com base no modelo abaixo:

```env
# Versão da imagem MySQL
MYSQL_VERSION=8.0

# Credenciais
MYSQL_ROOT_PASSWORD=root_secret
MYSQL_DATABASE=app_db
MYSQL_USER=app_user
MYSQL_PASSWORD=app_password

# Porta exposta no host
MYSQL_PORT=3306

# Fuso horário
TZ=America/Sao_Paulo
```

> [!WARNING]
> Nunca versione o arquivo `.env` com credenciais reais. Adicione-o ao `.gitignore`.

---

## Inicializando o Container

### Subir em background

```bash
docker compose up -d
```

### Acompanhar os logs em tempo real

```bash
docker compose logs -f db
```

### Verificar status e saúde

```bash
docker compose ps
```

O container está pronto quando o campo `STATUS` exibir `healthy`.

### Parar o container

```bash
docker compose down
```

### Reset completo — para e remove volumes

```bash
docker compose down -v
```

---

## Scripts de Inicialização

Arquivos `.sql` ou `.sh` colocados em `init-scripts/` são executados **automaticamente** na primeira inicialização, em ordem alfabética.

```
init-scripts/
├── 01_schema.sql      # criação de tabelas
├── 02_seed.sql        # dados iniciais
└── 03_procedures.sql  # stored procedures
```

> [!NOTE]
> Esses scripts só rodam quando o volume `app-mysql-data` ainda não existe. Para re-executá-los, remova o volume com `docker compose down -v`.

---

## Acesso via Terminal

```bash
# Como usuário da aplicação
docker exec -it app-mysql-db mysql -u app_user -p app_db

# Como root
docker exec -it app-mysql-db mysql -u root -p
```

---

## Conectando com Beekeeper Studio

[![Beekeeper Studio](https://img.shields.io/badge/Beekeeper_Studio-Guia_de_Conexão-F8C149?style=flat-square&logo=beekeeper-studio&logoColor=black)](https://www.beekeeperstudio.io/)

### 1 — Abrir o Beekeeper Studio

Inicie o aplicativo e clique em **New Connection**.

### 2 — Selecionar o tipo de conexão

Escolha **MySQL** na lista de bancos suportados.

### 3 — Preencher os dados de conexão

| Campo        | Valor                                        |
| ------------ | -------------------------------------------- |
| **Host**     | `127.0.0.1`                                  |
| **Port**     | `3306` _(ou o valor de `MYSQL_PORT`)_        |
| **User**     | `app_user` _(valor de `MYSQL_USER`)_         |
| **Password** | `app_password` _(valor de `MYSQL_PASSWORD`)_ |
| **Database** | `app_db` _(valor de `MYSQL_DATABASE`)_       |

### 4 — Testar e salvar

Clique em **Test Connection** → confirme o sucesso → **Save** → **Connect**.

### 5 — Executando queries

```sql
-- Listar todas as tabelas
SHOW TABLES;

-- Consultar registros
SELECT * FROM users LIMIT 10;

-- Inserir um registro
INSERT INTO users (name, email)
VALUES ('Cardoso', 'contato@cardosofiles.com.br');
```

| Atalho                 | Ação                       |
| ---------------------- | -------------------------- |
| `Ctrl + Enter`         | Executar query selecionada |
| `Ctrl + Shift + Enter` | Executar todas as queries  |

---

## Estrutura do Projeto

```
mysql-server-db/
├── docker-compose.yml        # definição do serviço MySQL
├── .env                      # variáveis de ambiente (não versionado)
├── init-scripts/             # scripts SQL executados na inicialização
└── docs/
    └── container-initialization.md
```

---

## Referência do Container

| Propriedade        | Valor                    |
| ------------------ | ------------------------ |
| **Nome**           | `app-mysql-db`           |
| **Imagem**         | `mysql:${MYSQL_VERSION}` |
| **Charset**        | `utf8mb4`                |
| **Collation**      | `utf8mb4_unicode_ci`     |
| **Autenticação**   | `caching_sha2_password`  |
| **Rede**           | `app-backend-network`    |
| **Volume**         | `app-mysql-data`         |
| **CPU limit**      | `1.0 vCPU`               |
| **Memory limit**   | `1 GB`                   |
| **Restart policy** | `unless-stopped`         |

---

<div align="center">
  <sub>Crafted with precision by <a href="https://github.com/Cardosofiles"><strong>Cardosofiles</strong></a></sub>
</div>
