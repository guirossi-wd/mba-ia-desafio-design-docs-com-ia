# ADR-006: Implementação do módulo de webhooks sobre os padrões já existentes na codebase

**Status:** Aceito
**Data:** 2026-07-29
**ADRs Relacionados:** ADR-001, ADR-002

## Contexto e Declaração do Problema

Larissa abriu o bloco de estrutura de código em `[09:27]` e passou a palavra para Bruno, que é quem conhece as convenções do sistema de pedidos.

A organização do código é uniforme: cada domínio é um módulo com controller, service, repository, rotas e schemas de validação. A validação de entrada é declarativa e aplicada na própria rota, como se vê nos módulos existentes.

Existem também convenções transversais: uma hierarquia de erros de aplicação com códigos em formato constante, um tratador central que já converte esses erros em resposta HTTP e um logger estruturado usado no projeto inteiro.

A uniformidade vai além do código de produção. O projeto tem um único padrão de teste, de ponta a ponta contra a aplicação e o banco reais, com limpeza de tabelas antes de cada caso e fábricas compartilhadas. Qualquer coisa fora desse padrão fica testável de um jeito diferente do resto.

A pergunta é se a feature de webhooks segue essas convenções ou constrói as suas.

Há também uma questão específica de acoplamento. O ADR-001 exige que a inserção do evento aconteça dentro da transação que muda o status do pedido. Isso significa que o serviço de pedidos precisa de alguma forma de gravar na outbox, e existe mais de um jeito de dar a ele essa capacidade. Bruno levantou o ponto em `[09:41]`: passar um repository de webhook para o serviço de pedidos, ou expor uma função de publicação que aceite a transação em curso.

O risco de errar aqui não é funcional, é de manutenção. Um módulo que não segue as convenções do projeto obriga quem der manutenção a manter dois modelos mentais, e a feature de webhooks vai ser tocada justamente por quem já mantém o resto.

## Fatores de Decisão

- As convenções existentes são uniformes em cinco módulos de domínio, o que as torna previsíveis para quem já trabalha no projeto.
- O tratador central de erro já cobre a hierarquia de erros de aplicação, os erros de validação e os erros de banco, o que significa que erros novos são atendidos sem alteração nenhuma nesse ponto.
- Não há intenção de introduzir dependência nova no projeto.
- A inserção na outbox precisa participar da transação já aberta pelo serviço de pedidos, restrição vinda do ADR-001.
- Acoplamento entre o domínio de pedidos e o de webhooks deve ser o mínimo necessário para satisfazer a atomicidade.
- O projeto tem um único padrão de teste automatizado, e a feature precisa caber nele para ser verificável como o resto do sistema.

## Alternativas Consideradas

1. Módulo de webhooks seguindo as convenções existentes, com integração por função que recebe a transação em curso.
2. Estrutura própria para a feature, com suas próprias convenções de erro, log e organização.
3. Convenções existentes, mas com injeção de um repository de webhook dentro do serviço de pedidos.

## Resultado da Decisão

Opção escolhida: **reuso máximo das convenções existentes**, com a integração feita por uma função de publicação que recebe a transação em curso. Bruno propôs a estrutura em `[09:27]`, Diego aprovou em `[09:28]`, e Larissa registrou a decisão em `[09:30]` enumerando o que é reaproveitado: a classe base de erro, o logger, o tratador central de erro, o padrão de módulos, o padrão de schemas de validação e o padrão de códigos de erro.

Os pontos concretos de reuso:

**Estrutura de módulo.** A feature vira um módulo em `src/modules/webhooks`, com a mesma composição dos módulos existentes. O módulo de clientes em `src/modules/customers` serve como referência direta, por reunir a composição completa do padrão.

**Hierarquia de erros.** Os erros da feature estendem as classes de `src/shared/errors/http-errors.ts` e recebem códigos com prefixo `WEBHOOK_`, prefixo que Larissa fixou em `[09:29]`. A classe base `AppError`, definida em `src/shared/errors/app-error.ts`, já carrega código de erro, status HTTP e detalhes estruturados, que é tudo o que a feature precisa.

**Tratador central de erro.** Nenhuma alteração em `src/middlewares/error.middleware.ts`. Ele já trata qualquer instância de `AppError`, e é esse fato que sustenta o reuso: os erros novos passam a ser atendidos sem tocar nesse arquivo.

**Autorização.** O reprocessamento da DLQ usa `requireRole` de `src/middlewares/auth.middleware.ts`, conforme decidido no ADR-003. O uso existente em `src/modules/users/user.routes.ts` é o modelo.

**Log.** O worker do ADR-002 usa o mesmo logger de `src/shared/logger/index.ts`, sem introduzir biblioteca nova.

**Integração transacional.** Bruno propôs em `[09:41]` uma função `publishWebhookEvent` que recebe como primeiro argumento a transação em curso, mais o pedido e os status de origem e destino. Diego aprovou no mesmo minuto com a justificativa do menor acoplamento: função recebendo a transação, sem injetar um repository inteiro. O serviço de pedidos passa a chamar essa função dentro do bloco transacional que já existe em `src/modules/orders/order.service.ts`, e o tipo da transação já está declarado nesse arquivo.

## Prós e Contras das Alternativas

### Convenções existentes com função recebendo a transação

- Boa, porque quem já mantém o projeto não precisa aprender nada novo para dar manutenção na feature.
- Boa, porque o tratador central de erro atende os erros novos sem nenhuma alteração, o que reduz a superfície de mudança em código compartilhado.
- Boa, porque o acoplamento fica no menor grau possível: o serviço de pedidos conhece uma função, não um objeto com ciclo de vida.
- Boa, porque não entra dependência nova no projeto, o que mantém a superfície de atualização e de segurança inalterada.
- Ruim, porque a feature herda também as limitações das convenções atuais, entre elas a ausência de instrumentação de métrica no projeto.
- Ruim, porque o serviço de pedidos passa a importar algo do domínio de webhooks, criando dependência entre módulos que antes eram independentes.

### Estrutura própria com convenções novas

- Boa, porque permitiria adotar de saída padrões mais adequados a processamento assíncrono, que as convenções atuais não cobrem.
- Ruim, porque cria dois modelos mentais no mesmo repositório, e o custo recai sobre o mesmo time.
- Ruim, porque erros fora da hierarquia existente não seriam tratados pelo tratador central, exigindo alteração justamente no ponto mais compartilhado do sistema.
- Ruim, porque contraria a intenção declarada na reunião de não introduzir nada novo.

### Injeção de repository de webhook no serviço de pedidos

- Boa, porque daria ao serviço de pedidos acesso completo às operações de webhook, útil se no futuro ele precisar de mais do que publicar evento.
- Boa, porque manteria a montagem de dependências explícita no ponto onde a aplicação já a monta.
- Ruim, porque acopla o serviço de pedidos ao ciclo de vida de um objeto de outro domínio, e obriga a mudar a montagem de dependências da aplicação.
- Ruim, porque expõe muito mais capacidade do que o caso de uso exige, alargando a superfície de acoplamento sem necessidade.

## Consequências

- Boa, porque o custo de manutenção da feature fica próximo ao dos módulos existentes, e qualquer pessoa do time consegue abrir o código e reconhecer a estrutura.
- Boa, porque os erros da feature aparecem no contrato da API com o mesmo formato de resposta dos demais, o que mantém a experiência de integração uniforme para o cliente.
- Boa, porque o prefixo comum nos códigos de erro torna trivial filtrar, contar e alarmar por falhas específicas da feature.
- Boa, porque a feature entra no mesmo padrão de teste do projeto, de ponta a ponta contra banco real, sem precisar de infraestrutura de teste própria.
- Ruim, porque o serviço de pedidos deixa de ser independente do domínio de webhooks, e uma falha na publicação do evento passa a ser capaz de desfazer uma mudança de status, que é o comportamento desejado do ADR-001 mas também um risco novo no caminho crítico da operação.
- Ruim, porque as convenções atuais não têm nada previsto para processo de longa duração: não existe padrão de log de ciclo de worker, nem de métrica, nem de verificação de saúde de processo que não seja HTTP. A feature vai precisar estabelecer esses padrões, e será a primeira a fazê-lo no projeto.
- Ruim, porque o padrão de teste atual pressupõe requisição HTTP contra a aplicação, e o ciclo do worker não é uma requisição, o que deixa a parte assíncrona da feature sem molde pronto para verificação automatizada.
- Neutra, porque a decisão de não introduzir dependência nova é viável apenas porque a plataforma de execução já oferece o necessário: geração de identificador único e função de hash criptográfico já estão disponíveis no projeto ou na runtime. Se algum requisito futuro exigir capacidade ausente, essa restrição precisa ser revisitada.

## Referências

- `src/modules/orders/order.service.ts:24` tipo da transação já declarado, usado pela função de publicação
- `src/shared/errors/http-errors.ts:33` classes de erro que os erros da feature vão estender
- `src/middlewares/error.middleware.ts:15` tratamento central que atende os erros novos sem alteração
- `src/middlewares/auth.middleware.ts:49` autorização por papel reaproveitada no reprocessamento da DLQ
- `src/modules/customers/customer.routes.ts:12` módulo de referência para a composição do módulo de webhooks
- `TRANSCRICAO.md` `[09:27]` Larissa e Bruno: abertura do bloco de padrões e proposta do módulo
- `TRANSCRICAO.md` `[09:28]` Diego e Bruno: aprovação da estrutura e hierarquia de erros existente
- `TRANSCRICAO.md` `[09:29]` Larissa, Bruno e Diego: prefixo dos códigos de erro, tratador central, logger e nenhuma dependência nova
- `TRANSCRICAO.md` `[09:30]` Bruno e Larissa: cliente de banco por processo e registro da decisão de reuso
- `TRANSCRICAO.md` `[09:36]` Larissa: reuso do mecanismo de autorização por papel no reprocessamento
- `TRANSCRICAO.md` `[09:41]` Bruno e Diego: função de publicação recebendo a transação em curso
