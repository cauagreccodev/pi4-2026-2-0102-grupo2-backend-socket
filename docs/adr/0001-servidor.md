# 1. Estruturas de dados do servidor

## Status

Aceito

## Contexto

O servidor de socket do Help Desk gerencia comunicação em tempo real (chat, fila de espera e notificações) sob alta concorrência. O sistema precisa:

- Mapear sessões ativas por usuário (suportando múltiplas abas) e salas de atendimento por chamado.
- Distribuir chamados prioritários para técnicos disponíveis.
- Enviar mensagens sem que clientes lentos ou oscilações de rede travem os demais.
- Controlar estouro de prazos de SLA.
- Persistir histórico no PostgreSQL sem que a latência de banco degrade o envio em tempo real.

Todas as conexões rodam concorrentemente e demandam estruturas *thread-safe* na memória, sem locks globais que criem gargalos.

## Decisão

Usaremos coleções e filas concorrentes da biblioteca padrão do Java (`java.util.concurrent`):

- **`ConcurrentHashMap` com conjuntos concorrentes**: armazena os chamados ativos, as salas de chat (`ticketId` → conexões dos participantes) e as conexões de cada usuário (`userId` → conexões ativas), permitindo múltiplas abas e broadcast em O(1). As sessões mantêm referências às suas salas para limpeza imediata na desconexão.
- **`PriorityBlockingQueue`**: fila de chamados pendentes, ordenada por criticidade e tempo de espera, garantindo entrega primeiro ao chamado mais urgente.
- **`ConcurrentLinkedQueue`**: fila de técnicos disponíveis para recebimento e despacho rápido de novos atendimentos.
- **`LinkedBlockingQueue` (com limite por cliente)**: buffer de saída (*outbox*) exclusivo de cada conexão. Nenhuma thread escreve direto no socket de outra; clientes que acumularem mensagens acima do limite são desconectados.
- **`LinkedBlockingQueue` (buffer de persistência)**: fila assíncrona de eventos consumida por workers dedicados ao PostgreSQL, desacoplando o I/O do banco do fluxo do chat.
- **`DelayQueue`**: monitoramento de prazos de SLA de chamados, acordando uma única thread apenas no vencimento do próximo prazo, sem timers individuais.
- **`ConcurrentLinkedDeque` (limitado por sala)**: buffer das últimas mensagens do chat em memória para histórico imediato ao técnico ou reconexão do cliente.

O acesso a essas estruturas fica centralizado em uma classe única de estado do servidor.

## Consequências

- Concorrência de alto desempenho usando apenas o Java nativo, sem dependências externas.
- O fluxo de mensagens em tempo real fica totalmente isolado de clientes lentos e da latência do banco de dados.
- Limpeza em O(1) e capacidade limitada nas filas evitam vazamentos e estouro de memória.
- Operações que alteram mais de uma estrutura exigem métodos coordenados na classe de estado.
- Estado primário volátil em memória; reinicializações demandam recarga a partir do PostgreSQL.
- Válido para arquitetura de nó único; clustering exigirá broker externo (nova ADR).

