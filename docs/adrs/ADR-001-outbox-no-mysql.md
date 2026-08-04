# ADR-001: Emissão de eventos de mudança de status via padrão Outbox no MySQL

**Status:** Aceito
**Data:** 2026-07-29
**ADRs Relacionados:** ADR-002, ADR-006, ADR-007, ADR-008

## Contexto e Declaração do Problema

Três clientes B2B pediram para ser notificados quando o status dos pedidos deles muda. Hoje eles descobrem a mudança fazendo consultas repetidas à API de pedidos, o que torna a integração lenta e cara do lado deles. O sistema atual não tem nenhum mecanismo de notificação externa, evento ou fila.

O pedido tem urgência de negócio, e não apenas mérito técnico: um desses clientes sinalizou a possibilidade de migrar para um concorrente caso a entrega não saia no prazo combinado. Isso define o problema como prioridade, e não como melhoria oportunista.

A primeira pergunta de arquitetura foi levantada por Larissa em `[09:03]`: disparar a notificação de forma síncrona no serviço de pedidos, no momento em que o status muda, ou registrar o evento em algum lugar e entregar depois.

Bruno argumentou contra o disparo síncrono em `[09:04]`. A transação de mudança de status já é pesada: atualiza o pedido, insere o histórico de status e decrementa o estoque dos produtos. Acrescentar uma chamada HTTP no meio disso faz com que qualquer cliente lento trave a mudança de status de outros pedidos.

No mesmo minuto, ele apontou a consequência mais grave: se o cliente estiver fora do ar, a única saída seria desfazer uma operação de negócio legítima, o que não é aceitável.

O problema por baixo dessa discussão é o *dual write*: gravar o estado de negócio no banco e notificar um sistema externo são duas escritas em recursos diferentes, sem transação comum. Se a gravação no banco tem sucesso e a notificação falha, o cliente nunca fica sabendo da mudança. Se a notificação tem sucesso e a gravação é desfeita, o cliente fica sabendo de algo que não aconteceu.

Diego nomeou o padrão que resolve isso em `[09:06]`: outbox.

## Fatores de Decisão

- A transação de mudança de status está no caminho crítico da operação e já executa três escritas.
- A disponibilidade do cliente não pode interferir na capacidade da plataforma de mudar o status de um pedido.
- O time é pequeno e não tem apetite para operar infraestrutura nova.
- A meta de latência percebida pelo cliente é abaixo de 10 segundos, o que é folgado.
- Não pode existir janela em que o status mudou e o evento não existe.
- O registro de eventos vai ser lido em ciclo curto e precisa ser consultável por estado e por ordem de criação.

## Alternativas Consideradas

1. Tabela de outbox no MySQL já existente, com inserção na mesma transação da mudança de status.
2. Disparo HTTP síncrono dentro da transação de mudança de status.
3. Redis Streams ou outro broker dedicado como fila intermediária.

## Resultado da Decisão

Opção escolhida: **tabela de outbox no MySQL existente**, com a inserção do evento dentro da mesma transação SQL que atualiza o pedido e grava o histórico de status. Larissa fechou a decisão em `[09:08]`, depois que Diego apresentou a mecânica em `[09:06]` e descartou o broker dedicado como overengineering para o tamanho do time em `[09:07]`.

A garantia que essa escolha compra é direta: se a transação principal fez commit, o evento foi registrado; se ela foi desfeita, o evento desaparece junto. Não existe estado intermediário observável. A entrega em si passa a ser responsabilidade de um processo separado, tratada no ADR-002.

O desenho da tabela ficou definido no mesmo minuto do fechamento: índice no campo de estado do evento, com os valores pendente, processando, falhou e entregue, e índice na data de criação. O worker lê apenas os pendentes, em lote pequeno, processa e marca. O arquivamento das linhas já entregues, estimado em torno de 30 dias, foi explicitamente colocado fora do escopo desta feature.

Vale delimitar o alcance desta decisão: ela resolve a emissão do evento, não a entrega. Quem lê a tabela, com que frequência e com que garantia de ordenação é assunto do ADR-002. O que a tabela armazena em cada linha está no ADR-007, e quais transições chegam a virar linha está no ADR-008.

## Prós e Contras das Alternativas

### Tabela de outbox no MySQL existente

- Boa, porque reduz uma escrita em dois recursos a uma única transação local, eliminando a janela de inconsistência do dual write.
- Boa, porque usa infraestrutura que já está em produção, sem novo componente para operar, monitorar e recuperar.
- Boa, porque a tabela é registro durável do que foi emitido, o que serve para auditoria, depuração e reprocessamento.
- Ruim, porque acrescenta carga de escrita ao mesmo banco que atende a operação do sistema de pedidos.
- Ruim, porque a latência de entrega passa a depender da cadência de leitura do worker, não do instante do commit.
- Ruim, porque a entrega deixa de ser resolvida aqui e passa a exigir um segundo componente, com custo operacional próprio, que o ADR-002 assume.

### Disparo HTTP síncrono dentro da transação

- Boa, porque entrega com a menor latência possível, no próprio instante do commit.
- Boa, porque não exige tabela nova nem processo novo: tudo acontece dentro do fluxo que já existe.
- Ruim, porque acopla a duração da transação de mudança de status ao tempo de resposta de um sistema de terceiro.
- Ruim, porque mantém locks de banco abertos durante uma chamada de rede.
- Ruim, porque um cliente indisponível obriga a escolher entre desfazer uma operação de negócio válida ou perder o evento.

### Redis Streams ou broker dedicado

- Boa, porque é infraestrutura feita para o problema, com grupos de consumo e escala horizontal nativos.
- Ruim, porque reintroduz exatamente o dual write que se quer evitar: gravar no MySQL e publicar no Redis continuam sendo duas transações independentes.
- Ruim, porque adiciona um componente novo ao ambiente, com custo de operação que o time não tem tamanho para absorver.
- Ruim, porque a atomicidade só voltaria com lógica de compensação entre commit e publicação, trabalho adicional de código para o mesmo time.

## Consequências

- Boa, porque a atomicidade entre mudança de status e registro do evento passa a ser garantida pelo próprio banco, sem código de compensação.
- Boa, porque a tabela de outbox funciona como histórico consultável do que a plataforma emitiu, o que a torna base natural para o histórico de entregas pedido pelo produto.
- Boa, porque não há custo de infraestrutura nova nem novo ponto de falha operacional.
- Boa, porque a modelagem da tabela nova cabe inteiramente nas convenções que o projeto já pratica, o que mantém a mudança de schema previsível.
- Ruim, porque o banco do sistema de pedidos absorve todo o volume de eventos, em escrita e em leitura repetida pelo worker, o que faz do dimensionamento desse banco uma preocupação nova.
- Ruim, porque o crescimento da tabela exige política de retenção, e essa política ficou fora do escopo desta entrega, o que deixa uma pendência operacional conhecida.
- Ruim, porque a mudança de status passa a poder falhar por um motivo que não é dela: erro ao gravar o evento desfaz a transação inteira, que é o comportamento pedido e ao mesmo tempo uma dependência nova no caminho crítico.
- Neutra, porque o padrão é amplamente documentado e adotado no mercado, o que reduz o custo de onboarding, mas também significa herdar suas limitações conhecidas: latência ligada à cadência de leitura e ordenação garantida apenas com worker único, ambas tratadas no ADR-002.

## Referências

- `src/modules/orders/order.service.ts:131` transação que passará a carregar a inserção do evento
- `src/modules/orders/order.service.ts:158` atualização do status do pedido dentro da transação, imediatamente antes do ponto de inserção do evento
- `prisma/schema.prisma:74` convenções de modelagem a seguir na nova tabela
- `prisma/schema.prisma:94` índice por data de criação já usado no projeto, padrão que a nova tabela repete
- `TRANSCRICAO.md` `[09:00]` Marcos: dor atual de consulta periódica e risco de perda de cliente
- `TRANSCRICAO.md` `[09:02]` Marcos: meta de latência percebida abaixo de 10 segundos
- `TRANSCRICAO.md` `[09:03]` Larissa: coloca a escolha entre disparo síncrono e registro para entrega posterior
- `TRANSCRICAO.md` `[09:04]` Bruno: peso da transação e recusa de desfazer a mudança de status
- `TRANSCRICAO.md` `[09:06]` Diego: nomeia o padrão outbox e descreve a mecânica
- `TRANSCRICAO.md` `[09:07]` Larissa e Diego: broker dedicado levantado e descartado como overengineering
- `TRANSCRICAO.md` `[09:08]` Diego e Larissa: desenho da tabela, arquivamento fora de escopo e fechamento da decisão
- `TRANSCRICAO.md` `[09:34]` Marcos: histórico de entregas apoiado no registro da outbox
- `TRANSCRICAO.md` `[09:40]` Bruno e `[09:41]` Diego: exigência de atomicidade entre status e evento
- [Pattern: Transactional outbox, microservices.io](https://microservices.io/patterns/data/transactional-outbox.html)
- [Transactional outbox pattern, AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)
