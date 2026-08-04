# ADR-005: Garantia de entrega at-least-once com deduplicação pelo cliente via identificador de evento

**Status:** Aceito
**Data:** 2026-07-29
**ADRs Relacionados:** ADR-003, ADR-004, ADR-007

## Contexto e Declaração do Problema

A política de retry do ADR-003 cria uma situação inevitável: a plataforma vai reenviar eventos cuja entrega ela não conseguiu confirmar. E não conseguir confirmar não é o mesmo que não ter entregue.

O caso concreto é o tempo esgotado na resposta. O worker envia o evento, o cliente recebe e processa com sucesso, mas a resposta se perde ou demora mais do que o limite de espera.

Do ponto de vista do worker, aquilo foi falha e entra na fila de retry. Do ponto de vista do cliente, o evento chegou e foi processado. Na próxima tentativa, ele recebe o mesmo evento de novo.

A janela em que isso acontece é larga. Com a progressão de backoff do ADR-003, a repetição pode chegar quase 15 horas depois da primeira entrega, e não apenas segundos depois. Isso muda a natureza do problema: não basta o cliente ignorar duas requisições idênticas em sequência, ele precisa de um critério estável para reconhecer um evento que já viu.

Eliminar esse cenário exigiria que as duas pontas coordenassem o resultado do processamento, e não apenas o resultado do transporte. Diego trouxe o tema em `[09:24]` já com a conclusão: a plataforma vai garantir at-least-once, e o cliente precisa estar preparado para receber o mesmo evento duas vezes.

Bruno fez a pergunta prática em `[09:25]`: como o cliente distingue um evento novo de uma repetição do mesmo evento. Sem um identificador estável, duas entregas do mesmo fato são indistinguíveis de dois fatos idênticos, e o cliente não tem como decidir.

Sofia levantou a objeção legítima no mesmo minuto: essa escolha transfere responsabilidade para o cliente.

## Fatores de Decisão

- O retry decidido no ADR-003 torna a duplicidade uma consequência estrutural, não um defeito a corrigir.
- Tempo esgotado na resposta é ambíguo por natureza: o worker não tem como saber se o cliente processou.
- A repetição pode chegar horas depois da primeira tentativa, por causa da progressão de backoff do ADR-003.
- Garantia de entrega única exigiria coordenação entre as duas pontas, com custo considerado desproporcional.
- Existe padrão de mercado consolidado para esse problema, com plataformas citadas nominalmente na reunião.
- A responsabilidade transferida ao cliente precisa vir acompanhada de documentação.

## Alternativas Consideradas

1. Entrega at-least-once, com identificador único por evento em cabeçalho e deduplicação no cliente.
2. Entrega exactly-once, com coordenação entre plataforma e cliente para confirmar processamento.

## Resultado da Decisão

Opção escolhida: **entrega at-least-once, com identificador único por evento enviado em cabeçalho, e deduplicação de responsabilidade do cliente**. Diego propôs em `[09:24]` e `[09:25]`, e Larissa registrou como decisão em `[09:26]`.

O identificador é gerado no momento em que o evento entra na outbox, é único por evento e permanece o mesmo em todas as tentativas de entrega daquele evento. É isso que o torna utilizável como chave de deduplicação: o cliente registra os identificadores que já processou e descarta repetição.

Diego respondeu à objeção de Sofia em `[09:25]` com dois argumentos. O primeiro é que se trata do comportamento padrão do mercado, citando nominalmente Stripe e GitHub como plataformas que operam assim. O segundo é de custo: garantir entrega única exigiria coordenação das duas pontas e ficaria muito mais complexo, enquanto at-least-once com identificador resolve a quase totalidade dos casos.

Marcos fechou a lacuna de comunicação em `[09:26]`, assumindo documentar o comportamento de forma destacada no portal do desenvolvedor, para que a expectativa de duplicidade seja explícita antes de o cliente integrar.

A escolha do formato do identificador acompanha a convenção já adotada no projeto para chaves primárias. A geração não exige dependência nova, porque a biblioteca que o projeto já usa para identificar requisição atende ao caso.

## Prós e Contras das Alternativas

### At-least-once com identificador de evento

- Boa, porque nenhum evento é perdido por falha de transporte ou por resposta ambígua.
- Boa, porque a plataforma não precisa manter estado sobre o processamento interno do cliente, apenas sobre o transporte.
- Boa, porque é o comportamento que o mercado consolidou, conforme os exemplos citados na reunião.
- Boa, porque o mesmo identificador serve como chave de correlação entre o log da plataforma e o log do cliente na investigação de incidente.
- Ruim, porque a correção do resultado final depende de o cliente implementar deduplicação, e a plataforma não tem como verificar se ele fez isso.
- Ruim, porque cliente que não deduplica pode processar o mesmo evento mais de uma vez, e o efeito colateral disso ocorre no domínio dele, fora do alcance da plataforma.

### Exactly-once com coordenação

- Boa, porque elimina a duplicidade na origem e dispensa qualquer tratamento do lado do cliente.
- Ruim, porque exige protocolo de confirmação entre as duas pontas, com estado transacional distribuído em sistemas de organizações diferentes.
- Ruim, porque o cliente teria que implementar o lado dele do protocolo corretamente para a garantia valer, o que troca uma exigência simples por uma bem mais difícil.
- Ruim, porque o custo é desproporcional ao benefício.

## Consequências

- Boa, porque a combinação de retry com identificador estável entrega confiabilidade real sem estado compartilhado entre as organizações.
- Boa, porque o contrato com o cliente fica explícito e verificável: sempre chega, pode chegar mais de uma vez, e sempre com o mesmo identificador.
- Boa, porque o identificador estável permite agrupar todas as tentativas de um mesmo evento no histórico de entregas, o que torna objetiva a investigação de reclamação de cliente.
- Ruim, porque a plataforma passa a ter uma obrigação de documentação, e não apenas de código. Se a duplicidade não estiver clara no material de integração, o cliente descobre em produção.
- Ruim, porque incidente causado por reprocessamento no cliente vai chegar como reclamação para a plataforma, mesmo sendo comportamento documentado, o que gera custo de suporte previsível.
- Ruim, porque a plataforma não consegue distinguir, na sua própria observação, entrega repetida de entrega efetivamente nova para o cliente: o registro mostra o que foi enviado, não o que o outro lado considerou novo.
- Neutra, porque a responsabilidade de deduplicar fica com o cliente, que foi exatamente o ponto questionado pela engenharia de segurança na reunião. A escolha foi consciente e alinhada ao que o mercado pratica, mas continua sendo transferência de trabalho para fora.
- Neutra, porque o identificador de evento passa a ser dado de contrato público, e não detalhe interno. Mudar sua forma ou seu ciclo de vida depois da publicação quebra a deduplicação do cliente.

## Referências

- `src/middlewares/request-logger.middleware.ts:6` padrão já existente de geração e propagação de identificador único por requisição
- `package.json:32` biblioteca de identificador já presente no projeto, suficiente para gerar o identificador do evento
- `prisma/schema.prisma:75` convenção de identificador adotada no projeto
- `TRANSCRICAO.md` `[09:17]` Larissa: progressão de backoff que define a janela em que a repetição pode ocorrer
- `TRANSCRICAO.md` `[09:24]` Diego: garantia at-least-once assumida explicitamente
- `TRANSCRICAO.md` `[09:25]` Bruno, Diego e Sofia: identificador por evento, objeção de transferência de responsabilidade e resposta com padrão de mercado
- `TRANSCRICAO.md` `[09:26]` Marcos e Larissa: compromisso de documentação e registro da decisão
- `TRANSCRICAO.md` `[09:34]` Marcos: histórico de entregas por endpoint
- `TRANSCRICAO.md` `[09:51]` Diego e Larissa: convenção de identificador das tabelas novas
- [Idempotência e deduplicação de webhook por identificador de evento](https://hookdeck.com/webhooks/guides/implement-webhook-idempotency)
- [Entrega at-least-once e identificadores usados por Stripe, GitHub e Shopify](https://www.hooklistener.com/learn/webhook-idempotency-and-deduplication)
