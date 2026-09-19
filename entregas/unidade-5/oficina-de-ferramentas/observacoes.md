# Observações — Oficina de ferramentas: RabbitMQ e consumidor idempotente

## Condição alterada

A oficina define a condição a tornar observável: **repetir a publicação do mesmo
`event_id`** (entrega pelo menos uma vez) e observar que o efeito de negócio não
se repete. Também foi alterado o **payload** para o caso de erro: republicar o
evento **sem `result_reference`**, para observar a rejeição de contrato e a
mensagem na DLQ. Não editei arquivo compartilhado: usei variáveis de porta do
terminal (`RABBITMQ_PORT=15672`, `RABBITMQ_MANAGEMENT_PORT=15673`) e UUIDs
sintéticos fixos dados pelo próprio enunciado.

## O que a saída revelou

### Idempotência em três camadas de observação

1. **Saída do consumidor** (`sequencia-essencial.txt`): a primeira entrega
   imprime `processed=True attempts=1`; a segunda, do mesmo `event_id`, imprime
   `processed=False attempts=2`. O contraste é imediato: **duas mensagens
   recebidas ≠ duas cobranças**. O broker entregou uma cópia de cada vez; o
   consumidor produziu efeito apenas na primeira.
2. **SQLite** (`consulta-sqlite.txt`): `select event_id, attempts` retorna
   `[('3fa85f64-…', 2)]` e `select count(*) from billing_effects` retorna `(1,)`.
   A tabela `processed_events` mede entregas **vistas**; `billing_effects` mede a
   consequência de negócio **idempotente**. A separação das duas tabelas deixou a
   semântica legível.
3. **Teste automatizado** (`testes-idempotencia.txt`): `2 passed, 1 skipped`. Os
   dois testes fora do Compose provam a regra em banco temporário: duplicar o
   evento produz uma linha de efeito e duas tentativas; uma mensagem inválida é
   rejeitada sem ack e sem derrubar o consumidor. O teste live fica `skip` porque
   é opt-in (`COMPOSE_LIVE=1`).

### A confirmação (ack) e o SQLite definem a janela de duplicação

A leitura do código confirmou por que confirmar antes do SQLite seria inseguro: a
confirmação ocorre **depois** de `processar_evento`, que grava
`processed_events` + `billing_effects` em uma transação `BEGIN IMMEDIATE`. Se o
processo caísse entre a escrita do efeito e o ack, a mensagem voltaria e o
`event_id` já registrado evitaria o segundo efeito. Confirmar antes deixaria a
mensagem "gasta" sem efeito garantido — perda silenciosa. O `event_id` (identifica
a ocorrência) é o que muda quando o fato se repete; o `exam_id` (identifica o
exame) permanece o mesmo nas duas tentativas — foi isso que a consulta expôs.

### Dead-letter queue como evidência de contrato violado

O payload sem `result_reference` falhou no `model_validate_json` do Pydantic
(`extra="forbid"` + campo obrigatório). O consumidor imprimiu
`Mensagem rejeitada para DLQ: schema inválido (1 erro)`, **não** imprimiu
`processed=…`, e o management API passou a mostrar `messages: 1` em
`billing.resultados.v1.dlq`. A infraestrutura (exchange topic, fila durável, DLX
`hospital.events.dlx` direcional, bindings) foi declarada pelo consumidor e
confirmada no endpoint antes de qualquer mensagem. Três evidências em arquivos
distintos fecharam o experimento: `dlq.txt` (manual), `estado-inicial.txt` (antes,
fila vazia e exchange inexistente) e `testes-idempotencia-live.txt`
(`COMPOSE_LIVE=1` → `3 passed`, o teste percorre publicação → validação →
rejeição → DLX → DLQ e ainda confere o corpo da mensagem na DLQ).

### Entrega pelo menos uma vez, não exactly-once

O experimento demonstra **entrega pelo menos uma vez com idempotência**, não
exactly-once. O broker não promete não repetir; a garantia é construída no
consumidor com banco. Exactly-once cruzaria broker, banco e efeito externo —
complexidade que a demonstração evita de propósito. O enunciado reforça que o
Compose não é produção (sem cluster, TLS, credenciais fortes, backup ou retenção
definida); a evidência serve para discutir semântica, não para validar um
datacenter.

## Questões exploratórias respondidas

- **Que dado seria excessivo no payload de um resultado?** Laudo inteiro,
  diagnóstico clínico ou qualquer dado sensível: o contrato carrega apenas a
  referência (`result_reference`), não o conteúdo. Excessivo também seria uma
  instrução de cobrança — isso é reação do consumidor, não parte do fato.
- **Qual é a diferença entre `exam_id` e `event_id` na repetição?** `exam_id`
  identifica o exame (mesmo nas duas tentativas); `event_id` identifica a
  ocorrência do fato. A repetição preserva `event_id` e reposiciona `attempts`.
- **Por que alterar variável de porta é mais seguro que editar arquivo
  compartilhado?** O override no terminal não altera `compose.eventos.yml`, que é
  compartilhado pelo repositório; editar o arquivo criaria diff indesejado e
  conflito entre alunos.
- **O que o healthcheck confirma e o que ele não confirma?** Confirma que o
  processo `rabbitmq` responde ao `ping` do diagnóstico — broker pronto para
  conexão. Não confirma routing, filas declaradas, contrato do payload ou
  idempotência: isso só aparece quando publicador e consumidor rodam.
- **Qual mudança exigiria uma nova versão do evento?** Alteração incompatível do
  contrato, como renomear campos obrigatórios ou mudar o significado persistido
  em `billing_effects` — exatamente o tipo de mudança que quebraria consumidores
  que ainda leem `ResultadoLaboratorialDisponibilizado.v1`.
- **Por que o store pertence ao consumidor, não à exchange?** O broker roteia
  cópias; ele não sabe se Faturamento já lançou a cobrança. Quem tem efeito de
  negócio decide idempotência; o store é privado do contexto que produz o efeito.
- **Em qual etapa uma queda poderia gerar redelivery?** Entre o ack implícito do
  broker e a confirmação efetiva do processamento — ou entre duas tentativas do
  consumidor. A janela é a do `message.process`: se o processo cair antes do ack
  concluído, o broker redelivers.
- **Por que confirmar antes do SQLite seria inseguro?** A mensagem seria
  considerada processada sem efeito garantido: perda silenciosa. A ordem correta é
  efeito idempotente no banco e só então ack.
- **Por que republicar o corpo inválido sem correção produz um ciclo?** Cada cópia
  falha a mesma validação e volta à DLQ; sem correção do contrato ou decisão de
  recuperação, a DLQ só cresce.
- **Qual sinal operacional avisaria que uma DLQ deixou de ser excepcional?**
  Contagem de mensagens, taxa de entrada, idade da mensagem mais antiga e alarme
  quando `messages > 0` persistir além de um SLA — DLQ que cresce sem ação vira
  depósito, não evidência.

## Divergências e notas

- **Bateria de teste:** sem `COMPOSE_LIVE=1` o resultado foi `2 passed, 1
  skipped`; com `COMPOSE_LIVE=1` e o Compose ativo, `3 passed`. Os arquivos de
  evidência deixam clara a diferença de escopo (unit/local × broker real).
- **Portas:** mantive os padrões do enunciado (15672 AMQP, 15673 management), que
  estavam livres; nenhum override foi necessário no `compose.eventos.yml`.
- **Estado antes de declarar:** capturei a ausência de filas e da exchange via
  management API antes do primeiro `consumidor --once`, para a evidência mostrar a
  declaração acontecendo e não "já existia".

## Responsabilidade arquitetural sustentada

A oficina transforma a tríade **evento / broker / consumidor idempotente** em
experimento observável:

1. **Evento como fato, não como comando**: `ResultadoLaboratorialDisponibilizado.v1`
   afirma um passado (resultado disponível); não carrega instrução de cobrança.
   Versionado pelo sufixo `.v1` e validado por Pydantic (`extra="forbid"`).
2. **Broker como canal, não como regra de negócio**: exchange topic roteia por
   chave; a DLX encaminha rejeições; o broker não decide cobrança nem contrato.
3. **Consumidor com store próprio**: `ProcessedEventStore` (SQLite) decide
   duplicação, e a falha de contrato vira evidência na DLQ (mensagem retida para
   diagnóstico, não silenciosamente perdida) — coerente com o "checklist de
   decisão" do módulo: quem tem efeito, qual chave preserva ordem, que duplicação
   é possível entre escrita e confirmação, onde aparece atraso e dead-letter.