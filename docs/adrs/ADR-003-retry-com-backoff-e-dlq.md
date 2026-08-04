# ADR-003: Tratamento de indisponibilidade do cliente com backoff exponencial de 5 tentativas e DLQ persistida

**Status:** Aceito
**Data:** 2026-07-29
**ADRs Relacionados:** ADR-002, ADR-005, ADR-007

## Contexto e Declaração do Problema

O worker do ADR-002 entrega eventos para endpoints que estão fora da infraestrutura da plataforma. Falha de entrega não é exceção, é operação normal: o cliente pode estar em manutenção, com instabilidade de rede ou com o endpoint quebrado.

Larissa abriu o tema em `[09:14]` com a pergunta direta: se o cliente está offline, o que a plataforma faz. Três decisões estavam embutidas nessa pergunta.

A primeira é **quantas vezes tentar**. Não tentar de novo perde o evento na primeira instabilidade de rede. Tentar para sempre cria um problema diferente: o evento fica pendurado indefinidamente se o cliente simplesmente desapareceu.

A segunda é **com que espaçamento**. Tentativas imediatas e seguidas não dão tempo de o cliente se recuperar e ainda somam carga sobre um sistema que já está com problema.

A terceira é **o que fazer quando as tentativas acabam**. O evento precisa ir para algum lugar onde possa ser diagnosticado e reprocessado, sem ficar sujo na fila principal atrapalhando a leitura dos eventos pendentes.

O que conta como falha ficou definido mais adiante na reunião: o tempo limite da chamada ao cliente é de 10 segundos, e cliente que não responde nesse prazo é tratado como falha sujeita a retry. Isso significa que a política aqui decidida governa tanto erro explícito quanto ausência de resposta.

Uma discussão concreta ancorou o dimensionamento. Bruno propôs 3 tentativas em `[09:16]`, argumentando que seria mais agressivo.

Diego rejeitou no mesmo minuto com evidência de operação: três tentativas cobririam cerca de 30 minutos, e a plataforma já teve cliente com indisponibilidade de duas horas em manutenção planejada.

## Fatores de Decisão

- Falha de entrega é esperada e não pode significar perda de evento.
- A janela de retry precisa cobrir indisponibilidade real de cliente, e há caso concreto de duas horas na operação da plataforma.
- Evento não pode ficar pendurado para sempre quando o cliente desaparece.
- A leitura da outbox principal precisa continuar limpa, para não degradar o ciclo do worker.
- Falha permanente precisa ser diagnosticável e reprocessável.
- Mexer em fila de entrega é operação sensível e precisa de trilha de auditoria.

## Alternativas Consideradas

1. Cinco tentativas com backoff exponencial e DLQ em tabela separada.
2. Retry indefinido com backoff, sem teto de tentativas.
3. Marcar o evento como falho na própria outbox, sem tabela de DLQ.

## Resultado da Decisão

Opção escolhida: **cinco tentativas com backoff exponencial em 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, seguidas de movimentação para uma DLQ em tabela separada**. Diego propôs a progressão em `[09:17]`, Marcos aceitou a janela resultante no mesmo minuto observando que um cliente fora do ar por 15 horas já tem problema próprio grave, e Larissa registrou a decisão no mesmo minuto.

A progressão soma quase 15 horas entre a primeira falha e a última tentativa, o que cobre com margem o caso de duas horas que motivou a rejeição das 3 tentativas.

Sobre o destino da falha permanente, Diego defendeu a tabela separada em `[09:18]`, guardando o conteúdo do evento, o motivo da falha e o horário, com dois argumentos: mantém a leitura da outbox principal limpa e cria registro dedicado para diagnóstico e reprocessamento. Bruno concordou no mesmo minuto.

O reprocessamento é manual e acionado por um endpoint administrativo que recoloca o item na outbox como pendente. Sofia exigiu em `[09:36]` que a operação seja restrita a papel administrativo, porque mexer em fila de entrega de notificação não é operação de atendimento, e que a ação registre quem a executou para fins de auditoria. Larissa fechou no mesmo minuto, determinando o reuso do mecanismo de autorização por papel que já existe no projeto.

## Prós e Contras das Alternativas

### Cinco tentativas com backoff exponencial e DLQ separada

- Boa, porque a janela de quase 15 horas cobre indisponibilidade planejada e não planejada de cliente com folga confortável.
- Boa, porque o teto de tentativas garante que nenhum evento fica em ciclo infinito consumindo recurso do worker.
- Boa, porque a tabela dedicada de falha permanente preserva o conteúdo e o motivo, tornando o diagnóstico possível sem consultar log.
- Boa, porque separar a falha permanente da fila ativa mantém a consulta de eventos pendentes previsível, independente do volume acumulado de falhas.
- Ruim, porque cria uma segunda tabela e um fluxo administrativo de reprocessamento que precisam ser mantidos e testados.
- Ruim, porque um cliente indisponível além da janela de quase 15 horas perde o evento da fila ativa, e a recuperação passa a exigir ação humana.

### Retry indefinido com backoff

- Boa, porque nenhum evento é dado como perdido, e um cliente que voltou depois de dias ainda recebe.
- Boa, porque dispensa tabela de falha permanente e todo o fluxo administrativo que vem com ela.
- Ruim, porque cliente que desapareceu de vez gera acumulação permanente na fila.
- Ruim, porque sem teto não existe o momento em que alguém é notificado de que a integração está quebrada, e a falha vira invisível.

### Marcar como falho na própria outbox

- Boa, porque não cria estrutura nova, e o histórico do evento fica em um só lugar.
- Boa, porque o reprocessamento seria apenas uma mudança de estado, sem cópia entre tabelas.
- Ruim, porque a tabela consultada a cada 2 segundos pelo worker cresce com linhas que nunca mais serão processadas, degradando a leitura.
- Ruim, porque mistura na mesma estrutura a fila operacional e o registro forense de falha, que têm ciclos de vida e padrões de acesso diferentes.

## Consequências

- Boa, porque a plataforma tolera indisponibilidade prolongada de cliente sem intervenção humana e sem perda de evento dentro da janela.
- Boa, porque a DLQ dá visibilidade objetiva de quais integrações estão quebradas, o que serve como métrica de saúde por cliente.
- Boa, porque o reprocessamento restrito a papel administrativo e com registro de quem executou atende a exigência de auditoria sem construir mecanismo novo de permissão.
- Ruim, porque a recuperação de um item da DLQ é manual: alguém precisa perceber, decidir e acionar, o que significa que a plataforma depende de vigilância humana para fechar o ciclo.
- Ruim, porque um evento pode chegar ao cliente quase 15 horas depois do fato, e nesse intervalo o pedido pode ter mudado de status várias vezes, o que torna o evento tecnicamente correto mas operacionalmente desatualizado. O ADR-007 registra a decisão que faz o conteúdo refletir o instante original.
- Ruim, porque a progressão é fixa e igual para todos os eventos, sem componente aleatório. Quando um cliente fica indisponível e muitos eventos falham juntos, todos passam a tentar de novo nos mesmos instantes, concentrando picos de requisição em cima de um sistema que está justamente se recuperando.
- Ruim, porque o tempo limite de 10 segundos por tentativa é o tempo que um cliente lento pode reter o ciclo do worker a cada tentativa, e os demais eventos pendentes daquele ciclo esperam esse prazo antes de serem processados.
- Neutra, porque a decisão de não alertar o cliente sobre falhas repetidas foi tomada em outro momento da reunião, com o aviso por e-mail colocado fora do escopo desta fase. A consequência é que o cliente só descobre que sua integração está quebrada consultando o histórico de entregas.

## Referências

- `src/middlewares/auth.middleware.ts:49` mecanismo de autorização por papel a ser reutilizado no reprocessamento
- `src/modules/users/user.routes.ts:15` único uso atual de restrição por papel administrativo, que serve de modelo
- `src/shared/logger/index.ts:13` criação do logger estruturado onde o registro de auditoria do reprocessamento será emitido
- `src/middlewares/request-logger.middleware.ts:14` padrão de log estruturado com evento nomeado, a ser seguido pelo registro de auditoria
- `TRANSCRICAO.md` `[09:14]` Larissa: abre o tema de indisponibilidade do cliente
- `TRANSCRICAO.md` `[09:15]` Diego e Bruno: backoff com teto proposto, retry indefinido descartado
- `TRANSCRICAO.md` `[09:16]` Bruno, Diego e Larissa: proposta de 3 tentativas rejeitada com caso de 2 horas, fechamento em 5
- `TRANSCRICAO.md` `[09:17]` Diego, Marcos e Larissa: progressão do backoff, aceite da janela e registro da decisão
- `TRANSCRICAO.md` `[09:18]` Diego e Bruno: DLQ em tabela separada e reprocessamento por endpoint administrativo
- `TRANSCRICAO.md` `[09:35]` Diego e Larissa: rota de reprocessamento e pergunta sobre permissão
- `TRANSCRICAO.md` `[09:36]` Sofia e Larissa: exigência de papel administrativo e de registro de auditoria
- `TRANSCRICAO.md` `[09:37]` Marcos e Larissa: aviso por e-mail colocado fora do escopo desta fase
- `TRANSCRICAO.md` `[09:42]` Sofia e Diego: tempo limite de 10 segundos por tentativa
- [Dead letter queue pattern e recuperação de poison message](https://www.abstractalgorithms.dev/dead-letter-queue-pattern-poison-message-recovery)
- [Retry storms, backoff exponencial e jitter](https://web-alert.io/blog/retry-storms-exponential-backoff-jitter-explained)
