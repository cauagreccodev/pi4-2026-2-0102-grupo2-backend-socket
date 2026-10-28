# HelpDesk — Backend Socket (WebSocket & Real-Time Events)

> **Projeto Integrador IV · Grupo 2** — Servidor de mensageria concorrente em tempo real, implementando o protocolo WebSocket (RFC 6455) e soquetes TCP em Java 21 Puro para distribuição de eventos de chamados e notificações push.

<p align="center">
  <img src="https://img.shields.io/badge/M%C3%B3dulo-Backend--Socket-00B4D8?style=for-the-badge&logo=git&logoColor=white" />
  <img src="https://img.shields.io/badge/Vers%C3%A3o_M%C3%B3dulo-v0.4.0-10B981?style=for-the-badge&logo=git&logoColor=white" />
  <img src="https://img.shields.io/badge/Monorepo-v0.5.0-00B4D8?style=for-the-badge&logo=github&logoColor=white" />
  <img src="https://img.shields.io/badge/Java-21_Puro-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white" />
  <img src="https://img.shields.io/badge/Protocolo-RFC_6455_WebSocket-7C3AED?style=for-the-badge&logo=socketdotio&logoColor=white" />
  <img src="https://img.shields.io/badge/Deploy-Render_Cloud-000?style=for-the-badge&logo=render&logoColor=white" />
  <img src="https://img.shields.io/badge/Sync-GitHub_Actions-2088FF?style=for-the-badge&logo=githubactions&logoColor=white" />
</p>

---

## Sumário

- [Visão Geral do Módulo](#visão-geral-do-módulo)
- [Status Atual do Módulo (v0.4.0)](#status-atual-do-módulo-v040)
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

O **Backend Socket** é o componente de infraestrutura responsável por garantir que o Help Desk opere de forma reativa e instantânea. Ele atua como um hub de mensageria leve, mantendo sessões WebSocket persistentes com os navegadores dos usuários e escutando eventos gerados pelo **Backend API**.

Principais atribuições:
1. **Orquestração de Sessões WebSocket**: Gerencia o ciclo de vida das conexões ativas (autenticação, keep-alive, desconexões e reconexões).
2. **Distribuição de Eventos de Chamados (`chamadoAtualizado`)**: Notifica instantaneamente os painéis de técnicos e clientes sempre que um chamado é aberto, atribuído ou alterado.
3. **Disparo de Notificações Push**: Envia alertas leves em tempo real direcionados ao usuário destinatário, atualizando o badge visual e acionando feedback sonoro.
4. **Interoperabilidade Dual (HTTP + WS)**: Atende requisições de Health Check na mesma porta pública do WebSocket, evitando que provedores de nuvem (Render) suspendam a aplicação por inatividade.

---

## Status Atual do Módulo (v0.4.0)

> **Camada de infraestrutura e utilitários (Neon DB, HMAC-SHA256, Gson) integrada, servidor WebSocket RFC 6455 operacional em Java 21 Puro, Health Check HTTP integrado e pipeline CI/CD de sincronização contínua com o monorepo.**

Principais entregas ativas na versão:
- **Conexão e Pooling com PostgreSQL/Neon DB (`Database.java`)**: Conexão JDBC nativa com driver `org.postgresql.Driver`, pool concorrente thread-safe (`ArrayBlockingQueue`) e suporte a SSL/TLS para nuvem.
- **Validação Criptográfica de Tokens (`AuthUtil.java`)**: Verificação de integridade de tokens Bearer via HMAC-SHA256, alinhada à política de autenticação do `backend-api` para validação no contrato `CT19`.
- **Serialização de Mensagens (`JsonUtil.java`)**: Manipulação e parsing JSON com Google Gson para os contratos de eventos em tempo real (`CT18`, `CT19`, `CT22`).
- **Servidor WebSocket RFC 6455 em Java Puro (`WebSocketUtil.java` / `Main.java`)**: Handshake HTTP `101 Switching Protocols`, decodificação de frames mascarados, frames de controle (`Ping`/`Pong`/`Close`) e probe HTTP de Uptime.
- **Refinamento da Especificação de Requisitos (`docs/escopo.pdf`)**: Documento oficial de escopo atualizado com detalhamento formal das notificações em tempo real.

---

## Histórico de Versões (Changelog)

### [v0.4.0] — Camada de Utilitários (Gson/HMAC), Conexão Neon DB e Refinamento do Escopo
*Versão atual do módulo*

- **Banco de Dados & Conexão**:
  - Implementação de `Database.java` com pool de conexões JDBC (`org.postgresql.Driver`) para o PostgreSQL hospedado no Neon.tech.
  - Suporte flexível a variáveis de ambiente (`DATABASE_URL`, `DB_URL` ou `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`).
- **Autenticação & Segurança**:
  - Implementação de `AuthUtil.java` com algoritmo HMAC-SHA256 compatível com o backend-api para validação do token no handshake WebSocket (`CT19: autenticarConexao`).
- **Serialização JSON**:
  - Implementação de `JsonUtil.java` encapsulando Google Gson para serialização de eventos e mensagens de notificação (`CT18`, `CT22`).
- **Documentação & Requisitos**:
  - Atualização do documento oficial `docs/escopo.pdf` com detalhamento formal das notificações push em tempo real.

### [v0.3.1] — Aprimoramento do README.md e Documentação Especializada do Módulo Socket

- **Documentação**:
  - Reestruturação completa do `README.md` focada especificamente na arquitetura interna do microsserviço **Backend Socket**.
  - Detalhamento do protocolo RFC 6455 (handshake HTTP `101 Switching Protocols`, máscara XOR de 4 bytes e opcodes de controle).
  - Mapeamento dos contratos de tempo real (`CT18`, `CT19`, `CT22`), ADRs (`0001` e `0002`) e instruções de execução e CI/CD.

### [v0.3.0] — Base do Servidor WebSocket (RFC 6455), Health Check HTTP e Sync CI/CD

- **Backend Socket**:
  - Implementação de `WebSocketUtil.java` para handshake e tratamento de frames RFC 6455 em Java puro.
  - Implementação de `Main.java` com `ServerSocket` e `ExecutorService` (`CachedThreadPool`) para conexões concorrentes.
  - Suporte nativo a rota pública HTTP `/health` e `/` na mesma porta do WebSocket para prevenção de *cold start* no Render.
- **CI/CD**:
  - Adição do workflow GitHub Actions (`.github/workflows/sync-to-main.yml`) para sincronização automática de código com o repositório principal.

### [v0.2.1] — Correção e Revisão de Escopo na ADR 0001

- **Documentação / ADR**:
  - Correção de seções faltantes em `docs/adr/0001-servidor.md` e formalização do descarte de salas de chat por chamado em favor de eventos de chamados e notificações push.

### [v0.2.0] — Especificação de Requisitos, Fluxos de Integração e ADRs

- **Documentação**:
  - Inclusão do documento oficial de escopo do projeto (`docs/escopo.pdf`).
  - Adição da `ADR 0002` (Notificações em Tempo Real via Push Ping) e contratos de fluxo em `docs/fluxos/`.

### [v0.1.0] — Inicialização do Microsserviço Backend-Socket Desacoplado

- **Infraestrutura**:
  - Estruturação inicial do repositório dedicado para o servidor de sockets, com `Dockerfile` multi-stage e dependências base (`postgresql.jar` e `gson.jar`).

---

## Responsável & Papel no Projeto

* **Responsável pelo Módulo**: Cauã Vasconcelos Grecco De Faria (25006367)
* **Repositório Oficial**: [pi4-2026-2-0102-grupo2-backend-socket](https://github.com/cauagreccodev/pi4-2026-2-0102-grupo2-backend-socket)
* **Repositório Central do Grupo**: [pi4-2026-2-0102-grupo2](https://github.com/cauagreccodev/pi4-2026-2-0102-grupo2)
* **Frontend**: [pi4-0102-g2-frontend](https://github.com/Vinicius-Heinzl/pi4-0102-g2-frontend) (Vinicius Fortes Heinzl)
* **Backend API & Banco**: [backend-api](https://github.com/Shinypuc/backend-api) (Kaue Rodrigues Seixas)

---

## Arquitetura e Concorrência

Construído sem frameworks externos (como Spring Boot, Netty ou Micronaut), o servidor utiliza exclusivamente os recursos concorrentes da biblioteca padrão do Java (`java.util.concurrent`) e os sockets de rede (`java.net.ServerSocket`).

### Estruturas de Dados Concorrentes

Conforme definido na **ADR 0001**, o estado de memória do servidor foi projetado para alta concorrência *thread-safe* sem bloqueios globais:

- **`ConcurrentHashMap<Integer, Set<Session>>`**: Mapeia o `userId` para o conjunto de conexões ativas daquele usuário, suportando múltiplos dispositivos e abas simultâneas com operações de busca e broadcast em $O(1)$.
- **`LinkedBlockingQueue<String>` (Outbox por Cliente)**: Buffer de saída exclusivo de cada conexão. Nenhuma thread escreve diretamente no socket de outra, garantindo que clientes com conexões lentas ou instáveis não bloqueiem o loop de entrega global.
- **`ExecutorService (CachedThreadPool)`**: Pool dinâmico de threads para processamento assíncrono e isolamento de I/O de rede.

### Fluxo de Comunicação entre Módulos

```
  ┌─────────────────────────────────────────────────────────────┐
  │                 Navegadores dos Usuários                    │
  │     (Painel do Solicitante / Painel Kanban dos Técnicos)    │
  └──────────────────────────────┬──────────────────────────────┘
                                 │ ▲
         1. Abre Conexão e       │ │ 4. Entrega Evento em Tempo Real
         Autentica Token (CT19)  │ │    via WebSocket (CT22)
                                 ▼ │
                   ┌─────────────────────────────┐
                   │       BACKEND SOCKET        │
                   │    (Java 21 / RFC 6455)     │
                   └─────────────▲───────────────┘
                                 │
                                 │ 3. Notifica via TCP Interno (:5001)
                                 │    com INTERNAL_SECRET (CT18)
                                 │
                   ┌─────────────┴───────────────┐
                   │         BACKEND API         │
                   │  (Java 21 / com.sun.httpserver)
                   └─────────────┬───────────────┘
                                 │ 2. Grava Chamado e Timeline
                                 ▼    com Transação Segura
                   ┌─────────────────────────────┐
                   │    PostgreSQL 15+ (Neon)    │
                   └─────────────────────────────┘
```

1. O **Frontend** abre a conexão WebSocket na porta pública e envia o token de autenticação (`CT19: autenticarConexao`).
2. O **Backend API** processa uma ação de negócio (ex: criação ou alteração de status) e consolida a transação no **PostgreSQL**.
3. O **Backend API** aciona o **Backend Socket** via TCP interno na porta 5001 (`CT18: publicarEvento`).
4. O **Backend Socket** identifica as sessões ativas dos interessados e transmite a atualização imediatamente via frames WebSocket (`CT22: chamadoAtualizado`).

---

## Implementação da RFC 6455 (Java Puro)

A classe `com.helpdesk.socket.util.WebSocketUtil` traduz o fluxo de bytes do socket nativo em conformidade com o padrão da RFC 6455:

### Handshake HTTP para WebSocket

Ao receber uma conexão, o servidor lê as linhas HTTP iniciais:
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

- **Leitura**: Por determinação da especificação, todo frame enviado pelo cliente (navegador) possui uma máscara de 4 bytes (`Masked`). O servidor extrai a chave de máscara e aplica a operação XOR byte a byte para revelar o payload original.
- **Tamanhos Suportados**: Decodifica comprimentos de payload padrão (até 125 bytes), estendidos de 16 bits (até 65.535 bytes) e de 64 bits.
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
  "version": "v0.4.3",
  "frontend": "https://help-desk-pi-iv.vercel.app"
}
```
Isso assegura compatibilidade com os cron jobs de keep-alive do plano gratuito do Render, eliminando cold starts sem necessidade de um servidor web separado.

---

## Contratos de Integração Atendidos

As especificações contratuais do módulo estão documentadas na pasta [`docs/fluxos/`](./docs/fluxos/):

| Contrato | Protocolo / Porta | Direção | Propósito | Atende |
|:---|:---|:---|:---|:---:|
| **[CT18](./docs/fluxos/CT18-publicarEvento.md)** | TCP Interno (:5001) | Backend API → Backend Socket | Recebe eventos de criação (`CREATED`) e alteração (`UPDATED`) de chamados protegidos por `INTERNAL_SECRET`. | E4, E5 |
| **[CT19](./docs/fluxos/CT19-autenticarConexao.md)** | WebSocket (:5000) | Frontend → Backend Socket | Autentica a sessão do usuário com base no Bearer Token JWT e valida o perfil/departamento no banco. | E4, E5, E6 |
| **[CT22](./docs/fluxos/CT22-chamadoAtualizado.md)** | WebSocket (:5000) | Backend Socket → Frontend | Transmite em tempo real a atualização do chamado para os solicitantes e técnicos conectados. | E4, E5 |

---

## Decisões Arquiteturais (ADRs) & Descarte de Ideias

### ADR 0001: Estruturas Concorrentes & Descarte de Chat

> **Documento:** [`docs/adr/0001-servidor.md`](./docs/adr/0001-servidor.md)

* **Status:** Aceito (com escopo revisado).
* **Decisão de Descarte Formal:**  
  Durante a concepção inicial, avaliou-se a criação de salas de chat em tempo real por chamado (`ticketId` → conexões), buffer de mensagens em memória (`ConcurrentLinkedDeque`), filas de prioridade (`PriorityBlockingQueue`) e controle de SLA via `DelayQueue` (estilo Teams/WhatsApp).  
  **Essa funcionalidade foi deliberadamente descartada do escopo** para simplificar a entrega, reduzir pontos de falha e garantir estabilidade. O servidor Socket foi focado estritamente na entrega confiável de eventos do Kanban (`chamadoAtualizado`) e notificações push. Toda a auditoria e histórico de mensagens permanecem transacionalmente salvos na tabela `ticket_timeline` do PostgreSQL via API REST.

### ADR 0002: Notificações em Tempo Real (Push Ping)

> **Documento:** [`docs/adr/0002-notificacao.md`](./docs/adr/0002-notificacao.md)

* **Status:** Aceito.
* **Mecânica Adotada:**  
  Substituição de polling contínuo por envio de evento leve (`NOTIFICATION_PING`) via WebSocket diretamente às sessões ativas do usuário destinatário (`userId`). O frontend atualiza o contador no sino do header e reproduz feedback sonoro (*ping sound*).

---

## Estrutura do Repositório

```
pi4-2026-2-0102-grupo2-backend-socket/
├── .github/
│   └── workflows/
│       └── sync-to-main.yml    # Workflow automático de sincronização com o monorepo
├── backend-socket/
│   ├── Dockerfile              # Dockerfile multi-stage (build & runtime Alpine)
│   ├── lib/
│   │   ├── postgresql.jar      # Driver JDBC PostgreSQL 42.7.3
│   │   └── gson.jar            # Google Gson 2.11.0
│   └── src/com/helpdesk/socket/
│       ├── Main.java           # Entry point, ServerSocket e pool de threads
│       ├── config/
│       │   └── Database.java   # Gerenciador de conexão e pool JDBC (Neon DB / PostgreSQL)
│       └── util/
│           ├── AuthUtil.java   # Validação criptográfica de token HMAC-SHA256 (CT19)
│           ├── JsonUtil.java   # Serialização e parsing JSON estruturado (Google Gson)
│           └── WebSocketUtil.java # Handshake RFC 6455, decodificação de frames e Health Check
├── docs/
│   ├── adr/
│   │   ├── 0001-servidor.md    # Registro de decisão de concorrência e descarte de chat
│   │   └── 0002-notificacao.md # Registro de decisão de notificações push
│   ├── fluxos/                 # Contratos CT18, CT19, CT22 e entregáveis E1 a E9
│   │   ├── CT18-publicarEvento.md
│   │   ├── CT19-autenticarConexao.md
│   │   ├── CT22-chamadoAtualizado.md
│   │   ├── visao-geral.md
│   │   └── README.md
│   └── escopo.pdf              # Especificação oficial de requisitos (revisada)
├── .gitignore
└── README.md                   # Esta documentação especializada
```

---

## Como Executar Localmente

### Pré-requisitos
- **Java JDK 21** instalado.
- Acesso à internet para conexão com o banco de dados Neon (ou PostgreSQL 15 local).

### Variáveis de Ambiente

O servidor pode ser configurado via variáveis de ambiente padrão ou utilizará fallbacks seguros para desenvolvimento:

| Variável | Padrão | Descrição |
|:---|:---|:---|
| `PORT` ou `WS_PORT` | `5000` | Porta pública para o handshake WebSocket e Health Check HTTP. |
| `INTERNAL_PORT` | `5001` | Porta TCP interna para comunicação exclusiva com o Backend API. |
| `INTERNAL_SECRET` | `secret-dev` | Segredo compartilhado com a API para autenticação de eventos (CT18). |
| `JWT_SECRET` | `jwt-secret` | Chave HMAC compartilhada para validação de tokens de usuários (CT19). |
| `DATABASE_URL` / `DB_URL` | *Neon DB URL* | URL completa de conexão JDBC PostgreSQL (`jdbc:postgresql://...`). |
| `DB_HOST` | `localhost` | Host do banco PostgreSQL (utilizado caso `DATABASE_URL` não esteja definida). |
| `DB_PORT` | `5432` | Porta do banco PostgreSQL. |
| `DB_NAME` | `helpdesk` | Nome do banco de dados relacional. |
| `DB_USER` | `postgres` | Usuário de autenticação no banco. |
| `DB_PASSWORD` | *(vazio)* | Senha de autenticação no banco. |

### Compilação e Execução Manual

A partir do diretório raiz deste repositório:

```bash
cd backend-socket

# 1. Compilar em Java Puro (sem Maven/Gradle)
mkdir -p out
javac -cp "lib/*" -d out $(find src -name "*.java")

# 2. Executar o servidor
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
2. Sincroniza via `rsync` os arquivos compiláveis e de configuração para a pasta correspondente no monorepo.
3. Cria um commit automatizado com a mensagem `chore(backend-socket): sync code from backend-socket repo (<SHA>)`.

Isso permite que o desenvolvimento do módulo de socket ocorra de forma desacoplada sem comprometer a integridade do monorepo principal.
