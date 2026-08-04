# ADR-007: Preservação do estado do pedido no momento do evento via snapshot do conteúdo na inserção

**Status:** Aceito
**Data:** 2026-07-29
**ADRs Relacionados:** ADR-001, ADR-002, ADR-003, ADR-004, ADR-005, ADR-008

## Contexto e Declaração do Problema

O ADR-001 estabelece que o evento é registrado na outbox dentro da transação que muda o status do pedido, e o ADR-002 estabelece que a entrega acontece depois, em outro processo. Existe portanto um intervalo obrigatório entre o instante em que o fato ocorre e o instante em que ele é comunicado.

O ADR-003 amplia bastante esse intervalo. Com a política de retry, um evento pode ser entregue até quase 15 horas depois do fato. Nesse período o pedido pode ter mudado de status várias vezes.

Isso obriga a responder o que exatamente a outbox armazena. Bruno colocou a pergunta em `[09:51]`, no fim da reunião: a linha da outbox guarda o conteúdo do evento já montado, ou guarda apenas a referência ao pedido e monta o conteúdo no momento do envio.

A diferença não é de implementação, é de semântica. Guardar apenas a referência significa que o conteúdo entregue reflete o estado do pedido no momento do envio. Guardar o conteúdo montado significa que ele reflete o estado no momento em que o status mudou.

Um exemplo concreto do problema: um pedido passa de pago para em processamento às 10h e a entrega falha; o cliente volta às 22h; nesse meio tempo o pedido foi para enviado.

Se o conteúdo for montado no envio, o cliente recebe um evento anunciando a transição para em processamento, mas com os dados do pedido dizendo enviado. O evento fica internamente contraditório.

Há uma restrição de tamanho que depende desta escolha. A reunião fixou um teto de 64KB para o conteúdo do evento, com recusa em vez de truncamento quando ele é ultrapassado. Persistir o conteúdo faz esse teto incidir sobre o que fica guardado, e não só sobre o que passa pela rede.

## Fatores de Decisão

- O intervalo entre o fato e a entrega é obrigatório por desenho e pode chegar a quase 15 horas com o retry do ADR-003.
- Um evento descreve um fato que aconteceu em um instante determinado, e precisa ser coerente com aquele instante.
- O cliente usa o evento para reagir a uma transição específica, não para sincronizar o estado atual do pedido.
- O conteúdo do evento é enxuto por decisão de produto, com os itens do pedido excluídos para não inflar a mensagem.
- Existe um caminho já previsto para o cliente obter o estado atual: consultar a API de pedidos.
- O conteúdo tem teto de 64KB, com recusa em vez de truncamento.

## Alternativas Consideradas

1. Snapshot: o conteúdo do evento é montado e persistido no momento da inserção na outbox.
2. Referência: a outbox guarda apenas a identificação do pedido e da transição, e o conteúdo é montado no momento do envio.

## Resultado da Decisão

Opção escolhida: **snapshot na inserção**. Larissa decidiu em `[09:52]`, com a justificativa de que se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou, e que a alternativa produz caso esquisito. Diego concordou no mesmo minuto e Bruno confirmou o fechamento.

A consequência semântica é que a linha da outbox deixa de ser uma tarefa a executar e passa a ser um fato registrado. Uma vez inserida, ela não depende de mais nenhuma consulta ao banco para ser entregue, e seu conteúdo é imutável.

Isso se combina de forma direta com duas decisões anteriores. Com o ADR-001, porque o snapshot é montado dentro da mesma transação, usando dados que já estão em memória naquele ponto. Com o ADR-005, porque o identificador de evento e o conteúdo passam a ser fixos desde a inserção, o que faz todas as tentativas de entrega do mesmo evento serem byte a byte idênticas. Isso importa para a assinatura do ADR-004: a assinatura é calculada sobre um conteúdo que não muda entre tentativas.

O conteúdo do snapshot foi especificado por Diego em `[09:43]`: identificador do evento, tipo do evento, horário, identificação do pedido, número do pedido, status de origem e destino, identificação do cliente e valores básicos do pedido. Os itens do pedido ficam de fora, e o cliente que precisar de detalhe consulta a API de pedidos.

A decisão também posiciona o teto de tamanho: como o conteúdo nasce montado, o limite de 64KB é avaliado sobre o que vai ser gravado, no mesmo ponto em que o evento é criado. Um conteúdo acima do teto deixa de ser um problema descoberto na hora de enviar e passa a ser um problema descoberto na hora de registrar.

## Prós e Contras das Alternativas

### Snapshot na inserção

- Boa, porque o evento entregue é sempre coerente com o fato que ele anuncia, independente de quanto tempo passou.
- Boa, porque a entrega não depende de consultar o pedido novamente, o que reduz o trabalho do worker a ler a outbox e enviar.
- Boa, porque todas as tentativas de um mesmo evento carregam conteúdo idêntico, o que dá sentido à deduplicação por identificador do ADR-005 e mantém a assinatura estável.
- Boa, porque a outbox passa a ser registro histórico do que foi efetivamente comunicado, o que serve para responder a contestação de cliente e alimenta o histórico de entregas.
- Boa, porque o evento continua entregável mesmo que o pedido seja alterado ou removido depois.
- Ruim, porque duplica dados que já existem na tabela de pedidos, aumentando o volume armazenado.
- Ruim, porque um defeito na montagem do conteúdo fica persistido: corrigir eventos já inseridos exige intervenção de dados, não apenas um deploy.

### Referência com montagem no envio

- Boa, porque a outbox fica mínima, guardando apenas identificação e transição.
- Boa, porque uma correção no formato do conteúdo passa a valer imediatamente para todos os eventos ainda não entregues.
- Ruim, porque o conteúdo entregue pode contradizer a transição que o evento anuncia, que é o caso esquisito apontado na reunião.
- Ruim, porque cada entrega e cada nova tentativa exigem consulta ao banco, aumentando a carga do worker justamente durante um período de retry.
- Ruim, porque o conteúdo variando entre tentativas quebra a premissa de deduplicação: o cliente veria o mesmo identificador de evento com dados diferentes.
- Ruim, porque um pedido removido tornaria o evento inentregável, sem que o fato original tenha deixado de ser verdadeiro.

## Consequências

- Boa, porque o contrato com o cliente ganha uma propriedade clara e explicável: o evento descreve o pedido como ele estava quando a transição aconteceu.
- Boa, porque a coerência do evento fica independente do tempo de retry, o que remove a pressão para encurtar a janela do ADR-003 por motivo de consistência.
- Boa, porque a outbox se torna registro auditável do que foi comunicado, e não apenas do que deveria ser comunicado.
- Boa, porque o worker fica mais simples: ler, assinar e enviar, sem nenhuma regra de montagem de conteúdo no caminho da entrega.
- Ruim, porque o mesmo dado passa a existir em dois lugares, e o custo de armazenamento da outbox cresce mais rápido, o que agrava a pendência de política de retenção deixada em aberto no ADR-001.
- Ruim, porque mudança no formato do conteúdo não se aplica retroativamente a eventos já na fila, o que significa que um deploy pode gerar dois formatos em trânsito ao mesmo tempo.
- Ruim, porque a verificação do teto de 64KB passa a acontecer dentro da transação de mudança de status, ou seja, no caminho crítico da operação de pedido, e não em um processo de fundo.
- Neutra, porque o cliente recebe um conteúdo deliberadamente enxuto e, para obter o estado atual do pedido, precisa consultar a API. O evento sinaliza a mudança, não substitui a consulta.

## Referências

- `src/modules/orders/order.service.ts:132` leitura do pedido dentro da transação, cujos dados alimentam o snapshot
- `src/modules/orders/order.service.ts:158` ponto da transação em que a transição já está determinada
- `prisma/schema.prisma:46` coluna de conteúdo estruturado já usada no projeto, precedente de modelagem para o snapshot
- `prisma/schema.prisma:125` remoção do pedido em cascata sobre os registros dependentes, cenário em que o snapshot mantém o evento entregável
- `TRANSCRICAO.md` `[09:23]` Sofia: recusar em vez de truncar conteúdo acima do limite
- `TRANSCRICAO.md` `[09:24]` Diego e Larissa: teto de 64KB fixado como requisito não funcional
- `TRANSCRICAO.md` `[09:43]` Diego: campos do conteúdo do evento e exclusão dos itens do pedido
- `TRANSCRICAO.md` `[09:51]` Bruno: pergunta entre conteúdo montado na inserção e montado no envio
- `TRANSCRICAO.md` `[09:52]` Larissa, Diego e Bruno: decisão pelo snapshot e confirmação
