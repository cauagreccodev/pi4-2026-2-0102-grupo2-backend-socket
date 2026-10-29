# HelpDesk — Backend Socket (WebSocket & Real-Time Events)

> **Projeto Integrador IV · Grupo 2** — Servidor de mensageria concorrente em tempo real, implementando o protocolo WebSocket (RFC 6455) em Java 21 Puro (sem frameworks) para distribuição de eventos de chamados (`CT22`) e notificações push (`ADR 0002`), conectado ao PostgreSQL em nuvem no **Neon.tech**.

<p align="center">
  <img src="https://img.shields.io/badge/M%C3%B3dulo-Backend--Socket-00B4D8?style=for-the-badge&logo=git&logoColor=white" />
  <img src="https://img.shields.io/badge/Vers%C3%A3o_M%C3%B3dulo-v0.5.0-10B981?style=for-the-badge&logo=git&logoColor=white" />
  <img src="https://img.shields.io/badge/Monorepo-v0.5.0-00B4D8?style=for-the-badge&logo=github&logoColor=white" />
  <img src="https://img.shields.io/badge/Java-21_Puro-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white" />
  <img src="https://img.shields.io/badge/Protocolo-RFC_6455_WebSocket-7C3AED?style=for-the-badge&logo=socketdotio&logoColor=white" />
  <img src="https://img.shields.io/badge/Database-Neon_PostgreSQL-336791?style=for-the-badge&logo=postgresql&logoColor=white" />
  <img src="https://img.shields.io/badge/Deploy-Render_Cloud-000?style=for-the-badge&logo=render&logoColor=white" />
  <img src="https://img.shields.io/badge/Sync-GitHub_Actions-2088FF?style=for-the-badge&logo=githubactions&logoColor=white" />
</p>

---

## Sumário

- [Visão Geral do Módulo](#visão-geral-do-módulo)
- [Status Atual do Módulo (v0.5.0)](#status-atual-do-módulo-v050)
- [Histórico de Versões (Changelog)](#histórico-de-versões-changelog)
- [Responsável & Papel no Projeto](#responsável--papel-no-projeto)
- [Arquitetura e Concorrência](#arquitetura-e-concorrência)
  - [Estruturas de Dados Concorrentes](#estruturas-de-dados-concorrentes)
  - [Fluxo de Comunicação entre Módulos](#fluxo-de-comunicação-entre-módulos)
- [Implementação da RFC 6455 (Java Puro)](#implementação-da-rfc-6455-java-puro)
  - [Handshake HTTP para WebSocket](#handshake-http-para-websocket)
  - [Tratamento de Frames & Máscaras](#tratamento-de-frames--máscaras)
  - [Health Check HTTP & Prevenção de Cold Start](#health-check-http--prevenção-de-cold-start)
- [Contratos de Integração Atendidos](#contratos-de-integração-atendidos)
- [Decisões Arquiteturais (ADRs) & Descarte de Ideias](#decisões-arquiteturais-adrs--descarte-de-ideias)
  - [ADR 0001: Estruturas Concorrentes & Descarte de Chat](#adr-0001-estruturas-concorrentes--descarte-de-chat)
  - [ADR 0002: Notificações em Tempo Real (Push Ping)](#adr-0002-notificações-em-tempo-real-push-ping)
- [Estrutura do Repositório](#estrutura-do-repositório)
- [Como Executar Localmente](#como-executar-localmente)
  - [Pré-requisitos](#pré-requisitos)
  - [Variáveis de Ambiente](#variáveis-de-ambiente)
  - [Compilação e Execução Manual](#compilação-e-execução-manual)
  - [Execução via Docker](#execução-via-docker)
- [Pipeline de CI/CD (Sync com o Monorepo)](#pipeline-de-cicd-sync-com-o-monorepo)

---

## Visão Geral do Módulo

O **Backend Socket** é o componente de infraestrutura responsável por garantir que o Help Desk opere de forma reativa e instantânea. Ele atua como um hub de mensageria leve 100% baseado no protocolo **WebSocket (RFC 6455)**, mantendo sessões persistentes com os navegadores dos usuários e recebendo eventos de chamados gerados pelo **Backend API**.

Principais atribuições:
1. **Orquestração de Sessões WebSocket (`CT19 — autenticarConexao`)**: Gerencia o ciclo de vida das conexões ativas, validando a assinatura HMAC-SHA256 do token e consultando o perfil (`role`) e status (`is_active`) do usuário no banco PostgreSQL do **Neon.tech** (`CT13`).
2. **Recebimento Seguro de Eventos (`CT18 — publicarEvento`)**: Recebe eventos de abertura e atualização de chamados emitidos pela `backend-api` via WebSocket, autenticando o segredo compartilhado (`INTERNAL_SECRET`) em tempo constante.
3. **Distribuição Seletiva em Tempo Real (`CT22 — chamadoAtualizado`)**: Notifica instantaneamente os painéis de todos os técnicos (`SUPPORT`/`ADMIN`) conectados e exclusivamente as abas do cliente solicitante (`clientId`) daquele chamado.
4. **Disparo de Notificações Push (`ADR 0002 — NOTIFICATION_PING`)**: Envia alertas leves em tempo real direcionados ao usuário destinatário, atualizando o contador visual do sino e acionando feedback sonoro.
5. **Interoperabilidade Dual (HTTP + WS)**: Atende requisições de Health Check (`GET /health`, `GET /`, `HEAD /`) na mesma porta pública do WebSocket (`PORT` / `5000`), evitando que o **Render** suspenda a aplicação por inatividade.

---

## Status Atual do Módulo (v0.5.0)

> **Servidor WebSocket RFC 6455 100% completo e operacional em Java 21 Puro: contratos `CT18`, `CT19`, `CT22` e `CT13` implementados, estado concorrente `ServerState` (`ADR 0001`), `ClientConnection` com *outbox* assíncrono, `UserDao` integrado ao Neon PostgreSQL via TLS e validado com Análise de Valores de Fronteira (BVA).**

Principais entregas ativas na versão:
- **Motor Completo do Servidor WebSocket (`WebSocketClientHandler.java`, `ServerState.java`, `ClientConnection.java`)**: Supervisão de sessões, fila assíncrona de eventos (`LinkedBlockingQueue<Runnable>`), *outbox* individual por conexão (`LinkedBlockingQueue<String>`) e roteamento em tempo real (`CT18`, `CT19`, `CT22` e `NOTIFICATION_PING`).
- **Camada de Acesso a Dados e Modelos (`UserDao.java`, `SocketMessage.java`, `UserSessionInfo.java`, `TicketEventPayload.java`)**: Consulta parametrizada na tabela `users` do Neon DB (`CT13`), validação rigorosa de campos obrigatórios e sanitização automática de dados sensíveis (`secret`).
- **Conexão e Pooling com PostgreSQL/Neon DB (`Database.java`)**: Conexão JDBC nativa com driver `org.postgresql.Driver`, pool concorrente thread-safe (`ArrayBlockingQueue`) e suporte obrigatório a SSL/TLS (`sslmode=require`).
- **Validação Criptográfica de Tokens (`AuthUtil.java`)**: Verificação de integridade e expiração de tokens via HMAC-SHA256, 100% compatível com o `backend-api`.
- **Serialização de Mensagens (`JsonUtil.java`)**: Manipulação e parsing JSON com Google Gson para envelopes padronizados (`{"type": "...", "data": {...}}`).

---

## Histórico de Versões (Changelog)

### [v0.5.0] — Servidor WebSocket Completo (CT18, CT19, CT22), Modelos e UserDao no Neon (CT13)
*Versão atual do módulo (Cards `#915` e `#924`)*

- **Motor do Servidor & Concorrência (`server/`)**:
  - Implementação de `ClientConnection.java` representando cada sessão WebSocket autenticada com fila de saída exclusiva limitada (`LinkedBlockingQueue<String>` de 100 mensagens) e thread dedicada de escrita (`ADR 0001`).
  - Implementação de `ServerState.java` com `ConcurrentHashMap<Integer, Set<ClientConnection>>` para múltiplas abas por usuário, fila assíncrona de eventos (`LinkedBlockingQueue<Runnable>`) e filtro seletivo de broadcast para o contrato `CT22 (chamadoAtualizado)` e `NOTIFICATION_PING` (`ADR 0002`).
  - Implementação de `WebSocketClientHandler.java` atendendo os contratos `CT19 (autenticarConexao)` e `CT18 (publicarEvento)` sobre WebSocket RFC 6455, com validação de `INTERNAL_SECRET` via `MessageDigest.isEqual` e tratamento completo dos códigos de erro (`NAO_AUTENTICADO`, `TOKEN_INVALIDO`, `USUARIO_INATIVO`).
  - Atualização da `Main.java` e do `Dockerfile` (compilação UTF-8) para execução de produção no Render.
- **Modelos & Persistência (`model/` e `dao/`)**:
  - Implementação de `SocketMessage.java`, `UserSessionInfo.java` e `TicketEventPayload.java` (com validação de domínio e omissão de `secret` na conversão para `CT22`).
  - Implementação de `UserDao.java` (`CT13: buscarUsuarioPorId`) consultando `id, name, email, role, is_active` no Neon PostgreSQL.
  - Validação completa com **Análise de Valores de Fronteira (BVA)**, Particionamento em Classes de Equivalência e testes de integração no Neon.tech e no Console do navegador.

### [v0.4.1] — Detalhamento das Notificações em Tempo Real no Escopo Oficial (`#912`)

- **Documentação & Escopo**:
  - Atualização de `docs/escopo.pdf` incluindo a especificação das notificações push e atualização em tempo real diretamente na Tabela de Entregáveis (`E4`, `E5` e `E6`) e quadro técnico dos contratos (`CT19`, `CT18`, `CT22` e `ADR 0002`).

### [v0.4.0] — Camada de Utilitários (Gson/HMAC) e Conexão Neon DB (`#912`, `#915`, `#924`)

- **Banco de Dados & Conexão**:
  - Implementação de `Database.java` com pool de conexões JDBC (`org.postgresql.Driver`) para o PostgreSQL hospedado no Neon.tech com TLS (`sslmode=require`).
- **Autenticação & Segurança**:
  - Implementação de `AuthUtil.java` com algoritmo HMAC-SHA256 compatível com o `backend-api` para validação do token no handshake WebSocket (`CT19: autenticarConexao`).
- **Serialização JSON**:
  - Implementação de `JsonUtil.java` encapsulando Google Gson para serialização de envelopes de eventos e erros (`CT18`, `CT19`, `CT22`).

### [v0.3.1] — Aprimoramento do README.md e Documentação Especializada do Módulo Socket

- **Documentação**:
  - Reestruturação completa do `README.md` focada especificamente na arquitetura interna do microsserviço **Backend Socket**.

### [v0.3.0] — Base do Servidor WebSocket (RFC 6455), Health Check HTTP e Sync CI/CD

- **Backend Socket**:
  - Implementação de `WebSocketUtil.java` para handshake e tratamento de frames RFC 6455 em Java puro, rota HTTP `/health` e workflow GitHub Actions (`sync-to-main.yml`).

### [v0.2.1] — Correção e Revisão de Escopo na ADR 0001

- **Documentação / ADR**:
  - Formalização do descarte de salas de chat por chamado em favor de eventos de chamados (`CT22`) e notificações push (`ADR 0002`).

### [v0.2.0] — Especificação de Requisitos, Fluxos de Integração e ADRs

- **Documentação**:
  - Inclusão do documento oficial de escopo (`docs/escopo.pdf`), `ADR 0002` e contratos de fluxo em `docs/fluxos/`.

### [v0.1.0] — Inicialização do Microsserviço Backend-Socket Desacoplado

- **Infraestrutura**:
  - Estruturação inicial do repositório dedicado para o servidor WebSocket, com `Dockerfile` multi-stage e dependências base (`postgresql.jar` e `gson.jar`).

---

## Responsável & Papel no Projeto

* **Responsável pelo Módulo**: Cauã Vasconcelos Grecco De Faria (25006367)
* **Cards no OpenProject**: `#912` (`H01.3 Repos`), `#915` (`H01.6 Socket — CT18`) e `#924` (`H05.1 WebSocket Público — CT19 / CT22`)
* **Repositório Oficial**: [pi4-2026-2-0102-grupo2-backend-socket](https://github.com/cauagreccodev/pi4-2026-2-0102-grupo2-backend-socket)
* **Repositório Central do Grupo**: [pi4-2026-2-0102-grupo2](https://github.com/cauagreccodev/pi4-2026-2-0102-grupo2)
* **Frontend**: [pi4-0102-g2-frontend](https://github.com/Vinicius-Heinzl/pi4-0102-g2-frontend) (Vinicius Fortes Heinzl)
* **Backend API & Banco**: [backend-api](https://github.com/Shinypuc/backend-api) (Kaue Rodrigues Seixas)

---

## Arquitetura e Concorrência

Construído sem frameworks externos (como Spring Boot, Netty ou Micronaut), o servidor utiliza exclusivamente os recursos concorrentes da biblioteca padrão do Java (`java.util.concurrent`), sockets nativos (`java.net.ServerSocket`) e as bibliotecas `gson.jar` e `postgresql.jar` em `lib/`.

### Estruturas de Dados Concorrentes

Conforme definido na **ADR 0001**, o estado de memória do servidor (`ServerState.java`) foi projetado para alta concorrência *thread-safe* sem bloqueios globais:

- **`ConcurrentHashMap<Integer, Set<ClientConnection>>`**: Mapeia o `userId` para o conjunto concorrente de conexões WebSocket ativas daquele usuário, suportando múltiplos dispositivos e abas simultâneas com operações de busca e broadcast em $O(1)$.
- **`LinkedBlockingQueue<String>` (Outbox Limitado por Cliente)**: Buffer de saída exclusivo de cada `ClientConnection` (capacidade de 100 mensagens) consumido por uma thread dedicada (`ws-writer-user-<id>`). Nenhuma thread escreve diretamente no socket de outra; clientes lentos que estourarem a fila são desconectados sem travar o restante do servidor.
- **`LinkedBlockingQueue<Runnable>` (Fila Assíncrona de Eventos)**: Desacopla o recebimento do evento `CT18` na rede do processamento e distribuição do broadcast `CT22` pelos *workers* (`ws-event-worker`).
- **`ExecutorService (CachedThreadPool)`**: Pool dinâmico de threads para atendimento simultâneo das conexões `WebSocketClientHandler`.

### Fluxo de Comunicação entre Módulos

```
  ┌─────────────────────────────────────────────────────────────┐
  │            Navegadores dos Usuários (Vercel)                │
  │     (Painel do Solicitante / Painel Kanban dos Técnicos)    │
  └──────────────────────────────┬──────────────────────────────┘
                                 │ ▲
         1. Abre WebSocket e     │ │ 4. Entrega Evento em Tempo Real
         Autentica Token (CT19)  │ │    via WebSocket (CT22 / Ping)
                                 ▼ │
                   ┌─────────────────────────────┐
                   │       BACKEND SOCKET        │
                   │  (Java 21 / RFC 6455 :5000) │
                   └──────┬──────────────▲───────┘
                          │              │
    2. Valida Usuário     │              │ 3. Publica Evento via WebSocket
       no Banco (CT13)    │              │    com INTERNAL_SECRET (CT18)
                          ▼              │
                   ┌─────────────┐ ┌─────┴───────────────────────┐
                   │ PostgreSQL  │◄┤         BACKEND API         │
                   │ (Neon.tech) │ │  (Java 21 / Render Cloud)   │
                   └─────────────┘ └─────────────────────────────┘
```

1. O **Frontend** abre a conexão WebSocket na porta pública (`ws://` ou `wss://`) e envia o token de autenticação (`CT19: autenticarConexao`).
2. O **Backend Socket** valida a assinatura HMAC-SHA256 do token e consulta a tabela `users` no **PostgreSQL (Neon.tech)** via `UserDao` (`CT13: buscarUsuarioPorId`), registrando a sessão no `ServerState`.
3. Quando um chamado é criado ou atualizado, o **Backend API** grava a transação no **Neon PostgreSQL** e envia o evento via WebSocket ao **Backend Socket** (`CT18: publicarEvento`) autenticado com `INTERNAL_SECRET`.
4. O **Backend Socket** identifica as sessões ativas dos interessados (solicitante `clientId`, técnico `supportId` e todos os `SUPPORT`/`ADMIN` online) e transmite a atualização imediatamente via frames WebSocket (`CT22: chamadoAtualizado`).

---

## Implementação da RFC 6455 (Java Puro)

A classe `com.helpdesk.socket.util.WebSocketUtil` traduz o fluxo de bytes do socket nativo em conformidade com o padrão da RFC 6455:

### Handshake HTTP para WebSocket

Ao receber uma conexão, o servidor lê os cabeçalhos HTTP byte a byte (sem usar `BufferedReader`, garantindo que nenhum byte do primeiro frame WebSocket seja engolido):
1. Verifica a presença do cabeçalho `Sec-WebSocket-Key`.
2. Concatena a chave com a constante normativa (Magic GUID): `258EAFA5-E914-47DA-95CA-C5AB0DC85B11`.
3. Calcula o hash criptográfico SHA-1 e codifica o resultado em Base64.
4. Devolve a resposta padronizada com o status `HTTP/1.1 101 Switching Protocols`:
   ```http
   HTTP/1.1 101 Switching Protocols
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
   ```

### Tratamento de Frames & Máscaras

- **Leitura**: Todo frame enviado pelo cliente possui uma máscara de 4 bytes (`Masked`). O servidor extrai a chave de máscara e aplica a operação XOR byte a byte para revelar o payload UTF-8 original.
- **Tamanhos Suportados**: Decodifica comprimentos de payload padrão (até 125 bytes), estendidos de 16 bits (até 65.535 bytes) e de 64 bits (com proteção contra frames acima de 64 KB).
- **Tipos de Frames (Opcodes)**:
  - `0x1`: Frame de Texto UTF-8.
  - `0x8`: Frame de Fechamento da Conexão (`Close`).
  - `0x9`: Frame de `Ping` (responde automaticamente com `0xA - Pong`).

### Health Check HTTP & Prevenção de Cold Start

Se a requisição inicial recebida não contiver `Sec-WebSocket-Key` (como requisições `GET /health`, `GET /` ou `HEAD`), o `WebSocketUtil` identifica o probe HTTP e responde imediatamente com `200 OK` e um JSON informativo:
```json
{
  "status": "UP",
  "service": "backend-socket",
  "protocol": "websocket",
  "version": "v0.5.0",
  "frontend": "https://help-desk-pi-iv.vercel.app"
}
```
Isso assegura compatibilidade com monitores de uptime e cron jobs de keep-alive no Render, eliminando *cold starts* sem necessidade de abrir outra porta HTTP.

---

## Contratos de Integração Atendidos

As especificações contratuais do módulo estão documentadas na pasta [`docs/fluxos/`](./docs/fluxos/):

| Contrato | Protocolo / Canal | Direção | Propósito | Atende |
|:---|:---|:---|:---|:---:|
| **[CT19](./docs/fluxos/CT19-autenticarConexao.md)** | WebSocket (`:5000` / `wss://`) | Frontend → Backend Socket | Autentica a sessão WebSocket com token HMAC-SHA256, consulta o usuário no Neon (`CT13`) e retorna `conexaoAutenticada`. | E4, E5, E6 |
| **[CT13](./docs/fluxos/visao-geral.md)** | JDBC TLS (`sslmode=require`) | Backend Socket → Banco (Neon) | Consulta `id, name, email, role, is_active` na tabela `users` (`UserDao.java`) durante a autenticação (`CT19`). | E4, E5, E6 |
| **[CT18](./docs/fluxos/CT18-publicarEvento.md)** | WebSocket (`:5000` / `wss://`) | Backend API → Backend Socket | Recebe eventos de criação (`CREATED`) e alteração (`UPDATED`) de chamados protegidos por `INTERNAL_SECRET`. | E4, E5 |
| **[CT22](./docs/fluxos/CT22-chamadoAtualizado.md)** | WebSocket (`:5000` / `wss://`) | Backend Socket → Frontend | Transmite em tempo real a atualização do chamado para o solicitante (`clientId`), técnico (`supportId`) e equipe `SUPPORT`/`ADMIN`. | E4, E5, E6 |

---

## Decisões Arquiteturais (ADRs) & Descarte de Ideias

### ADR 0001: Estruturas Concorrentes & Descarte de Chat

> **Documento:** [`docs/adr/0001-servidor.md`](./docs/adr/0001-servidor.md)

* **Status:** Aceito (com escopo revisado).
* **Decisão de Descarte Formal:**  
  Durante a concepção inicial, avaliou-se a criação de salas de chat em tempo real por chamado (`ticketId` → conexões), buffer de mensagens em memória (`ConcurrentLinkedDeque`), filas de prioridade (`PriorityBlockingQueue`) e controle de SLA via `DelayQueue` (estilo Teams/WhatsApp).  
  **Essa funcionalidade foi deliberadamente descartada do escopo** para simplificar a entrega, reduzir pontos de falha e garantir estabilidade. O servidor WebSocket foi focado estritamente na entrega confiável de eventos do Kanban (`chamadoAtualizado` — `CT22`) e notificações push (`ADR 0002`). Toda a auditoria e histórico de mensagens permanecem transacionalmente salvos na tabela `ticket_timeline` do PostgreSQL no Neon.

### ADR 0002: Notificações em Tempo Real (Push Ping)

> **Documento:** [`docs/adr/0002-notificacao.md`](./docs/adr/0002-notificacao.md)

* **Status:** Aceito.
* **Mecânica Adotada:**  
  Substituição de polling contínuo por envio de evento leve (`NOTIFICATION_PING`) via WebSocket diretamente às sessões ativas do usuário destinatário (`userId`). O frontend atualiza o contador no sino do header e reproduz feedback sonoro (*ping sound*).

---

## Estrutura do Repositório

```text
pi4-2026-2-0102-grupo2-backend-socket/
├── .github/
│   └── workflows/
│       └── sync-to-main.yml                  # Workflow automático de sincronização com o monorepo
├── backend-socket/
│   ├── Dockerfile                            # Dockerfile multi-stage (JDK 21 -> JRE 21 Alpine)
│   ├── lib/
│   │   ├── postgresql.jar                    # Driver JDBC PostgreSQL 42.7.3
│   │   └── gson.jar                          # Google Gson 2.11.0
│   └── src/com/helpdesk/socket/
│       ├── Main.java                         # Entry point, inicialização Neon DB e ServerSocket WebSocket
│       ├── config/
│       │   └── Database.java                 # Gerenciador de conexão e pool JDBC TLS (Neon.tech PostgreSQL)
│       ├── dao/
│       │   └── UserDao.java                  # Consulta parametrizada na tabela 'users' no Neon (CT13)
│       ├── model/
│       │   ├── SocketMessage.java            # Envelope padrão JSON {"type": "...", "data": {...}}
│       │   ├── TicketEventPayload.java       # Modelo e validador de domínio dos contratos CT18 e CT22
│       │   └── UserSessionInfo.java          # Dados de sessão do usuário autenticado (id, role, active)
│       ├── server/
│       │   ├── ClientConnection.java         # Sessão WebSocket autenticada com outbox LinkedBlockingQueue
│       │   ├── ServerState.java              # Estado concorrente (ConcurrentHashMap + fila de broadcast CT22)
│       │   └── WebSocketClientHandler.java   # Supervisora de conexões WebSocket (CT19 e CT18)
│       └── util/
│           ├── AuthUtil.java                 # Validação criptográfica de token HMAC-SHA256 (CT19)
│           ├── JsonUtil.java                 # Serialização e parsing JSON estruturado (Google Gson)
│           └── WebSocketUtil.java            # Handshake RFC 6455, decodificação de frames e Health Check
├── docs/
│   ├── adr/
│   │   ├── 0001-servidor.md                  # Registro de decisão de concorrência e descarte de chat
│   │   └── 0002-notificacao.md               # Registro de decisão de notificações push
│   ├── fluxos/                               # Contratos CT18, CT19, CT22 e entregáveis E1 a E9
│   │   ├── CT18-publicarEvento.md
│   │   ├── CT19-autenticarConexao.md
│   │   ├── CT22-chamadoAtualizado.md
│   │   ├── visao-geral.md
│   │   └── README.md
│   └── escopo.pdf                            # Especificação oficial de requisitos (revisada)
├── .gitignore
└── README.md                                 # Esta documentação especializada
```

---

## Como Executar Localmente

### Pré-requisitos
- **Java JDK 21** instalado.
- Acesso à internet para conexão TLS com o banco de dados PostgreSQL no **Neon.tech**.

### Variáveis de Ambiente

O servidor utiliza exatamente o mesmo padrão de variáveis de ambiente do `backend-api` no Render:

| Variável | Padrão | Descrição |
|:---|:---|:---|
| `PORT` ou `WS_PORT` | `5000` | Porta pública para conexões WebSocket (`CT19`, `CT18`, `CT22`) e Health Check HTTP. |
| `INTERNAL_SECRET` | `segredo-interno-entre-backends` | Segredo compartilhado com a API para autenticação de eventos (`CT18`). |
| `JWT_SECRET` | `minha-chave-secreta-super-segura-hmac-256` | Chave HMAC-SHA256 compartilhada com a API para validação de tokens (`CT19`). |
| `DB_URL` / `DATABASE_URL` | *Neon JDBC URL* | URL JDBC PostgreSQL (`jdbc:postgresql://<host>/neondb?sslmode=require`). |
| `DB_HOST` | *(Host do Neon)* | Host do banco PostgreSQL no Neon (usado caso `DB_URL` não seja informada). |
| `DB_PORT` | `5432` | Porta do banco PostgreSQL. |
| `DB_NAME` | `neondb` | Nome do banco de dados no Neon.tech. |
| `DB_USER` | `neondb_owner` | Usuário de autenticação no banco Neon.tech. |
| `DB_PASSWORD` | *(Senha do Neon)* | Senha de autenticação no banco Neon.tech. |

### Compilação e Execução Manual

A partir do diretório `backend-socket/`:

**No Windows (PowerShell):**
```powershell
javac -encoding UTF-8 -cp "lib/*" -d out src/com/helpdesk/socket/Main.java src/com/helpdesk/socket/config/*.java src/com/helpdesk/socket/util/*.java src/com/helpdesk/socket/model/*.java src/com/helpdesk/socket/dao/*.java src/com/helpdesk/socket/server/*.java
java -cp "out;lib/*" com.helpdesk.socket.Main
```

**No Linux / macOS / WSL:**
```bash
mkdir -p out
javac -encoding UTF-8 -cp "lib/*" -d out $(find src -name "*.java")
java -cp "out:lib/*" com.helpdesk.socket.Main
```

### Execução via Docker

```bash
cd backend-socket

# Build da imagem
docker build -t helpdesk-backend-socket .

# Execução do contêiner
docker run -p 5000:5000 --env-file ../.env helpdesk-backend-socket
```

---

## Pipeline de CI/CD (Sync com o Monorepo)

Este repositório possui uma automação em [`.github/workflows/sync-to-main.yml`](./.github/workflows/sync-to-main.yml).

Sempre que alterações são integradas à branch `main` dentro da pasta `backend-socket/`, o GitHub Actions:
1. Clona o repositório principal do grupo (`pi4-2026-2-0102-grupo2`).
2. Sincroniza via `rsync` os arquivos compiláveis e de configuração para a pasta `backend-socket/` no monorepo.
3. Cria um commit automatizado com a mensagem `chore(backend-socket): sync code from backend-socket repo (<SHA>)`.

Isso permite que o desenvolvimento do servidor WebSocket ocorra de forma modular e independente, mantendo o monorepo central sempre atualizado.
