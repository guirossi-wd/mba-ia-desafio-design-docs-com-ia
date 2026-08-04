# ADR-008: Redução de volume da outbox com filtro de eventos aplicado na inserção

**Status:** Aceito
**Data:** 2026-07-29
**ADRs Relacionados:** ADR-001, ADR-002, ADR-007

## Contexto e Declaração do Problema

Cada endpoint de webhook assina uma lista de status que quer receber. Marcos especificou o requisito em `[09:33]` com um exemplo prático: um cliente pode querer ser avisado somente quando o pedido vira enviado e entregue, ignorando as transições anteriores.

Isso significa que boa parte das transições de status não interessa a ninguém. O ciclo de vida de um pedido no sistema admite no máximo quatro transições sucessivas até o estado terminal de entrega, e um cliente que assina apenas duas delas torna as outras irrelevantes.

O conjunto de valores que uma assinatura pode conter não é arbitrário: ele vem da máquina de estados de pedido que já existe no código, com seis estados e transições fixas. O filtro opera sobre esse domínio fechado, e não sobre uma lista livre.

Existem dois momentos possíveis para aplicar esse filtro, e Diego formulou a escolha em `[09:34]`: filtrar na inserção na outbox, ou filtrar no momento do envio.

Filtrar no envio significa registrar toda transição de todo pedido e o worker descartar o que nenhum endpoint quer.

Filtrar na inserção significa consultar as assinaturas antes de gravar, e não gravar nada se ninguém quiser aquela transição.

A configuração do endpoint tem ainda um estado ativo, o que amplia a pergunta: o filtro precisa considerar quais assinaturas estão valendo, e não apenas quais status foram escolhidos.

A decisão importa porque a inserção acontece dentro da transação de mudança de status, que o ADR-001 já identificou como caminho crítico, e porque a outbox é lida a cada 2 segundos pelo worker do ADR-002. Volume desnecessário custa nos dois pontos.

## Fatores de Decisão

- O filtro por status é requisito de produto explícito.
- A tabela de outbox é lida em ciclo curto pelo worker, então seu tamanho afeta o custo contínuo de operação.
- A política de retenção da outbox ficou fora do escopo desta entrega, o que torna o crescimento da tabela uma preocupação real.
- A inserção acontece dentro da transação de mudança de status, e tudo que é feito ali soma no caminho crítico.
- O conteúdo do evento é um snapshot montado na inserção, conforme ADR-007, e montar snapshot que ninguém vai receber é trabalho jogado fora.
- A assinatura tem estado ativo, então o filtro depende de dados que mudam ao longo do tempo.

## Alternativas Consideradas

1. Filtrar na inserção: consultar as assinaturas ativas e gravar na outbox apenas se algum endpoint quiser aquela transição.
2. Filtrar no envio: gravar toda transição e o worker descartar as que nenhum endpoint assina.

## Resultado da Decisão

Opção escolhida: **filtrar na inserção**. Bruno decidiu em `[09:34]`, com a justificativa de que se nenhum webhook do cliente quer aquele status, não vale nem inserir, economizando linha na tabela. Diego, que havia levantado a alternativa no mesmo minuto, concordou em seguida.

A consequência de desenho é que toda linha que existe na outbox tem destinatário certo. A tabela deixa de ser um registro de tudo que aconteceu e passa a ser um registro do que precisa ser entregue.

Isso se encaixa com o ADR-007: como o conteúdo é montado na inserção, filtrar antes evita montar snapshot descartável. E se encaixa com o ADR-001: como a inserção participa da transação, a consulta às assinaturas acontece com a mesma transação, lendo dados consistentes com o instante da mudança.

A escolha fixa também o momento em que a assinatura é avaliada. Como a configuração tem estado ativo e lista de status editáveis, é a configuração vigente no instante da mudança de status que determina se o evento existe. Não há reavaliação depois, porque depois já não existe linha para reavaliar.

## Prós e Contras das Alternativas

### Filtrar na inserção

- Boa, porque a outbox cresce proporcionalmente ao que precisa ser entregue, e não ao volume total de transições da plataforma.
- Boa, porque o ciclo de leitura do worker a cada 2 segundos varre uma tabela menor, mantendo o custo de operação baixo e previsível.
- Boa, porque evita montar snapshot que seria descartado, poupando trabalho dentro da transação.
- Boa, porque toda linha na outbox tem destinatário, o que torna a interpretação da tabela e das métricas direta: nada ali é ruído.
- Ruim, porque acrescenta uma consulta às assinaturas dentro da transação de mudança de status, que é caminho crítico da operação.
- Ruim, porque a plataforma deixa de ter registro das transições que ninguém assinava. Se um cliente cadastrar um endpoint depois, não existe histórico para reenviar, e nem existe forma de auditar o que teria sido enviado.

### Filtrar no envio

- Boa, porque a transação de mudança de status não precisa consultar assinaturas, ficando mais simples e mais rápida.
- Boa, porque a outbox se torna registro completo de todas as transições, útil para auditoria e para eventual reenvio a endpoint cadastrado depois.
- Boa, porque mudança na assinatura de um cliente passaria a valer para eventos já gravados e ainda não entregues.
- Ruim, porque a tabela cresce com o volume total de pedidos da plataforma, e não com o volume dos clientes que assinaram webhook.
- Ruim, porque o worker gasta ciclo lendo e descartando linhas irrelevantes a cada 2 segundos, custo permanente que aumenta com o tempo.
- Ruim, porque combinado com o ADR-007 exigiria montar snapshot de eventos que nunca serão enviados.

## Consequências

- Boa, porque o custo de armazenamento e de leitura da outbox fica atrelado à adoção real da feature.
- Boa, porque a profundidade da fila passa a ser métrica limpa de saúde da entrega, sem precisar descontar linhas que nunca seriam enviadas.
- Boa, porque a decisão reduz o risco associado à ausência de política de retenção, que o ADR-001 deixou como pendência conhecida.
- Boa, porque o worker fica livre de qualquer regra de destinatário: se a linha existe, ela vai ser entregue.
- Ruim, porque a transação de mudança de status ganha uma consulta adicional, e essa consulta acontece em toda mudança de status de todo pedido, inclusive dos pedidos de clientes que não têm webhook nenhum.
- Ruim, porque a plataforma perde a capacidade de reconstruir o que teria sido enviado antes de um endpoint existir. Um cliente que cadastre webhook hoje não tem como receber o histórico de ontem, porque aquelas linhas nunca foram gravadas.
- Ruim, porque cria uma dependência temporal sutil: a assinatura vigente no instante da mudança de status é a que vale. Se o cliente alterar a lista de status assinados, a mudança só afeta transições futuras, e eventos já na fila continuam sendo entregues conforme a assinatura antiga.
- Ruim, porque uma falha na consulta de assinaturas passa a ser capaz de desfazer a mudança de status, pela atomicidade do ADR-001, o que estende ao domínio de configuração de webhook a responsabilidade sobre a operação de pedidos.
- Neutra, porque a escolha desloca custo do processamento contínuo para o caminho crítico. É uma troca deliberada: uma consulta a mais por mudança de status, em vez de leitura permanente de linhas inúteis a cada 2 segundos.

## Referências

- `src/modules/orders/order.service.ts:131` transação onde a consulta às assinaturas será feita antes da inserção
- `src/modules/orders/order.service.ts:132` leitura de dados dentro da transação, padrão que a consulta às assinaturas segue
- `src/modules/orders/order.status.ts:3` matriz de transições que define os status assináveis e o comprimento máximo do ciclo
- `prisma/schema.prisma:16` enumeração de status do pedido, que é o domínio de valores do filtro
- `TRANSCRICAO.md` `[09:08]` Diego: arquivamento da outbox fora do escopo desta feature
- `TRANSCRICAO.md` `[09:21]` Bruno e Sofia: configuração do endpoint com estado ativo
- `TRANSCRICAO.md` `[09:33]` Marcos: filtro como lista de status que o endpoint quer ouvir
- `TRANSCRICAO.md` `[09:34]` Diego e Bruno: escolha entre filtrar na inserção ou no envio e decisão pela inserção
