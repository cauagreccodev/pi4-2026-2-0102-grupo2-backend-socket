# 1. Estruturas de dados do servidor

## Status

Aceito

## Contexto

O servidor WebSocket do help desk precisa responder rapidamente a quatro perguntas:

- Onde está a conexão de um usuário?
- Qual é o próximo chamado a ser atendido?
- Como enviar mensagens sem que um cliente lento trave os outros?
- Quais chamados estouraram o prazo de atendimento (SLA)?

Cada conexão roda em sua própria thread, então todas as estruturas são acessadas ao mesmo tempo e precisam ser seguras para concorrência.

## Decisão

Usaremos HashMaps e filas concorrentes do próprio Java:

- **`ConcurrentHashMap`** guarda os chamados, as conexões de cada usuário e quem acompanha cada chamado. A busca é em O(1) e não há um lock global.
- **`PriorityBlockingQueue`** é a fila de chamados em espera, ordenada por prioridade e depois por chegada. O atendente sempre recebe o chamado mais urgente.
- **`LinkedBlockingQueue`**, com limite, é a fila de saída de cada cliente. Ninguém escreve direto no socket. Se a fila encher, o cliente é desconectado.
- **`DelayQueue`** controla os prazos de SLA. Uma única thread aguarda o próximo prazo vencer e escala o chamado, sem precisar de um timer por chamado.

O acesso a essas estruturas fica centralizado em uma única classe de estado.

## Consequências

- Concorrência segura e rápida, sem bibliotecas externas.
- Um cliente lento não afeta os demais.
- Os dados ficam só em memória e se perdem se o servidor reiniciar. A persistência fica com outra camada.
- Atualizações que envolvem mais de uma estrutura não são atômicas e exigem cuidado na classe de estado.
- Funciona para um único servidor. Escalar para vários exigirá uma nova ADR.
