# Da Reunião ao Documento: Design Docs Gerados por IA

Este repositório é a minha entrega do desafio de design docs do MBA. O enunciado original, que ocupava este arquivo, está preservado em [`docs/DESAFIO.md`](docs/DESAFIO.md). O que você lê aqui é o processo que produziu o pacote.

## Sobre o desafio

O ponto de partida são dois artefatos e uma restrição. Os artefatos: a transcrição literal de uma reunião técnica de 55 minutos, em que cinco pessoas decidem como construir um sistema de webhooks de notificação de pedidos, e a codebase de um Order Management System em produção, onde essa feature ainda não existe. A restrição: nada do que aparecer na documentação pode ser inventado. Todo requisito, decisão e limitação precisa vir de uma fala da reunião ou de uma linha do código.

A tarefa é transformar isso em um pacote de seis documentos que operam em alturas diferentes: um PRD de produto, um RFC de arquitetura, oito ADRs de decisão pontual, um FDD de implementação, um tracker de rastreabilidade e este README. O trabalho real não é escrever, é decidir o que entra. A reunião contém decisões fechadas, mas também ideias descartadas, pontos adiados, dúvidas nunca respondidas e detalhes secundários. Identificar o que **não** entra é metade do exercício, e é onde a IA erra com mais confiança.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude Code** (app desktop, Windows) | Ambiente único de trabalho. Acesso direto ao filesystem: leitura integral do codebase, produção dos documentos, execução de `git` e `gh`. Nada foi copiado e colado entre ferramentas. |
| **Claude Opus 5** | Modelo de análise e produção. Mineração da transcrição, mapeamento do código, decisões de arquitetura documental e redação de todos os documentos. |
| **Claude Sonnet 5** | Usado só na fase mecânica inicial: fork, clone, criação de estrutura de pastas, ajuste do `.gitignore`. Troquei para Opus quando o trabalho deixou de ser mecânico. |
| **Subagentes do Claude Code** | Revisão crítica. Cada documento foi revisado por um agente separado, com contexto próprio e instruções adversariais. É a peça mais produtiva do processo, explicada abaixo. |
| **GitHub CLI (`gh`)** | Fork do repositório base. |

## Workflow adotado

A ordem de produção segue a sugerida pelo enunciado, e a razão dela ficou clara na prática: **as decisões são o esqueleto**. Escrever os ADRs primeiro obriga a resolver, uma a uma, cada escolha da reunião. Os documentos seguintes se apoiam neles em vez de reabrir o assunto.

```
Contextualização  ->  ADRs  ->  RFC  ->  FDD  ->  PRD  ->  Tracker  ->  README
```

O princípio que organiza tudo: **separar mineração de redação**. Antes de escrever qualquer documento, produzi material de apoio que não faz parte da entrega: extração da transcrição com cada decisão, descarte e adiamento marcado com horário e falante; mapeamento do codebase com caminho e número de linha; e um destilado dos formatos exigidos pelo curso. Pedir "gere um PRD a partir dessa transcrição" sem essa etapa é o que produz documento genérico. O tracker denuncia na hora: a coluna Localização fica vazia.

Duas regras que apliquei em todos os ciclos:

**Todo documento passa por um revisor separado.** Não pedi ao mesmo contexto que produziu o texto para revisá-lo. Abri um subagente com contexto limpo, instruções adversariais e permissão de escrita. O resultado justifica: duas rodadas de revisão sobre os 8 ADRs, depois 15 defeitos corrigidos no RFC, 27 no FDD e 16 no PRD.

**Eu verifico o revisor.** Toda correção fundamentada em citação foi conferida por mim na fonte antes de aceitar. Em um caso o revisor estava certo sobre a regra e errado sobre o mérito, e eu reverti a correção dele (detalhado abaixo).

## Prompts customizados

### 1. Contextualização profunda, antes de escrever qualquer coisa

Este foi o primeiro prompt do projeto e o que mais determinou a qualidade do resto. O texto abaixo é a versão editada dele, com a mesma estrutura e as mesmas exigências do original, reescrita para leitura.

```
Você é um especialista em documentação técnica, metódico e cético.

Antes de escrever qualquer documento da entrega, faça a contextualização:

1. Leia o enunciado completo (README.md) e liste as exigências verificáveis,
   separando critério de aceite de recomendação.
2. Leia TRANSCRICAO.md integralmente e produza uma extração em que cada item
   tenha: o que foi dito, quem disse, o timestamp, e o status exato
   (decidido / descartado com justificativa / adiado / levantado e não decidido).
3. Varra o codebase inteiro e mapeie os pontos que a feature vai tocar, com
   caminho de arquivo e número de linha.

Produza isso como arquivos persistentes em material-base/, não como resposta
de chat. Registre o processo desde o começo, porque ele é parte da entrega.

Leve o tempo que for necessário.
```

O que fez esse prompt funcionar: definiu o papel, nomeou as três fontes a cruzar, exigiu artefato persistente em vez de resposta de conversa, pediu o registro do processo desde o início e removeu a pressa. A instrução de separar "descartado" de "adiado" de "não decidido" na extração é o que sustentou, depois, as seções de fora de escopo e de questões em aberto.

### 2. Revisor crítico com permissão de corrigir

Este é o prompt de revisão na forma que passei a usar do RFC em diante, reaproveitada depois no FDD e no PRD. Ele evoluiu de uma versão anterior, usada nos ADRs, que só apontava defeito e não corrigia.

```
Você é revisor técnico crítico. Revise <documento> e CORRIJA os defeitos
que encontrar diretamente no arquivo.

Não elogie e não tente agradar. Não reescreva o que já está correto. Seu valor
está em achar defeito, não em declarar aprovação. Se não achar defeito em uma
dimensão, diga isso em uma linha e siga.

Verifique, nesta ordem de importância:

1. FIDELIDADE. Para cada afirmação, verifique a origem na transcrição ou no
   código. Procure por: número que não aparece na reunião; [hh:mm] cuja fala
   não corresponde ao que o documento afirma (confira minuto e pessoa um por
   um); item descartado ou adiado reaparecendo como requisito; superlativo ou
   juízo de valor sem fonte.
2. EXATIDÃO CONTRA O CÓDIGO. Abra cada arquivo citado e confira linha por
   linha. Erro de número de linha, de nome de símbolo ou de descrição de
   comportamento é o defeito mais grave, porque o documento se propõe a ser
   acionável.
3. CRITÉRIOS DE ACEITE do enunciado, item por item.
4. COERÊNCIA com o resto do pacote.
5. ALTURA. Aponte trecho que desceu para o nível de detalhe de outro documento.

Ao corrigir:
- Afirmação sem fonte: remova, ou reescreva para dizer apenas o que a fonte
  sustenta. NÃO invente uma fonte para salvar a frase, e NÃO substitua por
  outra afirmação igualmente sem fonte.
- Prefira cortar a alongar.

Ao final, relate: tabela de defeitos com trecho, tipo, o que a fonte diz e a
correção aplicada; verificação dos critérios de aceite; o que verificou e
estava correto, com quantidade conferida; e o que continua frágil e você
deliberadamente NÃO corrigiu, com o motivo.
```

Três instruções carregam o peso. A primeira é "não elogie": revisor com permissão de escrita tende a reescrever o que já está certo para justificar o esforço. A segunda é a regra de como corrigir, porque sem ela a correção de uma alucinação costuma ser outra alucinação. A terceira é o pedido do que ficou em aberto e não foi corrigido, que produziu as observações mais honestas de todos os relatórios, incluindo autocrítica do próprio revisor.

### 3. Expurgo de conteúdo autoral

Este prompt resolveu o problema mais difícil do projeto, descrito na seção seguinte.

```
Ordem de prioridade, nesta ordem e sem negociação:
1. Fidelidade absoluta à reunião.
2. Coerência entre os documentos.
3. Tamanho. O piso de 100 linhas está REVOGADO.

Teste de admissão de cada frase: ela só fica se tiver uma fala com timestamp
que você conferiu abrindo a transcrição, ou um caminho:linha que você conferiu
abrindo o arquivo. Não vale "é plausível" nem "é verdade sobre o tema".

Remova especificamente estas alternativas, que não foram discutidas na reunião:
<lista com ADR, título da alternativa e motivo>

Quatro ADRs vão ficar com 2 alternativas em vez de 3. Isso é correto e
desejado. Não invente uma terceira para preencher.

Se para preservar o tamanho você precisaria inventar ou repetir, PARE.
Deixe o arquivo menor.
```

## Iterações e ajustes

Foram **14 ciclos registrados** até o fim do PRD, distribuídos em 6 fases: setup, contextualização, ADRs, RFC, FDD e PRD. Cada um dos quatro documentos grandes passou por pelo menos uma revisão crítica, e os ADRs passaram por duas. O tracker e este README vieram depois, com uma revisão própria cada. O registro completo, ciclo a ciclo, está no diário de bordo que mantive fora da entrega. Os momentos que mais mudaram o resultado:

### 1. Duas instruções minhas eram incompatíveis, e a IA satisfez a errada

O formato de ADR que adotei, tirado do material do curso, impõe entre 100 e 250 linhas. A regra do desafio proíbe inventar. Escrevendo os oito ADRs, os arquivos saíram com 77 a 91 linhas, porque **a reunião não dá material para mais do que isso**. Na revisão seguinte, pedindo conformidade de formato, os arquivos chegaram a 100 a 105 linhas. Fui ver como: quatro alternativas técnicas que ninguém discutiu na reunião tinham sido escritas do zero, com prós e contras convincentes.

O diagnóstico importa mais que a correção: entre uma restrição **contável** (número de linhas) e uma **não verificável automaticamente** (nada inventado), o modelo satisfaz a contável. Não por má-fé, por gradiente. Só destravou quando eu decidi qual das duas cede, com o prompt de expurgo acima. As quatro alternativas saíram, quatro ADRs ficaram com duas alternativas em vez de três, e os arquivos voltaram para 90 a 104 linhas.

### 2. A regra estava sendo cumprida, e por isso ninguém reclamava

Esse mesmo formato de ADR proíbe o campo de cabeçalho que registraria quem decidiu. Minha solução foi realocar essa informação para a prosa, com o horário. Correta como decisão. A execução passou do ponto: alguns ADRs chegaram a **21 citações de horário em 100 linhas**, uma a cada cinco linhas. O documento virava registro de auditoria em vez de registro de decisão.

Percebi ao me perguntar o que alguém veria abrindo aquilo em seis meses: nomes e horários sem contexto. Apliquei uma regra nova, que o horário só fica quando a frase é sobre alguém decidir, propor ou objetar, e sai quando a frase apenas afirma um fato técnico. As citações caíram para 3 a 9 por arquivo, com a procedência completa preservada na seção de referências. E reescrevi o `README.md` de `docs/adrs/`, que vinha do repositório base com três linhas genéricas, para explicar o que `[09:00]` significa, que era a lacuna real.

Nenhum verificador automático apontaria isso, porque a regra estava sendo obedecida.

### 3. Cada documento tem o seu formato preferido de alucinação

Este foi o padrão mais útil que o projeto revelou, e só apareceu porque houve revisão adversarial em todos os documentos.

- **No ADR**, a invenção preenche forma: alternativa criada para completar a lista de três.
- **No RFC**, a invenção é plausível demais. Eu havia escrito um risco sobre o cliente reserializar o conteúdo antes de verificar a assinatura, o que invalidaria a verificação. É verdade sobre HMAC, é um problema real de integração de webhook, e **ninguém falou disso na reunião**. Veio de conhecimento geral. Um revisor humano experiente leria e concordaria.
- **No PRD**, a invenção é a métrica com meta redonda: "100 por cento dos eventos entregues ou registrados como falha". Parece rastreável, porque a reunião de fato exige que nenhum evento se perca. Mas ninguém fixou taxa e ninguém mediu baseline. Virou "eventos emitidos sem registro: zero", com a origem da leitura declarada.

Nos três casos o texto passa no teste de leitura e falha no de origem.

### 4. O defeito que só apareceu ao abrir a implementação da classe

No FDD, especifiquei que os erros de 400 da feature estenderiam a classe de erro de validação do projeto. O caminho existe, a classe existe, e a frase parecia certa. Mas essa classe **fixa o código de erro no construtor e não aceita código do chamador**: os códigos com prefixo `WEBHOOK_` que estavam na matriz do documento sairiam como `VALIDATION_ERROR` na resposta real, quebrando um critério de aceite do desafio.

Nenhuma verificação de rastreabilidade pegaria isso, porque a referência era válida. Só apareceu porque o revisor abriu o arquivo e leu a implementação em vez de confirmar que o nome existia.

### 5. O documento seguinte revisa o anterior

Ao escrever o FDD, o revisor encontrou um defeito no RFC já finalizado e revisado: ele afirmava "nenhuma alteração no logger", enquanto o seu próprio risco de vazamento de credencial manda estender a lista de campos omitidos do log. Contradição interna que a revisão do próprio RFC não pegou.

O mesmo aconteceu na direção contrária no PRD, onde uma afirmação sobre remoção em cascata contradizia o FDD. Produzir o documento seguinte obriga a confrontar as afirmações do anterior com o código, e isso funciona como revisão de segunda passada.

### 6. Uma vez o material de apoio é que estava errado

No PRD, o revisor trocou o responsável do documento de Marcos para Larissa, citando como autoridade o destilado de formatos que eu mesmo tinha escrito como material de apoio. Estava certo sobre a regra e errado sobre o mérito: aquela regra justificava Larissa com uma fala em que ela diz que vai abrir o **doc de design**, o que sustenta a autoria dela no RFC e no FDD, não no PRD. O conteúdo do PRD veio do PM, que abriu o contexto, ditou os requisitos e negociou o prazo. Restaurei Marcos e corrigi o material de apoio, que era a fonte do defeito.

Registro porque mostra o limite do revisor automatizado: ele verifica consistência com a regra que você deu, inclusive quando a regra está errada.

## Como navegar a entrega

```
.
├── README.md                  este documento (processo)
├── TRANSCRICAO.md             a reunião, fonte primária (não alterada)
├── docs/
│   ├── DESAFIO.md             enunciado original do desafio
│   ├── PRD.md                 problema, escopo e métricas (produto)
│   ├── RFC.md                 proposta técnica e questões em aberto (arquitetura)
│   ├── FDD.md                 fluxos, contratos e integração (implementação)
│   ├── TRACKER.md             rastreabilidade de cada item à sua origem
│   └── adrs/
│       ├── README.md          chave de leitura das referências e índice
│       └── ADR-001 a ADR-008  as oito decisões, uma por arquivo
└── src/, prisma/, tests/      a aplicação existente (não alterada)
```

Ordem sugerida de leitura, para quem chega sem contexto:

1. **[`docs/PRD.md`](docs/PRD.md)** para entender o problema e por que a feature existe.
2. **[`docs/RFC.md`](docs/RFC.md)** para a proposta técnica em nível de arquitetura, as alternativas descartadas e o que ficou em aberto.
3. **[`docs/adrs/README.md`](docs/adrs/README.md)** e em seguida os ADRs que interessarem, para o porquê de cada decisão isolada.
4. **[`docs/FDD.md`](docs/FDD.md)** para implementar, incluindo a seção 11, que nomeia cada ponto do código existente que a feature toca.
5. **[`docs/TRACKER.md`](docs/TRACKER.md)** a qualquer momento, para conferir de onde veio uma afirmação específica.

Para conferir uma citação: `[hh:mm]` remete a [`TRANSCRICAO.md`](TRANSCRICAO.md), que está organizada como `[hh:mm] Nome: fala`, então basta buscar o horário. `caminho/arquivo.ext:linha` remete ao código deste repositório.

A entrega é puramente documental. Nenhum arquivo de `src/`, `prisma/`, `tests/` ou de configuração foi alterado, e a `TRANSCRICAO.md` está como veio.
