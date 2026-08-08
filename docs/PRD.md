### PRD: Order Management System Sistema de Webhooks de Notificação de Pedidos

Versão: v1

Data: 2026-07-29

Responsável: Marcos (Product Manager)

> Referências no formato `[hh:mm]` remetem à reunião registrada em [`TRANSCRICAO.md`](../TRANSCRICAO.md), na raiz do repositório. A proposta técnica está em [`RFC.md`](RFC.md), as decisões em [`docs/adrs/`](adrs/README.md) e a especificação de implementação em [`FDD.md`](FDD.md).

---

### Resumo

Esta feature dá ao Order Management System a capacidade de notificar sistemas de clientes quando o status de um pedido muda, sem que eles precisem consultar a API repetidamente. O cliente cadastra um endpoint HTTPS, escolhe quais status quer receber e passa a receber uma requisição assinada a cada transição relevante dos pedidos dele.

A entrega é exclusivamente de saída: a plataforma envia, o cliente recebe, e não existe caminho de entrada `[09:02]`. O escopo desta fase é de API, sem interface visual.

A feature nasce de um pedido formal de três clientes B2B e tem prazo comercial associado. Ela é composta por quatro partes: o cadastro e a gestão dos endpoints pelo cliente, a emissão do evento no momento da mudança de status, a entrega com assinatura e política de repetição, e a evidência do que foi enviado.

---

### Contexto e problema

Público-alvo

- Os três clientes B2B que fizeram o pedido formal: Atlas Comercial, MaxDistribuição e Nova Cargo `[09:00]`
- Os sistemas desses clientes, que hoje consultam a API de pedidos de tempos em tempos para descobrir mudanças
- Usuários com papel administrativo da plataforma, que precisam diagnosticar e reprocessar entregas que falharam em definitivo `[09:36]`

Cenários de uso chave

- Cliente cadastra o endpoint dele e escolhe receber apenas as transições que interessam, por exemplo apenas enviado e entregue `[09:33]`
- Um pedido muda para enviado e o sistema do cliente é notificado em segundos, sem consulta
- O endpoint do cliente fica fora do ar por duas horas em manutenção planejada e, ao voltar, recebe os eventos do período sem ninguém precisar agir `[09:16]`
- O cliente suspeita de um evento não recebido e consulta o histórico das últimas entregas feitas a ele, com sucesso ou falha, resposta e tempo `[09:34]`
- O cliente expôs a credencial de assinatura por engano e a troca pela API, mantendo a antiga válida em paralelo pelo tempo necessário para migrar `[09:21]` `[09:22]`
- A integração de um cliente permanece indisponível e o evento esgota as tentativas; alguém com papel administrativo reprocessa o item depois de o cliente corrigir o problema `[09:35]`

Onde essa feature será implantada

- No Order Management System existente, um sistema Node.js e TypeScript com banco MySQL, sem alterar nenhum contrato de API já publicado
- Como um módulo novo dentro do próprio sistema, mais um segundo processo dedicado à entrega, que roda separado da API `[09:11]`
- Sem infraestrutura nova: mesmo banco, mesma stack, mesmo ambiente `[09:07]`

Problemas priorizados

- **Os clientes descobrem mudanças por consulta repetida.** Hoje eles consultam a API de pedidos de tempos em tempos para ver se algo mudou, o que torna a integração lenta e cara do lado deles `[09:00]`. Prioridade alta.
- **Há risco comercial concreto.** A Atlas sinalizou que pode migrar para um concorrente se a entrega não sair até o fim do trimestre `[09:00]`. Prioridade alta.
- **A plataforma não tem nenhum mecanismo de notificação externa.** Não existe evento, fila ou webhook no sistema, então a feature parte do zero e precisa estabelecer o padrão. Prioridade alta.
- **Não há evidência do que foi comunicado.** O Product Manager pediu que o cliente consiga ver as últimas entregas feitas a ele, com sucesso ou falha, conteúdo, resposta e tempo de resposta `[09:34]`. Sem esse registro, não existe dado para mostrar ao cliente o que foi enviado. Prioridade média.

---

### Objetivos e métricas

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Substituir a consulta periódica por notificação para os clientes que pediram | Clientes B2B com endpoint ativo recebendo eventos | 3 clientes, até o fim de novembro |
| Notificar dentro do que o cliente considera tempo real | Tempo entre a mudança de status e a primeira tentativa de entrega | abaixo de 10 segundos |
| Não perder evento por indisponibilidade do destino | Eventos emitidos sem registro de entrega nem de falha permanente | zero |
| Tolerar indisponibilidade prolongada sem intervenção humana | Janela de repetição automática coberta a partir da primeira falha | quase 15 horas |
| Entregar dentro do compromisso comercial | Sprints até o deploy, com a revisão de segurança incluída no fim | 3 sprints |

Origem de cada meta. Os três clientes e o fim de novembro são o pedido da Atlas relatado pelo Product Manager `[09:45]`, sobre o risco de migração até o fim do trimestre `[09:00]`. Os 10 segundos são a expectativa que os clientes declararam ao Product Manager: para eles, qualquer coisa abaixo disso já é tempo real `[09:02]`. A janela de quase 15 horas é o resultado da progressão de repetição decidida na reunião `[09:17]`. O prazo de três sprints é a estimativa da Tech Lead `[09:46]` `[09:47]`.

A meta de zero eventos sem registro é a leitura, feita por este documento, de duas exigências da reunião: nenhuma mudança de status pode ser efetivada sem o evento correspondente registrado `[09:40]`, e todo evento que esgota as tentativas fica registrado com o motivo da falha `[09:18]`. A reunião não fixou taxa de entrega nem mediu baseline de perda, e este documento não cria nenhum.

---

### Escopo

Incluso

- Cadastro de endpoint de webhook por cliente, com URL de destino e lista de status assinados `[09:31]`
- Listagem, edição e remoção dos endpoints de um cliente `[09:33]`
- Filtro de eventos por status: o endpoint recebe apenas as transições que assinou `[09:33]`
- Geração da credencial de assinatura pela plataforma, devolvida ao cliente no momento do cadastro `[09:31]`
- Troca da credencial pela API, com a anterior válida em paralelo por 24 horas `[09:21]`
- Emissão do evento no momento exato da mudança de status, de forma indissociável dela `[09:40]`
- Entrega do evento por requisição HTTP assinada, com identificação de evento e horário de disparo `[09:44]`
- Repetição automática em caso de falha, com espaçamento crescente e teto de tentativas `[09:17]`
- Registro de falha permanente em fila dedicada, com reprocessamento manual por operação restrita e auditada `[09:18]` `[09:36]`
- Histórico das entregas feitas a um endpoint, com resultado, resposta e tempo `[09:34]`

Fora de escopo

- **Aviso ativo ao cliente quando a integração dele está falhando.** A proposta de enviar e-mail depois de falhas repetidas foi levantada e adiada para uma fase seguinte, condicionada a medir o impacto primeiro `[09:37]`
- **Painel visual para o cliente acompanhar os webhooks dele.** Nesta fase são apenas endpoints; o painel é projeto separado, do time de frontend `[09:40]`
- **Limitação de taxa de envio para um mesmo cliente.** Levantada como preocupação real e deixada em aberto, para observar e decidir depois `[09:39]`
- **Webhooks de entrada.** O escopo é apenas de saída `[09:02]`
- **Arquivamento das linhas já entregues.** Reconhecido como necessário e declarado fora do escopo desta feature `[09:08]`
- **Endurecimento das permissões do cadastro de webhook.** Por ora qualquer papel autenticado pode gerenciar a configuração; a restrição maior fica para depois `[09:37]`

---

### Requisitos funcionais

A prioridade de cada requisito é atribuída por este documento e não foi discutida na reunião. O critério aplicado: prioridade alta para o que precisa existir na primeira entrega para a feature cumprir o pedido do cliente, média para o que a torna operável e sustentável.

### RF-001 Cadastro de endpoint de webhook

O cliente registra uma URL de destino e a lista de status de pedido que quer receber. A credencial de assinatura é gerada pela plataforma e devolvida na resposta do cadastro `[09:31]`.

**Fluxo principal**

- Usuário autenticado envia a URL de destino, o cliente a que o cadastro pertence e a lista de status assinados
- A plataforma valida que a URL usa canal cifrado
- A plataforma gera a credencial de assinatura e grava o cadastro como ativo
- A resposta devolve o cadastro criado e a credencial, que não aparece em nenhuma consulta posterior

**Fluxos alternativos e exceções**

- O cliente pode cadastrar mais de um endpoint, cada um com sua própria lista de status e sua própria credencial `[09:44]`
- O cadastro é feito pela API da plataforma por um usuário operador autenticado, e não pelo painel do cliente. Por isso o cliente a que o cadastro pertence é informado na requisição, e não deduzido do token `[09:32]`

**Erros previstos**

- URL sem canal cifrado, recusada na validação `[09:23]`
- Lista de status vazia ou com valor que não existe na máquina de estados do pedido
- Cliente informado inexistente

**Prioridade:** alta

---

### RF-002 Consulta dos endpoints de um cliente

O cliente lista os endpoints cadastrados para ele, com a configuração de cada um `[09:33]`.

**Fluxo principal**

- Usuário autenticado consulta os endpoints de um cliente
- A plataforma devolve a lista com URL, status assinados e estado de cada cadastro

**Fluxos alternativos e exceções**

- A credencial de assinatura nunca aparece nesta resposta

**Erros previstos**

- Consulta sem autenticação

**Prioridade:** média

---

### RF-003 Edição de endpoint

O cliente altera a URL, a lista de status assinados ou o estado ativo de um cadastro `[09:33]`.

**Fluxo principal**

- Usuário autenticado envia os campos a alterar
- A plataforma valida os campos enviados e grava a alteração

**Fluxos alternativos e exceções**

- A alteração vale para transições futuras. Eventos já emitidos continuam sendo entregues conforme a assinatura vigente no momento em que foram criados
- Desativar o cadastro impede a emissão de eventos novos, mas não interrompe a entrega dos que já estavam registrados

**Erros previstos**

- URL nova sem canal cifrado
- Lista de status inválida

**Prioridade:** média

---

### RF-004 Remoção de endpoint

O cliente remove um cadastro de webhook `[09:33]`.

**Fluxo principal**

- Usuário autenticado solicita a remoção do cadastro
- A plataforma remove o cadastro e deixa de entregar eventos por ele

**Fluxos alternativos e exceções**

- Quem quer apenas interromper a emissão de eventos novos, sem apagar a configuração, desativa o cadastro em vez de removê-lo

**Erros previstos**

- Cadastro inexistente

**Prioridade:** média

---

### RF-005 Filtro de eventos por status assinado

Cada endpoint recebe apenas as transições que assinou. O filtro é aplicado no momento de registrar o evento: se nenhum endpoint quer aquela transição, o evento não chega a ser criado `[09:34]`.

**Fluxo principal**

- O status de um pedido muda
- A plataforma verifica quais endpoints ativos daquele cliente assinam o status de destino
- Um evento é registrado para cada endpoint interessado
- Se não há nenhum, nada é registrado

**Fluxos alternativos e exceções**

- Um cliente sem nenhum endpoint cadastrado não gera evento nenhum, e o comportamento da mudança de status permanece o que já era

**Erros previstos**

- Nenhum. A ausência de interessados é caminho normal e não é erro

**Prioridade:** alta

---

### RF-006 Troca da credencial de assinatura

O cliente pede uma credencial nova pela API. A anterior continua válida por 24 horas, para ele ter tempo de atualizar os sistemas dele antes de a antiga deixar de valer `[09:21]`. A operação atende ao caso de credencial exposta, situação que já ocorreu com um cliente da plataforma `[09:22]`.

**Fluxo principal**

- Usuário autenticado solicita a troca para um cadastro
- A plataforma gera a credencial nova, mantém a anterior registrada com prazo de validade e devolve a nova na resposta
- Passadas 24 horas, a anterior deixa de ser aceita

**Fluxos alternativos e exceções**

- Durante as 24 horas de sobreposição o cliente pode manter as duas credenciais configuradas em paralelo. Encerrado o prazo, apenas a nova é aceita `[09:21]`

**Erros previstos**

- Troca solicitada enquanto o período de sobreposição anterior ainda está em curso
- Cadastro inexistente

**Prioridade:** alta

---

### RF-007 Entrega do evento ao endpoint do cliente

A plataforma envia o evento por requisição HTTP ao endpoint cadastrado, assinada com a credencial daquele endpoint, para que o cliente possa verificar que a mensagem veio da plataforma e não foi alterada no caminho `[09:20]`.

**Fluxo principal**

- O evento registrado é enviado ao endpoint de destino
- A requisição carrega a assinatura do conteúdo, o identificador único do evento, o horário do disparo e a identificação do cadastro de origem `[09:44]`
- O conteúdo descreve a transição: identificação do evento e do pedido, número do pedido, status de origem e destino, cliente e valores básicos do pedido `[09:43]`
- Resposta de sucesso encerra a entrega daquele evento

**Fluxos alternativos e exceções**

- O conteúdo não inclui os itens do pedido, para não inflar a mensagem. O cliente que precisar de detalhe consulta a API de pedidos `[09:43]`
- O conteúdo reflete o estado do pedido no instante da transição, e não no instante do envio `[09:52]`
- O mesmo evento pode chegar mais de uma vez. O cliente reconhece a repetição pelo identificador do evento, que é estável entre tentativas `[09:25]`
- Cliente que não responde em 10 segundos é tratado como falha `[09:42]`

**Erros previstos**

- Conteúdo do evento acima de 64KB, caso em que o envio é recusado em vez de truncado `[09:23]` `[09:24]`
- Endereço inalcançável, conexão recusada ou tempo esgotado
- Resposta com código de erro do lado do cliente

**Prioridade:** alta

---

### RF-008 Repetição automática de entrega que falhou

Falha de entrega gera nova tentativa, com espaçamento crescente e teto de cinco tentativas `[09:16]` `[09:17]`.

**Fluxo principal**

- A entrega falha
- A plataforma agenda a tentativa seguinte com espaçamento crescente: 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas
- Qualquer tentativa bem-sucedida encerra o ciclo

**Fluxos alternativos e exceções**

- A janela total cobre quase 15 horas entre a primeira falha e a última tentativa, acima do caso real de indisponibilidade de duas horas em manutenção planejada que a plataforma já observou em um cliente `[09:16]`
- Esgotadas as cinco tentativas, o evento segue para o RF-009

**Erros previstos**

- Nenhum específico. A falha é a condição de entrada deste requisito

**Prioridade:** alta

---

### RF-009 Registro e reprocessamento de falha permanente

Evento que esgotou as tentativas vai para uma fila dedicada de falhas, com o conteúdo, o motivo e o horário, e pode ser recolocado na fila de entrega por operação manual `[09:18]`.

**Fluxo principal**

- O evento esgota a quinta tentativa e é movido para a fila de falhas permanentes, saindo da fila ativa
- Um usuário com papel administrativo consulta o item e aciona o reprocessamento
- O evento volta para a fila de entrega e passa por um ciclo novo de tentativas

**Fluxos alternativos e exceções**

- O reprocessamento é restrito a papel administrativo, porque mexer em fila de entrega não é operação de atendimento `[09:36]`
- A operação registra quem a executou, para auditoria `[09:36]`
- Nada reprocessa sozinho. A recuperação depende de alguém perceber e decidir

**Erros previstos**

- Reprocessamento solicitado por usuário sem papel administrativo
- Item inexistente ou já reprocessado

**Prioridade:** alta

---

### RF-010 Histórico de entregas por endpoint

O cliente consulta as entregas feitas ao endpoint dele, com resultado, conteúdo enviado, resposta recebida e tempo de resposta `[09:34]`.

**Fluxo principal**

- Usuário autenticado consulta o histórico de um endpoint
- A plataforma devolve as entregas mais recentes, com sucesso ou falha, resposta e duração

**Fluxos alternativos e exceções**

- Todas as tentativas de um mesmo evento aparecem agrupadas pelo identificador do evento, o que torna objetiva a investigação de uma reclamação
- É o único canal pelo qual o cliente descobre que a integração dele está com problema, já que não há aviso ativo nesta fase `[09:37]`

**Erros previstos**

- Cadastro inexistente

**Prioridade:** média

---

### Requisitos não funcionais

Performance

- O evento precisa chegar ao endpoint do cliente em menos de 10 segundos a partir da mudança de status, no caminho sem falha `[09:02]`
- A verificação de interessados e o registro do evento acontecem dentro da transação de mudança de status, que é caminho crítico da operação de pedidos, e por isso precisam ser mínimos `[09:04]`
- Tempo limite de 10 segundos para a resposta do endpoint do cliente `[09:42]`

Disponibilidade

- A entrega de eventos não pode depender do ciclo de vida da API. Reinício ou publicação da API não interrompe a entrega `[09:11]`
- A indisponibilidade do endpoint do cliente não pode impedir a mudança de status de um pedido `[09:04]`
- A entrega tolera indisponibilidade do destino por quase 15 horas sem intervenção `[09:17]`

Segurança e autorização

- Toda requisição enviada ao cliente é assinada com HMAC-SHA256 sobre o conteúdo `[09:20]`
- Cada endpoint tem credencial exclusiva. Não existe credencial única da plataforma, porque um vazamento comprometeria todos os clientes `[09:21]`
- A credencial é trocável pela API, com 24 horas de sobreposição `[09:21]`
- A URL de destino precisa usar canal cifrado, verificado na entrada `[09:23]`
- A credencial nunca aparece em log, em mensagem de erro ou em resposta que não seja a de criação ou a de troca
- O reprocessamento de falha permanente exige papel administrativo e registra o autor `[09:36]`
- A gestão do cadastro de webhook aceita qualquer papel autenticado nesta fase `[09:37]`

Observabilidade

- Toda tentativa de entrega é registrada com resultado, código de resposta e duração, e fica consultável pelo cliente `[09:34]`
- Toda falha permanente é registrada com o motivo, para diagnóstico sem depender de log `[09:18]`
- A profundidade da fila de eventos pendentes precisa ser observável. O processo de entrega roda separado da API `[09:11]`, então a parada dele não aparece em nenhum sinal da API

Confiabilidade e integridade de dados

- Não pode existir mudança de status commitada sem o evento correspondente registrado. Se o registro do evento falhar, a mudança de status é desfeita `[09:40]`
- A garantia de entrega é at-least-once: o evento sempre chega, e pode chegar mais de uma vez `[09:24]`
- O identificador do evento é único e estável entre todas as tentativas, e serve como chave para o cliente descartar repetição `[09:25]`
- O conteúdo do evento reflete o estado do pedido no instante da transição, mesmo que a entrega ocorra horas depois `[09:52]`
- Enquanto houver um único processo de entrega, os eventos de um mesmo pedido chegam na ordem em que aconteceram. Não há garantia de ordem global, e os clientes nunca pediram uma `[09:13]` `[09:14]`

Compatibilidade e portabilidade

- Nenhuma dependência nova entra no projeto `[09:29]`
- Nenhum endpoint existente muda de contrato. Os endpoints da feature são adição
- Mesmo banco e mesma stack do sistema atual. A única mudança de topologia é o segundo processo `[09:11]`

Compliance

- Nenhum requisito regulatório foi levantado na reunião, e este documento não cria nenhum. O dado transmitido é o dado de pedido do próprio cliente que recebe a notificação
- A exigência de trilha registrada vale para uma operação específica: o reprocessamento de falha permanente precisa registrar quem executou `[09:36]`

Acessibilidade no frontend consumidor

- Não se aplica nesta fase. A entrega é exclusivamente de API, sem interface visual, e o painel para o cliente está fora de escopo `[09:40]`

---

### Arquitetura e abordagem

Abordagem

- O evento é gravado em uma tabela intermediária dentro da mesma transação que muda o status do pedido, o que torna impossível existir uma sem a outra `[09:06]`
- A entrega acontece depois, em um processo separado que lê essa tabela em ciclo curto e faz as chamadas HTTP `[09:09]`
- A decisão de não usar infraestrutura de fila dedicada foi deliberada, para não acrescentar componente que o time precisaria operar `[09:07]`

Componentes

- Módulo de webhooks dentro do sistema existente, responsável pelo cadastro, pela consulta de histórico e pelo reprocessamento
- Processo separado de entrega, que lê os eventos pendentes e faz as chamadas ao cliente `[09:11]`
- Quatro tabelas novas no banco atual: configuração de endpoint, eventos a entregar, histórico de entregas e falhas permanentes

Integrações

- O serviço de pedidos passa a registrar o evento dentro da transação de mudança de status `[09:40]` `[09:41]`
- Os endpoints HTTP dos clientes são o destino das entregas
- O portal do desenvolvedor recebe a documentação de integração, incluindo a instrução de como verificar a assinatura e o aviso de que o mesmo evento pode chegar mais de uma vez `[09:26]` `[09:40]`

### Decisões e trade-offs

### Decisão: registrar o evento na mesma transação da mudança de status

- **Justificativa:** elimina a possibilidade de o status mudar sem o cliente ser notificado, ou o contrário, sem precisar de código de compensação `[09:06]`
- **Trade-off:** uma falha ao registrar o evento passa a desfazer uma mudança de status legítima, o que coloca o domínio de webhooks no caminho crítico da operação de pedidos. Detalhe em [ADR-001](adrs/ADR-001-outbox-no-mysql.md)

### Decisão: entregar em processo separado, lendo a tabela a cada 2 segundos

- **Justificativa:** isola a entrega do ciclo de vida da API e mantém a solução simples o suficiente para o time operar, dentro da folga que a meta de 10 segundos permite `[09:09]` `[09:11]`
- **Trade-off:** passa a existir um segundo processo em produção, com supervisão própria, e ele é ponto único de falha enquanto for único. Detalhe em [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md)

### Decisão: garantir entrega at-least-once, com deduplicação pelo cliente

- **Justificativa:** garantir entrega única exigiria coordenação entre a plataforma e o sistema do cliente, com custo desproporcional. É o comportamento que o mercado consolidou `[09:25]`
- **Trade-off:** transfere trabalho para o cliente, o que foi questionado explicitamente na reunião pela engenharia de segurança, e cria obrigação de documentação. Detalhe em [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)

### Decisão: credencial de assinatura por endpoint, e não da plataforma

- **Justificativa:** limita o dano de um vazamento a um único cliente, com precedente concreto de credencial exposta em log de cliente `[09:21]` `[09:22]`
- **Trade-off:** a plataforma passa a custodiar uma credencial por endpoint, com a obrigação permanente de nunca expô-la. Detalhe em [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)

### Decisão: filtrar os eventos no momento de registrar, e não no de enviar

- **Justificativa:** se nenhum endpoint assina aquela transição, o evento não precisa ser gravado, o que economiza linha na tabela `[09:34]`
- **Trade-off:** a plataforma perde a capacidade de reenviar histórico a um endpoint cadastrado depois, porque aquelas linhas nunca existiram. Detalhe em [ADR-008](adrs/ADR-008-filtro-de-eventos-na-insercao.md)

---

### Dependências

### Dependência organizacional: revisão de segurança antes do deploy

A engenharia de segurança reservou pelo menos dois dias úteis para revisar o código de assinatura e de geração de credencial antes da subida `[09:46]`. A revisão está incluída na estimativa de três sprints e é pré-requisito do deploy, não etapa opcional `[09:47]`.

### Dependência de produto: documentação de integração no portal do desenvolvedor

O Product Manager assumiu documentar, de forma destacada, que o mesmo evento pode chegar mais de uma vez e que o cliente precisa descartar repetição pelo identificador `[09:26]`, além da instrução geral de como integrar via API `[09:40]`. Sem isso, o cliente descobre o comportamento em produção.

### Dependência externa: preparo do lado do cliente

O cliente precisa expor um endpoint em canal cifrado `[09:23]`, verificar a assinatura recebida com a credencial que a plataforma forneceu `[09:20]` e descartar eventos repetidos pelo identificador `[09:25]`. A plataforma não tem como verificar se ele fez as três coisas.

### Dependência técnica: execução do processo de entrega

O segundo processo precisa ser empacotado, executado e supervisionado no ambiente. Ele é o primeiro processo de execução contínua do projeto, então não existe padrão anterior de operação para reaproveitar `[09:11]`.

---

### Riscos e mitigação

### Atraso na entrega e perda do cliente que motivou a feature

- **Probabilidade:** média
- **Impacto:** alto e comercial. A Atlas sinalizou possibilidade de migrar para um concorrente `[09:00]`
- **Mitigação:**
    - Prazo confirmado com o cliente pelo Product Manager logo após a reunião `[09:47]`
    - Estimativa feita por partes, com a revisão de segurança já incluída, e não acrescentada no fim `[09:46]`
    - Priorizar os requisitos de prioridade alta; os de prioridade média entregam valor mas não são o que o cliente pediu
- **Plano de contingência:** entregar primeiro a emissão, a entrega assinada e a repetição automática, que já cumprem o pedido original, deixando histórico e gestão do cadastro para uma segunda etapa

### Cliente que não descarta eventos repetidos

- **Probabilidade:** média
- **Impacto:** médio, e no domínio do cliente. O mesmo evento processado duas vezes pode gerar efeito duplicado no sistema dele
- **Mitigação:**
    - Identificador de evento estável entre tentativas, enviado em todas as entregas `[09:25]`
    - Documentação destacada do comportamento antes de o cliente integrar `[09:26]`
- **Plano de contingência:** o histórico de entregas permite mostrar ao cliente exatamente quantas vezes cada evento foi enviado, o que resolve a investigação sem depender do log dele

### Processo de entrega parado sem ninguém perceber

- **Probabilidade:** média
- **Impacto:** alto. Os eventos acumulam sem erro visível na API, e o cliente só percebe pela ausência de notificação
- **Mitigação:**
    - Acompanhar a profundidade da fila de eventos pendentes, que é o sinal que a API não dá
    - Supervisão do processo com reinício automático
- **Plano de contingência:** reiniciar o processo. Como o estado do trabalho vive no banco, ele retoma de onde parou, e a política de repetição cobre o período de parada dentro da janela de quase 15 horas

### Vazamento da credencial de assinatura

- **Probabilidade:** baixa na plataforma, com precedente conhecido do lado de cliente `[09:22]`
- **Impacto:** alto. Quem tem a credencial consegue forjar uma requisição que o cliente aceitaria como legítima
- **Mitigação:**
    - Credencial exclusiva por endpoint, o que limita o alcance de um vazamento a um cliente `[09:21]`
    - Credencial nunca registrada em log nem devolvida fora da criação e da troca
    - Revisão de segurança do código de assinatura e de geração antes do deploy `[09:46]`
- **Plano de contingência:** trocar a credencial do cadastro afetado. A sobreposição de 24 horas permite fazer isso sem combinar janela de indisponibilidade com o cliente `[09:21]`

### Integração quebrada que ninguém descobre

- **Probabilidade:** média
- **Impacto:** médio. Sem aviso ativo, o cliente só descobre consultando o histórico, e a plataforma só descobre olhando a fila de falhas permanentes
- **Mitigação:**
    - Registro de toda falha permanente com motivo, o que dá visibilidade objetiva de quais integrações estão quebradas `[09:18]`
    - Histórico de entregas disponível ao cliente como canal de diagnóstico `[09:34]`
- **Plano de contingência:** contato manual com o cliente a partir da fila de falhas permanentes, até que o aviso ativo entre em uma fase seguinte `[09:37]`

---

### Critérios de aceitação

Checklist objetivo que define se a feature está pronta.

- É possível cadastrar um endpoint com URL em canal cifrado e lista de status assinados, e a credencial de assinatura é devolvida apenas nessa resposta
- Uma URL sem canal cifrado é recusada no cadastro e na edição
- É possível listar, editar e remover os endpoints de um cliente
- É possível trocar a credencial de um endpoint, e a anterior continua sendo aceita por 24 horas
- Uma mudança de status de um pedido de cliente com endpoint assinante gera exatamente um evento por endpoint interessado
- Uma mudança de status sem nenhum endpoint interessado não gera evento nenhum
- Se o registro do evento falhar, a mudança de status não é efetivada
- O evento entregue carrega assinatura, identificador de evento, horário do disparo e identificação do cadastro de origem
- O conteúdo entregue reflete o estado do pedido no instante da transição, ainda que a entrega ocorra horas depois
- O identificador do evento é o mesmo em todas as tentativas de entrega daquele evento
- Um endpoint que não responde em 10 segundos é tratado como falha e entra em nova tentativa
- Cinco falhas consecutivas movem o evento para a fila de falhas permanentes, com o motivo registrado
- O reprocessamento de falha permanente só é aceito de usuário com papel administrativo, e registra quem executou
- O histórico de entregas mostra, para cada tentativa, resultado, resposta recebida e tempo de resposta
- A credencial de assinatura não aparece em nenhum log, mensagem de erro ou resposta de consulta
- Nenhum endpoint já existente da API mudou de comportamento ou de contrato

---

### Testes e validação

Tipos de teste obrigatórios

- Testes de ponta a ponta dos endpoints da feature, no mesmo padrão dos testes que o projeto já tem para autenticação e para pedidos, que rodam contra a aplicação e o banco reais
- Teste de atomicidade: mudar o status de um pedido com endpoint assinante e verificar o evento registrado; forçar falha no registro do evento e verificar que o status não mudou
- Teste do filtro: mudar o status de um pedido sem endpoint assinante e verificar que nenhum evento foi criado
- Teste da política de repetição: verificar o espaçamento entre tentativas e a movimentação para a fila de falhas permanentes na quinta falha
- Teste de autorização: reprocessamento recusado para usuário sem papel administrativo
- Teste de assinatura: recalcular a assinatura sobre o conteúdo recebido, com a credencial do cadastro, e confirmar que confere
- Teste de exposição de credencial: verificar que a credencial não aparece em consulta, em edição, em mensagem de erro nem em log

Estratégia de validação

- Revisão do desenho técnico com os engenheiros antes de começar a implementação, conforme combinado na reunião `[09:50]`
- Revisão de segurança do código de assinatura e de geração de credencial pela engenharia de segurança, antes do deploy `[09:46]`

As duas ações abaixo são proposta deste documento e não foram discutidas na reunião.

- Validação com um dos três clientes antes da liberação geral, usando o histórico de entregas como evidência do que foi enviado e recebido
- Acompanhamento, na primeira semana, da profundidade da fila de pendentes e da fila de falhas permanentes, que revelam problema de entrega sem depender de reclamação do cliente
