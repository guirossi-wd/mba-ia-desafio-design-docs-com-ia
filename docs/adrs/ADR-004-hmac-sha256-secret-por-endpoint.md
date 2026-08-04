# ADR-004: Autenticidade e integridade do evento via HMAC-SHA256 com secret por endpoint

**Status:** Aceito
**Data:** 2026-07-29
**ADRs Relacionados:** ADR-005, ADR-007

## Contexto e Declaração do Problema

Sofia abriu o bloco de segurança em `[09:19]` colocando o risco em uma frase: a plataforma passa a expor dados de pedidos para um endpoint fora da sua infraestrutura. Isso cria duas necessidades no lado do cliente que antes não existiam.

A primeira é **autenticidade**. O endpoint do cliente é público por definição, já que precisa aceitar requisição vinda da internet. Sem prova de origem, qualquer um que descubra a URL pode enviar conteúdos falsos, e o cliente vai processá-los como se fossem eventos legítimos de mudança de status de pedido.

A segunda é **integridade**. Mesmo com a origem correta, o cliente precisa saber que o conteúdo não foi alterado no caminho.

O ponto de partida é greenfield. O sistema hoje protege o próprio tráfego com token de portador e senha com hash, mas não assina nada, não verifica integridade de mensagem e não guarda credencial de terceiro. Qualquer mecanismo escolhido aqui é o primeiro do projeto nesse papel, e vai definir também como esse tipo de credencial passa a ser tratado em log e em resposta de API.

Há também um problema de escopo de credencial. Se uma única credencial protege todos os endpoints de todos os clientes, o vazamento dessa credencial em um único cliente compromete a comunicação com todos. Sofia formulou isso em `[09:21]` de forma econômica: se vaza uma, vaza tudo. Diego reforçou no minuto seguinte com caso concreto, relatando cliente que já expôs uma credencial em log de aplicação.

E há o problema de troca de credencial. Uma credencial que não pode ser trocada sem quebrar a integração significa que, na prática, ela nunca é trocada, mesmo depois de um vazamento conhecido.

## Fatores de Decisão

- O endpoint de destino é público e não pode distinguir requisição legítima de forjada sem prova criptográfica.
- O cliente precisa conseguir verificar origem e integridade com bibliotecas que já tenha, sem projeto de integração longo.
- Vazamento de credencial precisa ter raio de alcance limitado a um único endpoint.
- Troca de credencial precisa ser possível sem janela de indisponibilidade da integração.
- A plataforma não controla o ambiente do cliente, e o histórico mostra credencial exposta em log de cliente.
- A solução precisa caber na plataforma de execução atual sem dependência nova.

## Alternativas Consideradas

1. Assinatura HMAC-SHA256 do corpo da requisição, com secret exclusiva por endpoint e rotação com período de sobreposição.
2. Assinatura HMAC com uma única secret global da plataforma, compartilhada por todos os clientes.

## Resultado da Decisão

Opção escolhida: **HMAC-SHA256 sobre o corpo da requisição, com secret exclusiva por endpoint e suporte a rotação com período de sobreposição de 24 horas**. Sofia foi a decisora deste ADR e registrou a decisão fechada em `[09:22]`, depois de estabelecer o padrão e o algoritmo em `[09:20]` e o escopo da credencial em `[09:21]`.

A assinatura viaja em cabeçalho próprio da requisição e o cliente a recalcula do lado dele com a secret que possui. A escolha do SHA-256 foi justificada como padrão de mercado, com disponibilidade de biblioteca em qualquer stack séria.

A secret é gerada pela plataforma e devolvida ao cliente no momento do cadastro do endpoint. O cliente pode pedir uma nova secret pela API e, durante 24 horas, a secret antiga continua válida em paralelo, dando tempo de ele atualizar os sistemas dele antes de a antiga ser invalidada.

Duas medidas de segurança complementares foram decididas no mesmo bloco e ficam registradas aqui como contexto, sem constituir decisão arquitetural separada. A URL do endpoint precisa usar TLS, e cadastro em canal não cifrado é recusado na validação de entrada. E o envio passa a carregar um cabeçalho com o horário do disparo, com o objetivo explícito de permitir ao cliente detectar tentativa de reenvio de requisição capturada, mais um cabeçalho identificando qual cadastro de webhook originou o envio, para clientes que mantêm vários endpoints.

## Prós e Contras das Alternativas

### HMAC-SHA256 com secret por endpoint e rotação

- Boa, porque resolve autenticidade e integridade com um único mecanismo, sem infraestrutura de chaves.
- Boa, porque é o padrão de mercado para assinatura de webhook, e qualquer cliente sério já tem biblioteca disponível para verificá-la.
- Boa, porque o isolamento por endpoint limita o dano de um vazamento a um único cliente.
- Boa, porque a sobreposição de 24 horas permite trocar credencial sem coordenar janela de manutenção com o cliente.
- Ruim, porque quem obtiver a secret consegue forjar assinatura válida para qualquer conteúdo, o que faz do armazenamento e do transporte dessa secret o ponto crítico do desenho.
- Ruim, porque a plataforma passa a guardar uma credencial por endpoint, com obrigação de nunca expô-la em log, resposta de API ou mensagem de erro.

### HMAC com secret global da plataforma

- Boa, porque simplifica ao extremo a gestão: uma credencial para configurar e distribuir.
- Boa, porque dispensa modelar guarda e ciclo de vida de credencial por endpoint.
- Ruim, porque um vazamento em qualquer cliente compromete a comunicação com todos, consequência rejeitada explicitamente na reunião.
- Ruim, porque rotação exige coordenação simultânea com todos os clientes, o que na prática torna a rotação inviável.

## Consequências

- Boa, porque o cliente ganha capacidade de rejeitar requisição forjada e detectar alteração de conteúdo sem depender de nada além de uma função de hash.
- Boa, porque a superfície de comprometimento fica contida: cada endpoint tem sua própria credencial, e o incidente de um cliente não contamina os demais.
- Boa, porque a rotação com sobreposição de 24 horas transforma resposta a vazamento em operação de rotina, e não em incidente com indisponibilidade.
- Ruim, porque a plataforma assume a responsabilidade de custodiar uma credencial por endpoint, o que obriga a estender a política de omissão de dados sensíveis em log e a garantir que nenhuma resposta de API devolva a secret depois da criação.
- Ruim, porque durante as 24 horas de sobreposição existem duas credenciais válidas ao mesmo tempo, o que amplia temporariamente a janela de risco e cria estado adicional para modelar e expirar.
- Ruim, porque a verificação depende de o cliente calcular a assinatura sobre exatamente os mesmos bytes enviados. Qualquer reserialização do conteúdo do lado dele invalida a assinatura, e essa é uma falha de integração que a plataforma não consegue diagnosticar de fora, o que gera demanda de suporte previsível.
- Ruim, porque a entrega passa a ter dependência de revisão especializada: a engenharia de segurança reservou pelo menos dois dias úteis para revisar assinatura e geração de secret antes do deploy, o que coloca um portão obrigatório no fim do cronograma.
- Neutra, porque a responsabilidade de verificar a assinatura fica inteiramente com o cliente. A plataforma assina e entrega, mas não tem como saber se o cliente valida, e por isso a instrução precisa aparecer na documentação de integração publicada no portal do desenvolvedor.

## Referências

- `src/shared/logger/index.ts:4` lista de campos omitidos em log, que precisa passar a cobrir a secret do webhook
- `src/middlewares/validate.middleware.ts:11` mecanismo declarativo de validação de entrada onde a exigência de canal cifrado será aplicada
- `src/modules/customers/customer.schemas.ts:17` exemplo de validação de formato de string na entrada, padrão a seguir para a URL do endpoint
- `TRANSCRICAO.md` `[09:19]` Sofia: risco de expor dados de pedido para fora da infraestrutura
- `TRANSCRICAO.md` `[09:20]` Sofia e Bruno: assinatura HMAC e escolha do algoritmo
- `TRANSCRICAO.md` `[09:21]` Sofia e Bruno: secret por endpoint, rotação com sobreposição de 24 horas
- `TRANSCRICAO.md` `[09:22]` Diego e Sofia: caso de credencial vazada em log de cliente e fechamento da decisão
- `TRANSCRICAO.md` `[09:23]` Sofia: exigência de canal cifrado na URL cadastrada
- `TRANSCRICAO.md` `[09:29]` Bruno: nenhuma dependência nova no projeto
- `TRANSCRICAO.md` `[09:31]` Marcos: secret gerada pela plataforma e devolvida no cadastro
- `TRANSCRICAO.md` `[09:40]` Marcos: documentação de integração no portal do desenvolvedor
- `TRANSCRICAO.md` `[09:44]` Diego e Sofia: cabeçalhos de horário do disparo e de identificação do cadastro
- `TRANSCRICAO.md` `[09:46]` Sofia: dois dias úteis de revisão de segurança antes do deploy
- [Verificação de assinatura e prevenção de reenvio em webhooks](https://webhooks.fyi/security/replay-prevention)
- [Rotação de secret com sobreposição no modelo do Stripe](https://www.hooklistener.com/learn/stripe-webhook-security-guide)
