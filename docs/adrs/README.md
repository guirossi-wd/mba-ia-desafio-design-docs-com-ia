# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) do projeto.
Cada decisão arquitetural relevante é registrada aqui em um arquivo individual,
nomeado no formato `ADR-NNN-titulo-em-kebab-case.md`.

## Como ler as referências

Os ADRs deste pacote citam duas fontes, e nenhuma afirmação neles existe sem uma delas.

- **`[hh:mm]` seguido de um nome** remete à reunião técnica registrada em [`TRANSCRICAO.md`](../../TRANSCRICAO.md), na raiz do repositório. O arquivo é organizado como `[hh:mm] Nome: fala`, então qualquer citação pode ser localizada buscando o horário. A reunião não tem data de calendário registrada, apenas o horário.
- **`caminho/arquivo.ext:linha`** remete ao código da aplicação neste mesmo repositório.

No corpo dos documentos, o horário aparece somente quando a frase trata de alguém **decidir, propor ou objetar**, porque aí a autoria é parte da decisão. Afirmações técnicas são apresentadas como fato. A procedência completa de cada ADR, incluindo o que não é citado no corpo, está na seção **Referências** ao final de cada arquivo.

O mapeamento item a item entre os documentos e suas origens está em [`docs/TRACKER.md`](../TRACKER.md).

## Índice

| ADR | Decisão |
|---|---|
| [ADR-001](ADR-001-outbox-no-mysql.md) | Emissão de eventos via padrão Outbox no MySQL |
| [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) | Worker em processo separado com polling de 2 segundos |
| [ADR-003](ADR-003-retry-com-backoff-e-dlq.md) | Backoff exponencial de 5 tentativas e DLQ persistida |
| [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint |
| [ADR-005](ADR-005-at-least-once-com-x-event-id.md) | Entrega at-least-once com deduplicação pelo cliente |
| [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso dos padrões já existentes na codebase |
| [ADR-007](ADR-007-snapshot-do-payload-na-outbox.md) | Snapshot do conteúdo do evento na inserção |
| [ADR-008](ADR-008-filtro-de-eventos-na-insercao.md) | Filtro de eventos aplicado na inserção |
