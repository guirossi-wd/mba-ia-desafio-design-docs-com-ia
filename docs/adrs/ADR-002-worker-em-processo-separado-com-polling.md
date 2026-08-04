# ADR-002: Consumo da outbox por worker em processo separado com polling de 2 segundos

**Status:** Aceito
**Data:** 2026-07-29
**ADRs Relacionados:** ADR-001, ADR-003, ADR-006, ADR-007, ADR-008

## Contexto e Declaração do Problema

A decisão do ADR-001 garante que todo evento de mudança de status fica registrado na outbox de forma atômica, mas não diz quem lê essa tabela nem quando.

Duas questões independentes precisavam de resposta. A primeira é **como o worker descobre que existe evento novo**: o banco avisa, ou o worker pergunta. A segunda é **onde esse worker roda**: dentro do processo da API que já existe, ou em um processo próprio.

Na primeira questão, Bruno perguntou em `[09:09]` se não daria para usar trigger de banco, para o desenho ficar reativo em vez de baseado em consulta repetida.

Diego respondeu no mesmo minuto com a limitação concreta: o MySQL não tem mecanismo nativo de notificação para processo externo, diferente do PostgreSQL. Trigger no MySQL executa SQL, e só. Para avisar um processo de fora seria preciso improvisar algo como escrever em arquivo ou chamar um endpoint, o que ele classificou como esquisito.

Na segunda questão, Diego foi enfático em `[09:11]`: o worker tem que rodar como processo separado, porque se ele vive dentro da instância da API, um restart da API derruba a entrega de eventos junto.

O ponto de partida do projeto pesa nessa escolha. Hoje existe um único ponto de entrada, que sobe o servidor HTTP e encerra o cliente de banco no desligamento, e nenhum processo de execução contínua. O worker da outbox vai ser o primeiro processo do projeto que não atende requisição, e por isso o desenho escolhido aqui estabelece um padrão novo.

Existe ainda uma terceira questão, levantada por Larissa em `[09:12]`, que a escolha do worker determina: se um pedido muda de status três vezes em sequência rápida, o cliente recebe as notificações na ordem certa.

## Fatores de Decisão

- O MySQL não oferece notificação nativa para processo externo.
- A meta de latência é abaixo de 10 segundos, o que dá folga grande para uma abordagem baseada em consulta periódica.
- O ciclo de vida da entrega de eventos não pode estar preso ao ciclo de vida da API.
- O projeto já tem um ponto de entrada de processo estabelecido, que serve de modelo para um segundo análogo.
- Ordenação global nunca foi pedida pelos clientes.
- Não existe no projeto nenhum processo de execução contínua, então a solução precisa ser simples o suficiente para o time operar sem ferramental novo.

## Alternativas Consideradas

1. Worker em processo separado, lendo a outbox em ciclo de 2 segundos.
2. Worker dentro do próprio processo da API, em intervalo interno.
3. Trigger de banco notificando o worker de forma reativa.

## Resultado da Decisão

Opção escolhida: **worker em processo separado, com ciclo de leitura de 2 segundos**. Diego propôs a cadência em `[09:09]`, Marcos validou o número em `[09:10]` e Larissa registrou como decisão no mesmo minuto, aceitando explicitamente que a entrega passa a ter latência de até 2 segundos.

Sobre o isolamento de processo, Larissa propôs em `[09:11]` um segundo ponto de entrada no projeto, análogo ao que já existe para subir a API, acompanhado de um script próprio de execução. Bruno acrescentou no mesmo minuto que esse processo se conecta ao mesmo banco, e Diego fechou: mesmo banco e mesma stack, apenas processo diferente.

Um detalhe de infraestrutura ficou decidido mais adiante, em `[09:30]`: o worker abre o seu próprio cliente de banco, e não compartilha o da API, porque o cliente é por processo. Mesma string de conexão, instância nova.

Sobre ordenação, Diego explicou em `[09:12]` que com um único worker os eventos são processados na ordem de criação, o que faz o cliente receber na ordem correta por pedido. Com múltiplos workers em paralelo essa garantia se perde. Larissa registrou o desenho em `[09:13]` como limitação conhecida e não como garantia: ordenação por pedido, e somente enquanto o worker for único. Particionar por pedido ou usar lock pessimista resolveria a escala, e ficou classificado como problema do futuro.

## Prós e Contras das Alternativas

### Worker em processo separado com ciclo de 2 segundos

- Boa, porque isola completamente falha e reinício da API da entrega de eventos.
- Boa, porque a implementação é trivial e não depende de recurso de banco, extensão ou biblioteca nova.
- Boa, porque um único worker entrega ordenação por pedido sem nenhum mecanismo de coordenação.
- Boa, porque o processo pode ser reiniciado a qualquer momento sem efeito sobre o tráfego HTTP, já que o estado do trabalho vive na tabela e não em memória.
- Ruim, porque consulta o banco continuamente mesmo quando não há evento pendente, gerando carga constante e em boa parte inútil.
- Ruim, porque introduz latência de até 2 segundos entre o commit e o início da entrega.

### Worker dentro do processo da API

- Boa, porque não acrescenta processo novo para empacotar, subir e monitorar.
- Boa, porque reaproveita o cliente de banco já instanciado pela API, sem nova conexão.
- Ruim, porque um restart ou deploy da API interrompe a entrega de eventos.
- Ruim, porque múltiplas instâncias da API produziriam múltiplos workers concorrentes, quebrando a ordenação sem que ninguém tenha decidido isso.
- Ruim, porque o trabalho de entrega competiria por CPU e pool de conexões com o tráfego HTTP dos usuários.

### Trigger de banco notificando o worker

- Boa, porque em teoria elimina a consulta repetida e entrega latência próxima de zero.
- Ruim, porque o MySQL não tem o mecanismo de notificação que tornaria isso viável, ao contrário do PostgreSQL, que oferece `LISTEN` e `NOTIFY` para exatamente esse fim.
- Ruim, porque qualquer improviso para contornar a limitação, como trigger escrevendo em arquivo ou chamando endpoint, coloca efeito colateral externo dentro de uma transação de banco.
- Ruim, porque colocaria regra de entrega no banco, longe do código da aplicação, o que piora depuração e teste sem trazer ganho de confiabilidade.

## Consequências

- Boa, porque a entrega de eventos sobrevive a deploy e restart da API, e pode ser reiniciada de forma independente.
- Boa, porque a latência de entrega fica em até 2 segundos, folgada frente à meta de 10 segundos, o que dá margem para o tempo de retry e para lentidão do cliente sem furar o requisito.
- Boa, porque o worker único torna desnecessário qualquer mecanismo de coordenação entre leitores, o que elimina uma classe inteira de defeito de concorrência nesta entrega.
- Ruim, porque passa a existir um segundo processo em produção, com necessidade própria de empacotamento, supervisão, log e alarme, custo operacional que antes não existia.
- Ruim, porque o worker único é ponto único de falha: se o worker morre e ninguém percebe, os eventos acumulam silenciosamente na outbox sem nenhum erro visível na API.
- Ruim, porque não existe garantia de ordenação global, apenas por pedido, e essa garantia depende de o worker permanecer único.
- Neutra, porque a escala horizontal do worker fica bloqueada por desenho: crescer o número de workers exige antes decidir entre particionamento por pedido e lock pessimista, questão deixada em aberto na reunião. Quando essa decisão for tomada, ela substitui este registro.
- Neutra, porque a consulta periódica troca eficiência por simplicidade de forma consciente, e a troca é aceitável apenas porque o MySQL não oferece notificação para processo externo.

## Referências

- `src/server.ts:6` ponto de entrada existente que serve de modelo para o processo do worker
- `src/config/database.ts:4` criação do cliente de banco, usada pelo worker para abrir instância própria
- `src/config/database.ts:10` instância única por processo, que é o motivo de o worker abrir a sua
- `src/modules/orders/order.status.ts:3` matriz de transições de status, que define quais eventos existem para entregar
- `package.json:11` padrão de scripts de execução do projeto
- `TRANSCRICAO.md` `[09:02]` Marcos: meta de latência abaixo de 10 segundos
- `TRANSCRICAO.md` `[09:08]` Larissa: abre a questão de como o worker lê a outbox
- `TRANSCRICAO.md` `[09:09]` Bruno e Diego: trigger de banco proposto e descartado, cadência de 2 segundos proposta
- `TRANSCRICAO.md` `[09:10]` Marcos e Larissa: validação do intervalo e registro da decisão
- `TRANSCRICAO.md` `[09:11]` Diego, Larissa e Bruno: isolamento de processo e segundo ponto de entrada
- `TRANSCRICAO.md` `[09:12]` Larissa e Diego: pergunta de ordenação e resposta com worker único
- `TRANSCRICAO.md` `[09:13]` Bruno, Diego e Larissa: escala adiada e limitação de ordenação registrada
- `TRANSCRICAO.md` `[09:14]` Marcos: clientes nunca pediram ordenação global
- `TRANSCRICAO.md` `[09:29]` Diego e `[09:30]` Bruno: cliente de banco próprio para o worker
- [LISTEN e NOTIFY no PostgreSQL, Cybertec](https://www.cybertec-postgresql.com/en/listen-notify-automatic-client-notification-in-postgresql/)
