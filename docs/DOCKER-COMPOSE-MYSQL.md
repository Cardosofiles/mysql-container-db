# 📦 Documentação: Docker Compose — Banco de Dados MySQL

Este documento explica, linha a linha, o arquivo `docker-compose.yml` responsável por subir
um container MySQL de forma isolada, segura e configurável.

---

## O que é o Docker Compose?

O **Docker Compose** é uma ferramenta que permite definir e rodar múltiplos containers Docker
usando um único arquivo de configuração chamado `docker-compose.yml`. Em vez de digitar longos
comandos no terminal, você descreve tudo nesse arquivo e sobe o ambiente com um simples:

```bash
docker compose up
```

---

## Estrutura Geral do Arquivo

O arquivo é dividido em três seções principais:

- **`services`** — define os containers que serão criados
- **`networks`** — define as redes de comunicação entre containers
- **`volumes`** — define onde os dados serão armazenados de forma persistente

---

## Seção `services`

```yaml
services:
```

Inicia a declaração dos serviços (containers). Cada item abaixo representa um container
que o Docker vai criar e gerenciar.

---

### Serviço `db`

```yaml
  db:
```

Nome **lógico** do serviço dentro do Compose. Outros containers podem referenciar este
banco de dados pelo nome `db` na rede interna.

```yaml
    image: mysql:${MYSQL_VERSION}
```

Define a **imagem Docker** a ser usada — neste caso, a imagem oficial do MySQL.
O `${MYSQL_VERSION}` é uma **variável de ambiente** lida de um arquivo `.env`
(ex: `MYSQL_VERSION=8.0`). Isso evita deixar versões fixas no código.

```yaml
    container_name: mysql-container-db
```

Define o **nome do container** que aparecerá no `docker ps`. Sem isso, o Docker
geraria um nome aleatório como `projeto_db_1`.

---

### `environment`

```yaml
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      TZ: ${TZ}
```

**Variáveis de ambiente** passadas para dentro do container. O MySQL as lê na inicialização
para se configurar automaticamente:

- `MYSQL_ROOT_PASSWORD` — senha do usuário administrador `root`
- `MYSQL_DATABASE` — nome do banco de dados criado automaticamente ao subir o container
- `MYSQL_USER` / `MYSQL_PASSWORD` — cria um usuário comum com acesso ao banco acima
- `TZ` — define o fuso horário dentro do container (ex: `America/Sao_Paulo`)

> ⚠️ **Nunca** coloque senhas diretamente aqui. Sempre use um arquivo `.env` que **não deve
> ser commitado** no Git — adicione-o ao `.gitignore`.

---

### `ports`

```yaml
    ports:
      - "${MYSQL_PORT}:3306"
```

Mapeia uma **porta do seu computador (host)** para a porta do container.
O formato é sempre `HOST:CONTAINER`:

- `3306` é a porta padrão do MySQL dentro do container
- `${MYSQL_PORT}` é a porta que você usará para conectar pelo host
  (ex: `3307:3306` evita conflito se você já tiver MySQL instalado localmente)

---

### `volumes`

```yaml
    volumes:
      - db_data:/var/lib/mysql
      - ./init-scripts:/docker-entrypoint-initdb.d
```

Volumes são usados para **persistir dados** além do ciclo de vida do container:

- `db_data:/var/lib/mysql` — mapeia o volume nomeado `db_data` para o diretório onde o MySQL
  armazena seus dados. Se o container for destruído e recriado, os dados **não são perdidos**.
- `./init-scripts:/docker-entrypoint-initdb.d` — mapeia uma pasta local `init-scripts/` para
  um diretório especial do MySQL. Qualquer arquivo `.sql` ou `.sh` colocado lá é **executado
  automaticamente** na primeira inicialização do banco. Ótimo para criar tabelas e seeds iniciais.

---

### `command`

```yaml
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
```

Passa **argumentos diretamente para o processo do MySQL** no startup:

- `utf8mb4` — suporte completo a Unicode, incluindo emojis (o `utf8` padrão do MySQL não
  suporta todos os caracteres)
- `utf8mb4_unicode_ci` — define a ordenação e comparação de strings (`ci` = case-insensitive)

---

### `networks`

```yaml
    networks:
      - backend-network
```

Conecta este container à rede `backend-network`. Containers na **mesma rede** podem se
comunicar pelo nome do serviço (ex: sua API pode conectar ao banco usando `db:3306`).
Containers fora dessa rede não conseguem acessar.

---

### `restart`

```yaml
    restart: unless-stopped
```

Define a **política de reinício** do container:

| Valor           | Comportamento                                          |
| --------------- | ------------------------------------------------------ |
| `no`            | Nunca reinicia automaticamente                         |
| `always`        | Sempre reinicia, inclusive após reboot da máquina      |
| `on-failure`    | Reinicia apenas se o processo sair com erro            |
| `unless-stopped`| Reinicia sempre, exceto se você parou manualmente      |

`unless-stopped` é a opção mais usada em produção.

---

### `healthcheck`

```yaml
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "mysql -h localhost -u root -p$$MYSQL_ROOT_PASSWORD -e 'SELECT 1' || exit 1",
        ]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
```

O **healthcheck** verifica se o MySQL está realmente pronto para aceitar conexões —
não apenas se o processo está rodando:

- `test` — comando executado dentro do container para testar a saúde. Tenta executar um
  `SELECT 1` simples. Se falhar, retorna `exit 1`.
- `$$MYSQL_ROOT_PASSWORD` — o `$$` é necessário para escapar o `$` no YAML e fazer o Docker
  interpretar como variável de ambiente do container, não do host.
- `interval: 10s` — executa o teste a cada 10 segundos
- `timeout: 5s` — se o teste demorar mais de 5 segundos, considera falha
- `retries: 5` — após 5 falhas consecutivas, o container é marcado como `unhealthy`
- `start_period: 30s` — aguarda 30 segundos antes de começar os testes (tempo para o MySQL
  inicializar)

> 💡 Outros containers podem usar `depends_on: condition: service_healthy` para só iniciar
> após este healthcheck passar.

---

### `deploy.resources`

```yaml
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 10G
        reservations:
          cpus: "0.25"
          memory: 256M
```

Define **limites de recursos** do container:

- `limits` — máximo que o container pode consumir. `cpus: "1.0"` equivale a 1 núcleo de CPU;
  `memory: 10G` é o limite de RAM.
- `reservations` — mínimo garantido para o container. Serve como referência e soft limit
  em ambientes Compose standalone.

> Ajuste o `memory` conforme o tamanho do seu banco e a RAM disponível na sua máquina.

---

### `logging`

```yaml
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

Configura o **driver de logs** do container:

- `json-file` — driver padrão do Docker, salva logs em arquivos JSON no host
- `max-size: "10m"` — cada arquivo de log tem no máximo 10 MB
- `max-file: "3"` — mantém no máximo 3 arquivos (rotação automática)

> Sem essa configuração, os logs crescem indefinidamente e podem encher o disco.

---

## Seção `networks`

```yaml
networks:
  backend-network:
    name: app-backend-network
    driver: bridge
```

Declara a rede usada pelos containers:

- `backend-network` — nome lógico usado dentro do Compose
- `name: app-backend-network` — nome real criado no Docker (visível no `docker network ls`)
- `driver: bridge` — cria uma rede isolada onde os containers se comunicam entre si,
  mas não ficam expostos diretamente à rede externa do host

---

## Seção `volumes`

```yaml
volumes:
  db_data:
    name: app-mysql-data
```

Declara o volume nomeado usado pelo banco:

- `db_data` — nome lógico usado dentro do Compose
- `name: app-mysql-data` — nome real visível no `docker volume ls`

> Volumes nomeados **sobrevivem** a `docker compose down`. Para remover os dados junto
> com os containers, use `docker compose down -v`.

---

## Arquivo `.env` de Exemplo

Crie um arquivo `.env` na raiz do projeto e **nunca o commite no Git**:

```env
MYSQL_VERSION=8.0
MYSQL_ROOT_PASSWORD=super_senha_secreta
MYSQL_DATABASE=meu_banco
MYSQL_USER=app_user
MYSQL_PASSWORD=senha_do_usuario
MYSQL_PORT=3306
TZ=America/Sao_Paulo
```

Adicione ao `.gitignore`:

```gitignore
.env
```

---

## Comandos Essenciais

```bash
# Subir o ambiente em background
docker compose up -d

# Ver os containers rodando
docker compose ps

# Ver os logs do banco em tempo real
docker compose logs -f db

# Parar os containers (sem remover dados)
docker compose down

# Parar e remover TUDO, incluindo volumes (dados perdidos!)
docker compose down -v

# Aplicar mudanças no docker-compose.yml sem perder dados
docker compose up -d --force-recreate
```

---

> 💡 **Dica final:** Sempre que alterar o `docker-compose.yml`, rode
> `docker compose up -d --force-recreate` para aplicar as mudanças sem perder os dados do volume.
