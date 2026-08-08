# Tracker de Rastreabilidade

Este documento mapeia cada item registrado no pacote de design docs à sua origem: uma fala da reunião técnica ou um trecho do código da aplicação.

Ele existe para uma finalidade específica: se uma linha não consegue ter a coluna **Localização** preenchida, aquele item não tem origem identificável e não deveria estar na documentação. É a defesa do pacote contra afirmação inventada.

## Como ler

- **Fonte `TRANSCRICAO`**: a origem é [`TRANSCRICAO.md`](../TRANSCRICAO.md), na raiz do repositório. A localização é o horário mais o nome de quem falou, no formato `[hh:mm] Nome`, que é o formato em que o arquivo está organizado. A reunião não tem data de calendário registrada, apenas o horário.
- **Fonte `CODIGO`**: a origem é um arquivo deste repositório, identificado pelo caminho.

Alguns itens dos documentos são decisões de desenho tomadas pela própria especificação, e não falas da reunião: nome de tabela, de rota, de código de erro, prioridade de requisito e as metas que ninguém fixou. Eles estão marcados como tal no ponto em que aparecem nos documentos e **não entram neste tracker**, porque não têm origem externa a rastrear. A seção final lista quais são.

## Cobertura

| Documento | Itens rastreados |
|---|---|
| `docs/adrs/` (8 arquivos) | 9 |
| `docs/RFC.md` | 40 |
| `docs/FDD.md` | 74 |
| `docs/PRD.md` | 53 |
| **Total** | **176** |

Das 176 linhas, 147 têm fonte `TRANSCRICAO` (84 por cento) e 29 têm fonte `CODIGO` (16 por cento), apontando para 23 arquivos distintos do repositório.

Quando um item se apoia em duas falas, ou em uma fala e um arquivo, a coluna **Localização** traz as duas, cada uma com a parte do item que ela sustenta.

---

## ADRs

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Emissão de eventos via padrão Outbox no MySQL, na mesma transação da mudança de status | TRANSCRICAO | `[09:06] Diego` (inserção na mesma transação) e `[09:08] Larissa` (decisão fechada) |
| ADR-002 | `docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md` | Decisão | Worker em processo separado, lendo a outbox a cada 2 segundos | TRANSCRICAO | `[09:10] Larissa` (ciclo de 2 segundos) e `[09:11] Diego` (processo separado) |
| ADR-003 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Decisão | Cinco tentativas com backoff exponencial e DLQ em tabela separada | TRANSCRICAO | `[09:17] Larissa` (cinco tentativas e progressão) e `[09:18] Diego` (dead letter em tabela separada) |
| ADR-004 | `docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md` | Decisão | HMAC-SHA256 sobre o corpo, com secret exclusiva por endpoint e rotação | TRANSCRICAO | `[09:22] Sofia` |
| ADR-005 | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Decisão | Entrega at-least-once com deduplicação pelo cliente via identificador de evento | TRANSCRICAO | `[09:26] Larissa` |
| ADR-006 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Decisão | Reuso máximo dos padrões já existentes na codebase | TRANSCRICAO | `[09:30] Larissa` |
| ADR-007 | `docs/adrs/ADR-007-snapshot-do-payload-na-outbox.md` | Decisão | Snapshot do conteúdo do evento no momento da inserção | TRANSCRICAO | `[09:52] Larissa` |
| ADR-008 | `docs/adrs/ADR-008-filtro-de-eventos-na-insercao.md` | Decisão | Filtro de eventos aplicado na inserção, e não no envio | TRANSCRICAO | `[09:34] Bruno` |
| ADR-006-COD | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Restrição | Classe base de erro que já carrega código, status HTTP e detalhes estruturados, e por isso não precisa de nada novo | CODIGO | `src/shared/errors/app-error.ts` |

---

## RFC

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-CTX-01 | `docs/RFC.md` | Contexto | Três clientes B2B pediram notificação formal: Atlas Comercial, MaxDistribuição e Nova Cargo | TRANSCRICAO | `[09:00] Marcos` |
| RFC-CTX-02 | `docs/RFC.md` | Restrição | Risco comercial: a Atlas pode migrar para um concorrente se não houver entrega até o fim do trimestre | TRANSCRICAO | `[09:00] Marcos` |
| RFC-CTX-03 | `docs/RFC.md` | Requisito Não Funcional | Meta de latência percebida abaixo de 10 segundos | TRANSCRICAO | `[09:02] Marcos` |
| RFC-CTX-04 | `docs/RFC.md` | Restrição | Escopo exclusivamente outbound, sem webhooks de entrada | TRANSCRICAO | `[09:02] Marcos` |
| RFC-CTX-05 | `docs/RFC.md` | Contexto | A transação de mudança de status já executa três escritas e não pode carregar chamada HTTP | TRANSCRICAO | `[09:04] Bruno` |
| RFC-PROP-01 | `docs/RFC.md` | Decisão | Inserção do evento na outbox dentro da transação, com função que recebe a transação em curso | TRANSCRICAO | `[09:41] Bruno` |
| RFC-PROP-02 | `docs/RFC.md` | Decisão | Worker abre o próprio cliente de banco, apontando para o mesmo banco da API | TRANSCRICAO | `[09:30] Bruno` |
| RFC-PROP-03 | `docs/RFC.md` | Restrição | Nenhuma dependência nova entra no projeto | TRANSCRICAO | `[09:29] Bruno` |
| RFC-PROP-04 | `docs/RFC.md` | Decisão | Ponto de entrada de processo do worker espelha o que já existe para subir a API | CODIGO | `src/server.ts` |
| RFC-PROP-05 | `docs/RFC.md` | Decisão | Identificador das tabelas novas segue a convenção de UUID do resto do projeto | TRANSCRICAO | `[09:51] Larissa` |
| RFC-PROP-06 | `docs/RFC.md` | Decisão | Reuso dos padrões existentes: hierarquia de erros, tratador central, autorização por papel e logger | TRANSCRICAO | `[09:30] Larissa` |
| RFC-PROP-07 | `docs/RFC.md` | Decisão | Filtro de assinatura aplicado na inserção, de modo que toda linha da outbox tem destinatário | TRANSCRICAO | `[09:34] Bruno` |
| RFC-PROP-08 | `docs/RFC.md` | Decisão | Linha da outbox guarda o conteúdo do evento já montado, e não apenas a referência ao pedido | TRANSCRICAO | `[09:52] Larissa` |
| RFC-PROP-09 | `docs/RFC.md` | Requisito Não Funcional | Assinatura HMAC-SHA256 do corpo, com secret exclusiva do endpoint e rotação com 24 horas de sobreposição | TRANSCRICAO | `[09:22] Sofia` |
| RFC-PROP-10 | `docs/RFC.md` | Requisito Não Funcional | URL cadastrada precisa usar canal cifrado, verificado na validação de entrada | TRANSCRICAO | `[09:23] Sofia` |
| RFC-PROP-11 | `docs/RFC.md` | Requisito Não Funcional | Tempo limite de 10 segundos na chamada ao cliente, com ausência de resposta contando como falha | TRANSCRICAO | `[09:42] Diego` |
| RFC-PROP-12 | `docs/RFC.md` | Requisito Funcional | Superfície de API: CRUD de configuração, rotação de secret, histórico de entregas e replay administrativo | TRANSCRICAO | `[09:33] Bruno` |
| RFC-ESC-01 | `docs/RFC.md` | Restrição | Fora do alcance: aviso ao cliente por e-mail ou outro canal quando a integração falha | TRANSCRICAO | `[09:37] Larissa` |
| RFC-ESC-02 | `docs/RFC.md` | Restrição | Fora do alcance: painel visual para o cliente, projeto separado do time de frontend | TRANSCRICAO | `[09:40] Larissa` |
| RFC-ESC-03 | `docs/RFC.md` | Restrição | Fora do alcance: webhooks de entrada; o escopo é exclusivamente de saída | TRANSCRICAO | `[09:02] Marcos` |
| RFC-ESC-04 | `docs/RFC.md` | Restrição | Fora do alcance: política de arquivamento das linhas já entregues | TRANSCRICAO | `[09:08] Diego` |
| RFC-IMP-01 | `docs/RFC.md` | Restrição | Único ponto de alteração no código existente é a transação de mudança de status | CODIGO | `src/modules/orders/order.service.ts` |
| RFC-IMP-02 | `docs/RFC.md` | Restrição | Configuração de omissão de campo em log precisa passar a cobrir a secret do webhook | CODIGO | `src/shared/logger/index.ts` |
| RFC-ALT-01 | `docs/RFC.md` | Trade-off | Disparo HTTP síncrono descartado: cliente fora do ar obrigaria a desfazer mudança de status legítima | TRANSCRICAO | `[09:04] Bruno` |
| RFC-ALT-02 | `docs/RFC.md` | Trade-off | Redis Streams ou broker dedicado descartado como overengineering para o tamanho do time | TRANSCRICAO | `[09:07] Diego` |
| RFC-ALT-03 | `docs/RFC.md` | Trade-off | Trigger de banco descartado: o MySQL não notifica processo externo, ao contrário do PostgreSQL | TRANSCRICAO | `[09:09] Diego` |
| RFC-ALT-04 | `docs/RFC.md` | Trade-off | Worker dentro do processo da API descartado: restart da API derrubaria a entrega | TRANSCRICAO | `[09:11] Diego` |
| RFC-ALT-05 | `docs/RFC.md` | Trade-off | Três tentativas descartadas: cobririam 30 minutos, e houve caso real de indisponibilidade de duas horas | TRANSCRICAO | `[09:16] Diego` |
| RFC-ALT-06 | `docs/RFC.md` | Trade-off | Retry indefinido descartado: evento fica pendurado para sempre se o cliente sumiu | TRANSCRICAO | `[09:15] Diego` |
| RFC-ALT-07 | `docs/RFC.md` | Trade-off | Secret global descartada: se vaza uma, vaza tudo | TRANSCRICAO | `[09:21] Sofia` |
| RFC-ALT-08 | `docs/RFC.md` | Trade-off | Exactly-once descartado: exigiria coordenação das duas pontas, com custo desproporcional | TRANSCRICAO | `[09:25] Diego` |
| RFC-QA-01 | `docs/RFC.md` | Questão em aberto | Rate limiting de saída por cliente: observar e implementar apenas se virar problema | TRANSCRICAO | `[09:39] Diego` |
| RFC-QA-02 | `docs/RFC.md` | Questão em aberto | Escala do worker além de uma instância: particionar por pedido ou lock pessimista, problema do futuro | TRANSCRICAO | `[09:13] Diego` |
| RFC-QA-03 | `docs/RFC.md` | Questão em aberto | Política de retenção e arquivamento da outbox, estimada em 30 dias e deixada fora do escopo | TRANSCRICAO | `[09:08] Diego` |
| RFC-QA-04 | `docs/RFC.md` | Questão em aberto | Endurecimento das permissões do CRUD de configuração fica para mais adiante | TRANSCRICAO | `[09:37] Sofia` |
| RFC-QA-05 | `docs/RFC.md` | Questão em aberto | Aviso ao cliente sobre integração quebrada adiado para a próxima fase | TRANSCRICAO | `[09:37] Larissa` |
| RFC-RISCO-01 | `docs/RFC.md` | Restrição | Revisão de segurança de dois dias úteis é pré-requisito de deploy | TRANSCRICAO | `[09:46] Sofia` |
| RFC-RISCO-02 | `docs/RFC.md` | Restrição | Prazo estimado em três sprints, com a revisão de segurança incluída no fim | TRANSCRICAO | `[09:47] Larissa` |
| RFC-RISCO-03 | `docs/RFC.md` | Restrição | Compromisso comercial de entrega para o fim de novembro | TRANSCRICAO | `[09:45] Marcos` |
| RFC-RISCO-04 | `docs/RFC.md` | Trade-off | Mitigação do risco de cliente que não deduplica: documentação destacada no portal do desenvolvedor, assumida pelo PM | TRANSCRICAO | `[09:26] Marcos` |

---

## FDD

### Fluxos e comportamento

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-FLUXO-01 | `docs/FDD.md` | Requisito Funcional | Inserção do evento dentro da transação de mudança de status, com rollback se falhar | TRANSCRICAO | `[09:40] Bruno` |
| FDD-FLUXO-02 | `docs/FDD.md` | Requisito Funcional | Se nenhum endpoint assina a transição, nada é gravado na outbox | TRANSCRICAO | `[09:34] Bruno` |
| FDD-FLUXO-03 | `docs/FDD.md` | Requisito Funcional | Worker lê os pendentes mais antigos em lote, processa e marca | TRANSCRICAO | `[09:09] Diego` |
| FDD-FLUXO-04 | `docs/FDD.md` | Requisito Não Funcional | Ciclo de leitura da outbox a cada 2 segundos | TRANSCRICAO | `[09:10] Larissa` |
| FDD-FLUXO-05 | `docs/FDD.md` | Requisito Funcional | Falha gera nova tentativa com espaçamento crescente até o teto de cinco | TRANSCRICAO | `[09:17] Larissa` |
| FDD-FLUXO-06 | `docs/FDD.md` | Requisito Funcional | Evento que esgota as tentativas é movido para tabela de dead letter com conteúdo, motivo e horário | TRANSCRICAO | `[09:18] Diego` |
| FDD-FLUXO-07 | `docs/FDD.md` | Requisito Funcional | Reprocessamento manual recoloca o item na outbox como pendente | TRANSCRICAO | `[09:18] Diego` |
| FDD-FLUXO-08 | `docs/FDD.md` | Requisito Funcional | Conteúdo do evento é montado e persistido na inserção, e não no envio | TRANSCRICAO | `[09:52] Larissa` |
| FDD-FLUXO-09 | `docs/FDD.md` | Requisito Não Funcional | Ordenação por pedido garantida apenas enquanto o worker for único | TRANSCRICAO | `[09:12] Diego` |
| FDD-FLUXO-10 | `docs/FDD.md` | Restrição | Ordenação global nunca foi pedida pelos clientes | TRANSCRICAO | `[09:14] Marcos` |

### Modelo de dados

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-DADOS-01 | `docs/FDD.md` | Decisão | Tabela de outbox no MySQL existente, com o evento da mudança de status | TRANSCRICAO | `[09:06] Diego` |
| FDD-DADOS-02 | `docs/FDD.md` | Decisão | Índice no estado do evento e na data de criação | TRANSCRICAO | `[09:08] Diego` |
| FDD-DADOS-03 | `docs/FDD.md` | Decisão | Estados do evento: pendente, processando, falhou e entregue | TRANSCRICAO | `[09:08] Diego` |
| FDD-DADOS-04 | `docs/FDD.md` | Decisão | Tabela de dead letter separada da outbox | TRANSCRICAO | `[09:18] Diego` |
| FDD-DADOS-05 | `docs/FDD.md` | Decisão | Tabela de configuração de webhook com URL, secret, cliente e estado ativo | TRANSCRICAO | `[09:21] Bruno` |
| FDD-DADOS-06 | `docs/FDD.md` | Requisito Funcional | Registro das entregas para sustentar o histórico consultável pelo cliente | TRANSCRICAO | `[09:34] Marcos` |
| FDD-DADOS-07 | `docs/FDD.md` | Decisão | Chave primária em UUID, seguindo o padrão do resto do projeto | TRANSCRICAO | `[09:51] Larissa` |
| FDD-DADOS-08 | `docs/FDD.md` | Restrição | Convenções de modelagem a seguir: identificador, timestamps, índices e mapeamento de nome | CODIGO | `prisma/schema.prisma` |
| FDD-DADOS-09 | `docs/FDD.md` | Restrição | Domínio de valores do filtro de status vem da enumeração de status do pedido | CODIGO | `prisma/schema.prisma` |
| FDD-DADOS-10 | `docs/FDD.md` | Restrição | Coluna de conteúdo estruturado como precedente para o snapshot do evento | CODIGO | `prisma/schema.prisma` |

### Contratos públicos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-CONTRATO-01 | `docs/FDD.md` | Requisito Funcional | Cadastro de webhook com URL, lista de status assinados e secret devolvida na criação | TRANSCRICAO | `[09:31] Marcos` |
| FDD-CONTRATO-02 | `docs/FDD.md` | Requisito Funcional | Endpoints de edição, remoção e listagem dos webhooks de um cliente | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-03 | `docs/FDD.md` | Requisito Funcional | Endpoint de rotação de secret, com a anterior válida por 24 horas | TRANSCRICAO | `[09:21] Sofia` |
| FDD-CONTRATO-04 | `docs/FDD.md` | Requisito Funcional | Consulta ao histórico de entregas de um endpoint | TRANSCRICAO | `[09:34] Marcos` |
| FDD-CONTRATO-05 | `docs/FDD.md` | Requisito Funcional | Endpoint administrativo de replay de item da dead letter | TRANSCRICAO | `[09:35] Diego` |
| FDD-CONTRATO-06 | `docs/FDD.md` | Restrição | Identificador do cliente vem no corpo ou no caminho, e não do token de autenticação | TRANSCRICAO | `[09:32] Larissa` |
| FDD-CONTRATO-07 | `docs/FDD.md` | Requisito Funcional | Conteúdo do evento com identificação, tipo, horário, pedido, transição, cliente e valores básicos | TRANSCRICAO | `[09:43] Diego` |
| FDD-CONTRATO-08 | `docs/FDD.md` | Restrição | Itens do pedido ficam fora do conteúdo; o cliente consulta a API de pedidos se precisar | TRANSCRICAO | `[09:43] Diego` |
| FDD-CONTRATO-09 | `docs/FDD.md` | Requisito Funcional | Cabeçalhos de identificação do evento, assinatura, horário do disparo e tipo de conteúdo | TRANSCRICAO | `[09:44] Diego` |
| FDD-CONTRATO-10 | `docs/FDD.md` | Requisito Funcional | Cabeçalho identificando o cadastro de webhook que originou o envio | TRANSCRICAO | `[09:44] Sofia` |
| FDD-CONTRATO-11 | `docs/FDD.md` | Restrição | Envelope de resposta e formato de listagem paginada seguem o padrão do projeto | CODIGO | `src/shared/http/response.ts` |

### Erros e resiliência

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-ERRO-01 | `docs/FDD.md` | Restrição | Códigos de erro do módulo usam o prefixo `WEBHOOK_` | TRANSCRICAO | `[09:29] Larissa` |
| FDD-ERRO-02 | `docs/FDD.md` | Decisão | Erros da feature seguem a hierarquia de erros já existente no projeto | TRANSCRICAO | `[09:28] Bruno` |
| FDD-ERRO-03 | `docs/FDD.md` | Restrição | Classes de erro que a feature estende, com código, status e detalhes | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-ERRO-04 | `docs/FDD.md` | Restrição | Tratador central de erro já converte qualquer erro de aplicação em resposta, sem alteração | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-ERRO-05 | `docs/FDD.md` | Requisito Funcional | URL sem canal cifrado é recusada com erro de validação | TRANSCRICAO | `[09:23] Sofia` |
| FDD-ERRO-06 | `docs/FDD.md` | Requisito Não Funcional | Conteúdo acima do limite é recusado, e não truncado | TRANSCRICAO | `[09:23] Sofia` |
| FDD-ERRO-07 | `docs/FDD.md` | Requisito Não Funcional | Teto de 64KB para o conteúdo do evento | TRANSCRICAO | `[09:24] Diego` |
| FDD-RES-01 | `docs/FDD.md` | Requisito Não Funcional | Tempo limite de 10 segundos na chamada ao endpoint do cliente | TRANSCRICAO | `[09:42] Diego` |
| FDD-RES-02 | `docs/FDD.md` | Decisão | Cinco tentativas de entrega | TRANSCRICAO | `[09:16] Larissa` |
| FDD-RES-03 | `docs/FDD.md` | Decisão | Progressão de espera em 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas | TRANSCRICAO | `[09:17] Diego` |
| FDD-RES-04 | `docs/FDD.md` | Requisito Não Funcional | Janela total de retry de quase 15 horas, aceita pelo produto | TRANSCRICAO | `[09:17] Marcos` |
| FDD-RES-05 | `docs/FDD.md` | Requisito Funcional | Cliente que não responde no prazo é tratado como falha sujeita a retry | TRANSCRICAO | `[09:42] Diego` |
| FDD-RES-06 | `docs/FDD.md` | Requisito Não Funcional | Garantia at-least-once, com possibilidade de entrega repetida | TRANSCRICAO | `[09:24] Diego` |
| FDD-RES-07 | `docs/FDD.md` | Requisito Funcional | Identificador único por evento, gerado na inserção, usado pelo cliente para deduplicar | TRANSCRICAO | `[09:25] Diego` |

### Segurança

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-SEG-01 | `docs/FDD.md` | Requisito Não Funcional | Assinatura do corpo com HMAC, enviada em cabeçalho próprio | TRANSCRICAO | `[09:20] Sofia` |
| FDD-SEG-02 | `docs/FDD.md` | Decisão | Algoritmo SHA-256, por ser padrão de mercado com biblioteca disponível em qualquer stack | TRANSCRICAO | `[09:20] Sofia` |
| FDD-SEG-03 | `docs/FDD.md` | Requisito Não Funcional | Secret exclusiva por endpoint, e não uma secret global da plataforma | TRANSCRICAO | `[09:21] Sofia` |
| FDD-SEG-04 | `docs/FDD.md` | Requisito Funcional | Secret gerada pela plataforma e devolvida ao cliente na criação | TRANSCRICAO | `[09:31] Marcos` |
| FDD-SEG-05 | `docs/FDD.md` | Requisito Funcional | Replay da dead letter restrito a papel administrativo, com registro de quem executou | TRANSCRICAO | `[09:36] Sofia` |
| FDD-SEG-06 | `docs/FDD.md` | Decisão | Reuso do mecanismo de autorização por papel já existente | TRANSCRICAO | `[09:36] Larissa` |
| FDD-SEG-07 | `docs/FDD.md` | Restrição | Mecanismo de autorização por papel a reutilizar, e seu único uso atual como modelo | CODIGO | `src/middlewares/auth.middleware.ts` (declaração) e `src/modules/users/user.routes.ts` (único uso atual) |
| FDD-SEG-08 | `docs/FDD.md` | Restrição | Lista de campos omitidos em log, que precisa passar a cobrir a secret do webhook | CODIGO | `src/shared/logger/index.ts` |

### Observabilidade e integração com o código

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-OBS-01 | `docs/FDD.md` | Decisão | Log estruturado do projeto reaproveitado, sem biblioteca nova | TRANSCRICAO | `[09:29] Bruno` |
| FDD-OBS-02 | `docs/FDD.md` | Restrição | Padrão de log com evento nomeado e campos estruturados, a ser seguido | CODIGO | `src/middlewares/request-logger.middleware.ts` |
| FDD-OBS-03 | `docs/FDD.md` | Requisito Funcional | Registro de cada entrega com resultado, resposta e tempo de resposta | TRANSCRICAO | `[09:34] Marcos` |
| FDD-INT-01 | `docs/FDD.md` | Restrição | Transação de mudança de status onde a inserção do evento passa a acontecer | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | `docs/FDD.md` | Decisão | Função de publicação recebendo a transação em curso, sem injetar repository | TRANSCRICAO | `[09:41] Bruno` |
| FDD-INT-03 | `docs/FDD.md` | Decisão | Módulo em `src/modules/webhooks` com a mesma composição dos módulos existentes | TRANSCRICAO | `[09:27] Bruno` |
| FDD-INT-04 | `docs/FDD.md` | Restrição | Composição de módulo a espelhar: rota com autenticação e validação declarativa | CODIGO | `src/modules/customers/customer.routes.ts` |
| FDD-INT-05 | `docs/FDD.md` | Decisão | Worker como ponto de entrada separado, com a lógica dentro do módulo | TRANSCRICAO | `[09:28] Bruno` |
| FDD-INT-06 | `docs/FDD.md` | Restrição | Cliente de banco é uma instância por processo, por isso o worker abre a sua | CODIGO | `src/config/database.ts` |
| FDD-INT-07 | `docs/FDD.md` | Decisão | Cliente de banco próprio para o worker, mesma string de conexão | TRANSCRICAO | `[09:30] Bruno` |
| FDD-INT-08 | `docs/FDD.md` | Restrição | Mecanismo declarativo de validação de entrada aplicado na rota | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-INT-09 | `docs/FDD.md` | Restrição | Padrão de validação de formato de string na entrada, modelo para a URL do endpoint | CODIGO | `src/modules/customers/customer.schemas.ts` |
| FDD-INT-10 | `docs/FDD.md` | Restrição | Registro do roteador do módulo junto dos cinco existentes | CODIGO | `src/routes/index.ts` |
| FDD-INT-11 | `docs/FDD.md` | Restrição | Montagem de repository, service e controller a seguir | CODIGO | `src/app.ts` |
| FDD-INT-12 | `docs/FDD.md` | Restrição | Máquina de estados do pedido, que define quais transições existem para notificar | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-INT-13 | `docs/FDD.md` | Restrição | Padrão de teste de ponta a ponta com limpeza de tabelas antes de cada caso | CODIGO | `tests/setup.ts` |
| FDD-INT-14 | `docs/FDD.md` | Restrição | Padrão de scripts de execução do projeto, onde entra o script do worker | CODIGO | `package.json` |
| FDD-INT-15 | `docs/FDD.md` | Restrição | Ponto de entrada existente que serve de modelo para o processo do worker | CODIGO | `src/server.ts` |
| FDD-INT-16 | `docs/FDD.md` | Decisão | Segundo ponto de entrada de processo com script próprio de execução | TRANSCRICAO | `[09:11] Larissa` |
| FDD-INT-17 | `docs/FDD.md` | Restrição | Biblioteca de geração de identificador já presente no projeto | CODIGO | `package.json` |
| FDD-INT-18 | `docs/FDD.md` | Restrição | Versão do MySQL declarada no ambiente do projeto | CODIGO | `docker-compose.yml` |

---

## PRD

### Contexto, objetivos e escopo

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | `docs/PRD.md` | Contexto | Pedido formal de três clientes B2B nominados | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-02 | `docs/PRD.md` | Contexto | Os clientes hoje descobrem mudanças consultando a API de pedidos repetidamente | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-03 | `docs/PRD.md` | Restrição | Risco de o cliente migrar para um concorrente se não houver entrega no prazo | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-04 | `docs/PRD.md` | Restrição | Escopo apenas de saída, confirmado em pergunta explícita de escopo | TRANSCRICAO | `[09:02] Sofia` |
| PRD-CTX-05 | `docs/PRD.md` | Contexto | A aplicação não tem hoje nenhum mecanismo de notificação, evento ou fila | CODIGO | `src/routes/index.ts` |
| PRD-OBJ-01 | `docs/PRD.md` | Requisito Não Funcional | Notificar dentro de 10 segundos, que é o que o cliente chama de tempo real | TRANSCRICAO | `[09:02] Marcos` |
| PRD-OBJ-02 | `docs/PRD.md` | Restrição | Compromisso comercial de entrega para o fim de novembro | TRANSCRICAO | `[09:45] Marcos` |
| PRD-OBJ-03 | `docs/PRD.md` | Restrição | Estimativa de três sprints, com a revisão de segurança incluída no fim | TRANSCRICAO | `[09:47] Larissa` |
| PRD-OBJ-04 | `docs/PRD.md` | Requisito Não Funcional | Janela de repetição automática de quase 15 horas | TRANSCRICAO | `[09:17] Diego` |
| PRD-OBJ-05 | `docs/PRD.md` | Requisito Não Funcional | Nenhuma mudança de status efetivada sem o evento correspondente registrado | TRANSCRICAO | `[09:40] Bruno` |
| PRD-ESC-01 | `docs/PRD.md` | Restrição | Fora de escopo: aviso ao cliente por e-mail quando a integração falha | TRANSCRICAO | `[09:37] Larissa` |
| PRD-ESC-02 | `docs/PRD.md` | Restrição | Fora de escopo: painel visual para o cliente, que é projeto do time de frontend | TRANSCRICAO | `[09:40] Larissa` |
| PRD-ESC-03 | `docs/PRD.md` | Restrição | Em aberto: limitação de taxa de envio, a observar e decidir depois | TRANSCRICAO | `[09:39] Larissa` |
| PRD-ESC-04 | `docs/PRD.md` | Restrição | Fora de escopo: arquivamento das linhas já entregues | TRANSCRICAO | `[09:08] Diego` |
| PRD-ESC-05 | `docs/PRD.md` | Restrição | Fora de escopo: webhooks de entrada | TRANSCRICAO | `[09:02] Marcos` |
| PRD-ESC-06 | `docs/PRD.md` | Restrição | Adiado: endurecimento das permissões do cadastro de webhook | TRANSCRICAO | `[09:37] Sofia` |

### Requisitos funcionais

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-FR-01 | `docs/PRD.md` | Requisito Funcional | Cadastro de endpoint com URL, lista de status assinados e secret devolvida na criação | TRANSCRICAO | `[09:31] Marcos` |
| PRD-FR-02 | `docs/PRD.md` | Requisito Funcional | Consulta dos endpoints cadastrados de um cliente | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-03 | `docs/PRD.md` | Requisito Funcional | Edição de endpoint | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-04 | `docs/PRD.md` | Requisito Funcional | Remoção de endpoint | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-05 | `docs/PRD.md` | Requisito Funcional | Filtro de eventos por lista de status que o endpoint quer ouvir | TRANSCRICAO | `[09:33] Marcos` |
| PRD-FR-06 | `docs/PRD.md` | Requisito Funcional | Troca da secret pela API, com a anterior válida por 24 horas | TRANSCRICAO | `[09:21] Sofia` |
| PRD-FR-07 | `docs/PRD.md` | Requisito Funcional | Entrega do evento assinado ao endpoint do cliente | TRANSCRICAO | `[09:20] Sofia` |
| PRD-FR-08 | `docs/PRD.md` | Requisito Funcional | Repetição automática com espaçamento crescente e teto de cinco tentativas | TRANSCRICAO | `[09:17] Larissa` |
| PRD-FR-09 | `docs/PRD.md` | Requisito Funcional | Registro de falha permanente e reprocessamento manual por endpoint administrativo | TRANSCRICAO | `[09:18] Diego` |
| PRD-FR-10 | `docs/PRD.md` | Requisito Funcional | Histórico das entregas feitas a um endpoint, com resultado, resposta e tempo | TRANSCRICAO | `[09:34] Marcos` |

### Requisitos não funcionais, decisões e riscos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-RNF-01 | `docs/PRD.md` | Requisito Não Funcional | A indisponibilidade do cliente não pode impedir a mudança de status de um pedido | TRANSCRICAO | `[09:04] Bruno` |
| PRD-RNF-02 | `docs/PRD.md` | Requisito Não Funcional | A entrega não pode estar presa ao ciclo de vida da API | TRANSCRICAO | `[09:11] Diego` |
| PRD-RNF-03 | `docs/PRD.md` | Requisito Não Funcional | Tempo limite de 10 segundos na chamada ao cliente | TRANSCRICAO | `[09:42] Diego` |
| PRD-RNF-04 | `docs/PRD.md` | Requisito Não Funcional | Assinatura HMAC-SHA256 sobre o conteúdo enviado | TRANSCRICAO | `[09:22] Sofia` |
| PRD-RNF-05 | `docs/PRD.md` | Requisito Não Funcional | Credencial exclusiva por endpoint, com rotação e sobreposição de 24 horas | TRANSCRICAO | `[09:22] Sofia` |
| PRD-RNF-06 | `docs/PRD.md` | Requisito Não Funcional | URL de destino obrigatoriamente em canal cifrado | TRANSCRICAO | `[09:23] Sofia` |
| PRD-RNF-07 | `docs/PRD.md` | Requisito Não Funcional | Teto de 64KB no conteúdo do evento, com recusa em vez de truncamento | TRANSCRICAO | `[09:24] Larissa` |
| PRD-RNF-08 | `docs/PRD.md` | Requisito Não Funcional | Garantia at-least-once com identificador estável para deduplicação | TRANSCRICAO | `[09:26] Larissa` |
| PRD-RNF-09 | `docs/PRD.md` | Requisito Não Funcional | Conteúdo reflete o estado do pedido no instante da transição | TRANSCRICAO | `[09:52] Larissa` |
| PRD-RNF-10 | `docs/PRD.md` | Requisito Não Funcional | Ordem por pedido garantida enquanto o processo de entrega for único | TRANSCRICAO | `[09:13] Larissa` |
| PRD-RNF-11 | `docs/PRD.md` | Restrição | Nenhuma dependência nova no projeto | TRANSCRICAO | `[09:29] Bruno` |
| PRD-RNF-12 | `docs/PRD.md` | Restrição | Mesmo banco e mesma stack; apenas o processo é diferente | TRANSCRICAO | `[09:11] Diego` |
| PRD-RNF-13 | `docs/PRD.md` | Requisito Não Funcional | Trilha de auditoria no reprocessamento de falha permanente | TRANSCRICAO | `[09:36] Sofia` |
| PRD-RNF-14 | `docs/PRD.md` | Restrição | Gestão do cadastro aceita qualquer papel autenticado nesta fase | TRANSCRICAO | `[09:37] Sofia` |
| PRD-RNF-15 | `docs/PRD.md` | Restrição | Nenhum contrato de API existente é alterado | CODIGO | `src/routes/index.ts` |
| PRD-DEC-01 | `docs/PRD.md` | Trade-off | Registrar o evento na mesma transação coloca o domínio de webhooks no caminho crítico | TRANSCRICAO | `[09:06] Diego` |
| PRD-DEC-02 | `docs/PRD.md` | Trade-off | Processo separado isola a entrega, mas acrescenta um processo a operar | TRANSCRICAO | `[09:11] Diego` |
| PRD-DEC-03 | `docs/PRD.md` | Trade-off | At-least-once transfere a deduplicação para o cliente | TRANSCRICAO | `[09:25] Sofia` |
| PRD-DEC-04 | `docs/PRD.md` | Trade-off | Credencial por endpoint limita o vazamento mas obriga a plataforma a custodiar uma por cadastro | TRANSCRICAO | `[09:21] Sofia` |
| PRD-DEC-05 | `docs/PRD.md` | Trade-off | Filtrar na inserção economiza linha mas impede reenviar histórico a endpoint cadastrado depois | TRANSCRICAO | `[09:34] Bruno` |
| PRD-DEP-01 | `docs/PRD.md` | Restrição | Revisão de segurança de dois dias úteis antes do deploy | TRANSCRICAO | `[09:46] Sofia` |
| PRD-DEP-02 | `docs/PRD.md` | Restrição | Documentação do comportamento de duplicidade no portal do desenvolvedor | TRANSCRICAO | `[09:26] Marcos` |
| PRD-DEP-03 | `docs/PRD.md` | Restrição | Documentação de integração via API no portal do desenvolvedor | TRANSCRICAO | `[09:40] Marcos` |
| PRD-RISCO-01 | `docs/PRD.md` | Restrição | Precedente de cliente que expôs credencial em log de aplicação | TRANSCRICAO | `[09:22] Diego` |
| PRD-RISCO-02 | `docs/PRD.md` | Restrição | Caso real de cliente com indisponibilidade de duas horas em manutenção planejada | TRANSCRICAO | `[09:16] Diego` |
| PRD-VAL-01 | `docs/PRD.md` | Restrição | Sessão de revisão do desenho com os engenheiros antes de começar a implementação | TRANSCRICAO | `[09:50] Larissa` |
| PRD-VAL-02 | `docs/PRD.md` | Restrição | Padrão de teste existente que os testes da feature seguem, para autenticação e para pedidos | CODIGO | `tests/auth.test.ts` e `tests/orders.test.ts` |

---

## Itens sem origem externa

Os itens abaixo aparecem nos documentos e **não** têm linha neste tracker, porque não vêm da reunião nem do código. São decisões tomadas pela própria documentação, e cada um está marcado como tal no ponto em que aparece.

| Item | Onde aparece | Natureza |
|---|---|---|
| Nomes de tabela `webhook_endpoints` e `webhook_deliveries` | `docs/FDD.md` seção 4.5 | Decisão de desenho, sobre a convenção de nomes do schema existente |
| Caminho da rota de rotação de secret | `docs/FDD.md` seção 5.5 | Decisão de desenho, sobre a operação exigida em `[09:21]` |
| Sete dos dez códigos de erro `WEBHOOK_` | `docs/FDD.md` seção 6.1 | Decisão de desenho, cada um sobre uma regra decidida na reunião; três foram nomeados por Bruno em `[09:28]` |
| Tamanho do lote de 20 eventos por ciclo | `docs/FDD.md` seção 6.2 | Decisão de desenho, sobre a interação entre o ciclo de 2 segundos e o tempo limite de 10 segundos |
| Valores de motivo de falha de entrega: `TIMEOUT`, `CONNECTION_ERROR` e `HTTP_ERROR` | `docs/FDD.md` seção 6.1 | Decisão de desenho, sobre a regra de tempo limite de `[09:42]` |
| Condições dos alarmes e dos painéis propostos | `docs/FDD.md` seção 7.4 | Proposta do documento; a reunião não discutiu monitoramento |
| Recuperação de eventos presos em processando | `docs/FDD.md` seção 4.2 | Decisão de desenho, sobre a mecânica de estados de `[09:08]` |
| Divisão entre schema e serviço na validação da URL | `docs/FDD.md` seção 11.4 | Decisão de desenho, para conciliar `[09:23]` com `[09:29]` |
| Nomes das métricas e dos eventos de log | `docs/FDD.md` seções 7.1 e 7.2 | Decisão de desenho, sobre o padrão de log observável no projeto |
| Prioridade de cada requisito funcional | `docs/PRD.md` seção de requisitos funcionais | Juízo do documento, com o critério declarado no início da seção |
| Meta de zero eventos sem registro | `docs/PRD.md` seção de objetivos e métricas | Leitura do documento sobre `[09:40]` e `[09:18]`; a reunião não fixou taxa |
| Blocos "O que destravaria a decisão" | `docs/RFC.md` seção de questões em aberto | Proposta do documento; a reunião adiou as questões sem definir critério de retomada |
| Mitigações dos riscos | `docs/RFC.md` e `docs/PRD.md` | Proposta do documento; a reunião identificou os riscos sem discutir mitigação |
