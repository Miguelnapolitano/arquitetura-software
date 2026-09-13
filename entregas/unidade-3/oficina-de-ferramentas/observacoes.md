# Observações — Oficina de ferramentas: dois serviços, dois bancos e uma falha parcial

## Condição alterada

Diferente das oficinas anteriores, o próprio roteiro define a condição a tornar
observável: **parar um dos dois serviços e observar a falha parcial**. A
alteração executada foi `docker compose stop elegibilidade`; o estado nominal
serviu de controle (mesmos comandos, respostas diferentes). Também diferi das
portas-padrão 8001/8002 para `18001`/`18002`, conforme sugerido para não
disputar portas.

## O que a saída revelou

### Falha parcial em três camadas de observação

1. **`docker compose ps`** revelou a topologia esvaziada: com `elegibilidade`
   parado, os contêineres restantes são `db_elegibilidade`, `db_exames` e
   `exames`, os três `healthy`. O serviço de Exames não parou nem ficou
   `unhealthy` — o health check dele não mira dependências remotas.
2. **`POST /exames`** expôs a capacidade interrompida com semântica correta:
   `503 Service Unavailable` e `detail.codigo = dependencia_indisponivel`.
   O código em `src/hospital/servicos/exames.py` mapeia a falha de
   `httpx.Client.get` (neste caso `ConnectError` por serviço parado) para esse
   código, e não para um `500`. Isso é visível no corpo da resposta
   (`evidencias/post-exame-503.txt`) — deu para distinguir a causa na própria
   fronteira.
3. **`GET /health` de Exames** continuou `200`, confirmando que o processo e a
   base própria seguiram vivos. A frase do enunciado ficou demonstrada: o health
   check "confirma processo e banco próprio, não todas as dependências remotas".

A classificação de acoplamento do material da unidade (conceitos.md) ficou
observável aqui: Exames mantém **acoplamento temporal** com Elegibilidade
(espera resposta síncrona, com timeout de 2 s) e abre mão do **acoplamento de
dados** (não acessa a tabela do outro serviço). Quando o provedor para, o
consumidor falha de forma explícita em vez de adivinhar.

### Fronteira de dados reforçada por topologia

A propriedade dos dados tem dois mecanismos na demonstração, e ambos apareceram
na verificação:

- **Código**: `test_exames_source_cannot_access_eligibility_table_directly`
  varre `src/hospital/servicos/exames.py` e não encontra
  `elegibilidade.beneficiarios`, `from elegibilidade`, `join elegibilidade` ou
  `db_elegibilidade`. O acesso de Exames ao estado do outro contexto é 100%
  HTTP.
- **Rede**: cada banco está em uma rede `internal` própria
  (`elegibilidade-db-net` e `exames-db-net`); os serviços compartilham apenas a
  `application-net`. Exames resolveria `db_elegibilidade` apenas se houvesse uma
  rota — não há. Mesmo um bug de SQL não encontraria a base vizinha.

### Recuperação com dado persistido

O caminho de recuperação `up -d --wait` religou Elegibilidade e a segunda
solicitação retornou `201` com `solicitacao_id: 2` — e não `1`. Isso aconteceu
porque `stop` não remove recursos: o contêiner e o volume `exames_data`
sobreviveram, então o contador de identidade continuou de onde parou. É um
contraste didático com o `down -v` final, que remove contêineres, redes e
volumes; depois dele, uma nova subida recomeçaria do `solicitacao_id: 1`.

## Questões exploratórias respondidas

- **Qual dependência permanece saudável?** Os dois bancos e o processo de
  Exames (inclusive `db_elegibilidade`, que não depende de nenhum outro
  serviço). O health check de Exames reflete apenas Exames.
- **Qual capacidade deixa de ser concluída?** A criação de solicitação de
  exame: `POST /exames` não conclui enquanto Elegibilidade não responder. A
  capacidade "verificar elegibilidade" obviamente também não está disponível,
  mas o ponto arquitetural é o consumidor: ele observa a indisponibilidade da
  dependência no momento em que precisa dela.

## Divergências e notas

- **Testes**: o enunciado prevê `4 passed`, e o resultado real foi exatamente
  `4 passed` (com um aviso de depreciação do `TestClient` do Starlette sobre
  `httpx`/`httpx2`, sem impacto). Differente do Módulo 2, aqui a contagem do
  enunciado bateu com a do repositório.
- **Portas**: defini as variáveis de ambiente antes de qualquer comando do
  Compose; `config --services` não dependeu delas (names são estáticos), mas os
  mapeamentos de porta sim (`0.0.0.0:18001->8000` e `0.0.0.0:18002->8000` no
  `ps`).
- **Health check posterior**: confirmei `/health` das duas aplicações já no
  estado nominal e novamente após a falha; em nenhum momento o health check de
  Exames acusou problema, mesmo com a dependência parada — este é exatamente o
  comportamento que o enunciado pede para observar.

## Responsabilidade arquitetural sustentada

A oficina transforma o conceito de **banco por serviço / fronteira lógica e
física** em um experimento observável:

1. **Contrato HTML sobre propriedade de dados**: Exames lê o estado do outro
   contexto mediante `GET /elegibilidades/{id}` e grava somente na própria base;
2. **Falha parcial como semântica de erro**: `503 dependencia_indisponivel`
   comunica "a capacidade depende de algo que não está disponível" sem derrubar
   o serviço consumidor;
3. **Testes de fronteira sem rede**: os quatro testes usam `TestClient` +
   `MockTransport`/`monkeypatch`, provando que o contrato de fronteira é
   verificável independentemente do Compose — o mesmo padrão de "três
   perspectivas" (implementação, contrato e operação) do Módulo 2, agora com
   topologia real de contêineres.