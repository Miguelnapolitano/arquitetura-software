# 5.5 — Oficina de ferramentas: RabbitMQ e consumidor idempotente

Oficina do Módulo 5 (Arquiteturas orientadas por eventos) executada sobre a
demonstração descartável `laboratorios/plataforma-hospitalar/infra/compose.eventos.yml`.
Ela reúne um broker RabbitMQ 4 com management plugin, um publicador
(`src/hospital/eventos/publicador.py`) e um consumidor de Faturamento
(`src/hospital/eventos/consumidor.py`) que só produz efeito quando a entrega é
nova. Duas propriedades arquiteturais são tornadas observáveis: **entrega pelo
menos uma vez** (o mesmo `event_id` pode chegar mais de uma vez, e o broker não
sabe disso) e **idempotência do efeito** (guardada em SQLite pelo consumidor,
antes do ack). Uma **mensagem inválida** demonstra a dead-letter queue.

## Arquitetura da demonstração

```mermaid
graph LR
    subgraph Broker["RabbitMQ 4 :15672 (AMQP) / :15673 (management)"]
        X["hospital.events\nexchange topic\ndurable"]
        DLX["hospital.events.dlx\nexchange direct"]
        Q["billing.resultados.v1\nfila durável\n(dead-letter → DLX)"]
        DLQ["billing.resultados.v1.dlq"]
    end

    P["publicador.py\nResultadoLaboratorialDisponibilizado.v1"]
    C["consumidor.py\nConsumidorFaturamento"]
    S[("processed-events.sqlite3\nevidências/modulo-5\nprocessed_events · billing_effects")]

    P -->|ROUTING_KEY\nlaboratory.result.available.v1| X
    X --> Q
    C -->|declara exchange, fila, DLX, DLQ| X
    C -->|consome, valida Pydantic,\nconfirma| Q
    C -->|executa ProcessedEventStore\n(idempotência) e efeito| S
    Q -->|rejeição sem ack| DLX
    DLX --> DLQ

    classDef broker fill:#1a3a5c,stroke:#4d9ef7,color:#cce4ff
    classDef app fill:#1a3d1a,stroke:#4caf50,color:#d4f0d4
    classDef db fill:#2d1a40,stroke:#9c6ab3,color:#e8d4f0
    class P,C app
    class X,DLX,Q,DLQ broker
    class S db
```

O broker não decide regra de negócio: a exchange roteia por chave e a fila de
trabalho segura as mensagens. Quem decide se um `event_id` já produziu efeito é o
`ProcessedEventStore`, que pertence ao consumidor. Quem decide se uma mensagem
inválida deve parecer sucesso é o próprio consumidor — ele rejeita sem ack e a
mensagem segue à DLQ.

## Ferramentas

| Ferramenta | Versão | Papel nesta oficina |
| --- | --- | --- |
| Docker Engine | 29.8.1 | contêiner do RabbitMQ isolado |
| Docker Compose | v5.5.1 | orquestração + healthcheck do broker |
| RabbitMQ 4 (management) | imagem `4-management` do Compose | exchange, fila, confirmações e DLQ |
| Python | 3.12.12 (`.venv`) | publicador, consumidor e testes |
| aio-pika + Pydantic | 10.0.1 / 2.13.5 | AMQP assíncrono e contrato do evento |
| SQLite | embutido no Python | store durável idempotente |

Ver versões completas em `evidencias/versoes.txt`.

## Roteiro executado

Portas padrão mantidas: `RABBITMQ_PORT=15672` (AMQP, para aio-pika) e
`RABBITMQ_MANAGEMENT_PORT=15673` (web plugin, só para inspeção).
`RABBITMQ_URL="amqp://guest:guest@localhost:15672/"`. A conta e o volume
pertencem apenas ao ambiente descartável do Compose.

### 1. Validar configuração e iniciar o broker

```bash
docker compose -f infra/compose.eventos.yml config --quiet   # exit 0, sem texto
docker compose -f infra/compose.eventos.yml up -d --build --wait
docker compose -f infra/compose.eventos.yml ps
```

`config --quiet` terminou sem texto de erro (`evidencias/config-quiet.txt` expõe o
`exit=0` e o YAML resolvido). `up --wait` só retornou quando o healthcheck
`rabbitmq-diagnostics -q ping` ficou saudável; `ps`
(`evidencias/up-inicial.txt` e `evidencias/ps-inicial.txt`) lista o serviço
`rabbitmq` `Up … (healthy)` com os mapeamentos `0.0.0.0:15672->5672` e
`0.0.0.0:15673->15672`.

Antes de qualquer declaração, o management API confirmava o estado inicial vazio:
nenhuma fila e exchange `hospital.events` ainda não criada
(`evidencias/estado-inicial.txt`).

### 2. Sequência essencial — publicação repetida e um único efeito

Primeiro `consumidor --once` com fila vazia: ele **declara** a exchange, a fila de
trabalho, a DLX e a DLQ e termina sem efeito. O management API passa a listar
`billing.resultados.v1` e `billing.resultados.v1.dlq`, e a exchange
`hospital.events` como `topic durable=True`. Depois: publicar, consumir, republicar
o **mesmo** `event_id`, consumir de novo. UUID sintético fixo.

```bash
python -m hospital.eventos.consumidor --once --store evidencias/modulo-5/processed-events.sqlite3
python -m hospital.eventos.publicador --event-id 3fa85f64-5717-4562-b3fc-2c963f66afa6
python -m hospital.eventos.consumidor --once --store evidencias/modulo-5/processed-events.sqlite3
python -m hospital.eventos.publicador --event-id 3fa85f64-5717-4562-b3fc-2c963f66afa6
python -m hospital.eventos.consumidor --once --store evidencias/modulo-5/processed-events.sqlite3
python -m pytest tests/test_event_idempotency.py -q
```

Saída real (`evidencias/sequencia-essencial.txt`):

```
=== passo 2: consumir (1a entrega) ===
ResultadoLaboratorialDisponibilizado.v1 event_id=3fa85f64-5717-4562-b3fc-2c963f66afa6 processed=True attempts=1

=== passo 4: consumir (2a entrega) ===
ResultadoLaboratorialDisponibilizado.v1 event_id=3fa85f64-5717-4562-b3fc-2c963f66afa6 processed=False attempts=2
```

A primeira entrega valida Pydantic, grava em `processed_events` **e** em
`billing_effects` e ack. A segunda entrega do mesmo `event_id` apenas incrementa
`attempts` e devolve `processed=False` — **nenhum efeito duplicado**. O teste com
broker desligado confirma a mesma regra em banco temporário
(`evidencias/testes-idempotencia.txt`):

```
..s                                                                      [100%]
2 passed, 1 skipped in 0.33s
```

O teste `test_live_invalid_event_reaches_dead_letter_queue` fica `skip` — ele é
opt-in e só roda com `COMPOSE_LIVE=1` (seção 4).

### 3. Explorar o estado persistido

```bash
python -c "import sqlite3; c=sqlite3.connect('evidencias/modulo-5/processed-events.sqlite3'); print(c.execute('select event_id, attempts from processed_events').fetchall()); print(c.execute('select count(*) from billing_effects').fetchone())"
```

Saída real (`evidencias/consulta-sqlite.txt`):

```
[('3fa85f64-5717-4562-b3fc-2c963f66afa6', 2)]
(1,)
```

Uma identidade com **duas tentativas** e apenas **um efeito** persistido. A tabela
`processed_events` mede entregas vistas; `billing_effects` representa a
consequência de negócio idempotente.

### 4. Extensão — mensagem inválida na DLQ

Payload deliberadamente **sem `result_reference`** (`--invalid`), mesmo ambiente,
UUID sintético próprio:

```bash
python -m hospital.eventos.publicador --event-id 65e95d82-4f8c-4e93-9bb3-3e0e92deaf1d --invalid
python -m hospital.eventos.consumidor --once --store evidencias/modulo-5/processed-events.sqlite3
curl -u guest:guest "http://localhost:${RABBITMQ_MANAGEMENT_PORT}/api/queues/%2F/billing.resultados.v1.dlq"
```

O consumidor não faz ack de sucesso: a validação Pydantic falha e ele rejeita com
`requeue=False`, encaminhando a mensagem pela DLX `hospital.events.dlx` à DLQ
`billing.resultados.v1.dlq`. Saída real (`evidencias/dlq.txt`):

```
Mensagem rejeitada para DLQ: schema inválido (1 erro)
...
messages: 1
messages_ready: 1
```

A evidência automatizada opt-in percorre publicação → validação → rejeição → DLX →
DLQ (`COMPOSE_LIVE=1 python -m pytest tests/test_event_idempotency.py -q`):

```
...                                                                      [100%]
3 passed in 0.32s
```

(`evidencias/testes-idempotencia-live.txt`.)

## Limpeza

```bash
docker compose -f infra/compose.eventos.yml down -v
rm -rf evidencias/modulo-5
```

`down -v` remove apenas o serviço e o volume nomeado por este arquivo Compose
(`rabbitmq_eventos_data`), sem tocar recursos de outros estudos. `ps -a` não lista
nenhum recurso ativo (`evidencias/down-v.txt` e `evidencias/ps-pos-limpeza.txt`).
Uma nova subida recomeça sem filas e sem o SQLite anterior. Antes da limpeza, a
evidência é copiada para `entregas/unidade-5/oficina-de-ferramentas/evidencias/`.

## Interpretação e respostas às questões

**Por que há entrega pelo menos uma vez com idempotência, e não exactly-once?**

A sequência demonstra que o broker entrega `processed=True attempts=1` e depois
`processed=False attempts=2`: a **mesma mensagem lógica chegou duas vezes** e o
efeito de negócio foi criado uma única vez. Nenhuma fila promete não repetir —
entre a escrita do efeito e a confirmação (ack) pode cair o processo, e então a
mensagem volta. Exactly-once exigiria uma transação distribuída entre broker,
banco e efeito externo; aqui o desenho é: **entregar pelo menos uma vez** e fazer
do efeito uma função idempotente do `event_id`. Confirmar antes do SQLite seria
inseguro porque "recebi" deixaria de significar "processei com efeito".

**Por que o store pertence ao consumidor, não à exchange?** O broker roteia
cópias; ele não sabe (nem deveria saber) se Faturamento já lançou a cobrança. A
propriedade do `event_id` processado é responsabilidade de quem produz efeito.

**Em qual cenário Kafka valeria como extensão do desenho?**

Kafka entraria como log distribuído quando houvesse requisito mensurável de
**reprocessamento**: vários grupos de consumidores independentes lendo as mesmas
rejeições/resultados em ritmos próprios, ou a necessidade de voltar a uma posição
(retenção + offsets) por um período definido, por exemplo reprocessar resultados
dos últimos 30 dias para novos consumidores de auditoria. Para isso usaríamos um
tópico com **chave de partição por `exam_id`** (ordem por exame), retenção
definida e grupos de consumidores de Faturamento e Notificação. Kafka é extensão
— não substituição automática: **mesmo com replay, ele não elimina a necessidade
de `event_id`, idempotência nem versão de contrato**; uma fila distribui trabalho
pendente, um log permite posições independentes de leitura, e a proteção de
referências (nunca transportar laudo inteiro, só `result_reference`) permanece.

## Arquivos de evidência

```
entregas/unidade-5/oficina-de-ferramentas/
├── README.md
├── observacoes.md
└── evidencias/
    ├── versoes.txt                  # docker 29.8.1 · compose v5.5.1 · python 3.12.12
    ├── config-quiet.txt             # config --quiet exit 0 + YAML resolvido
    ├── up-inicial.txt               # up -d --build --wait (healthy) + ps
    ├── ps-inicial.txt               # rabbitmq Up (healthy)
    ├── estado-inicial.txt           # filas [] e hospital.events não declarada
    ├── sequencia-essencial.txt      # processed=True attempts=1 → processed=False attempts=2
    ├── testes-idempotencia.txt      # 2 passed, 1 skipped
    ├── consulta-sqlite.txt          # 1 event_id, attempts=2, count(billing_effects)=1
    ├── dlq.txt                      # rejeição por schema + messages=1 na DLQ
    ├── testes-idempotencia-live.txt # COMPOSE_LIVE=1 → 3 passed
    ├── ps-final.txt                 # rabbitmq segue Up (healthy) antes da limpeza
    ├── down-v.txt                   # remoção do contêiner, volume e rede do Compose
    └── ps-pos-limpeza.txt           # nenhum recurso ativo
```

As mesmas evidências também estão em
`laboratorios/plataforma-hospitalar/evidencias/modulo-5/`, conforme a preparação
do laboratório.