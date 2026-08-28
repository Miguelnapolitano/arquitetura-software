# Observações — Oficina de ferramentas: contrato, execução e comparação

## Condição alterada

Cada oficina do Módulo 1 propunha alterar uma condição do experimento e observar o
efeito na saída. Nesta oficina do Módulo 2, o experimento comparável é a **falha
deliberada do contrato**. Mantive `contratos/openapi.yaml` intocado e trabalhei numa
cópia, `evidencias/openapi-experimento.yaml`. Nela alterei **somente** o exemplo de
mídia da requisição:

- caminho: `paths./elegibilidades.post.requestBody.content.application/json.examples.pedidoValido.value.cpf`
- antes: `12345678901` (onze dígitos, atende ao padrão `^\d{11}$`)
- depois: `123` (três dígitos, viola o padrão)

Não toquei no schema de `components.schemas.PedidoElegibilidade` nem no exemplo de
`components`, que o enunciado pede explicitamente para preservar.

## O que a saída revelou

- **Contrato original** (`spectral-valido.txt`): `No results with a severity of 'error' found!`
- **Cópia alterada** (`spectral-experimento-falha.txt`):

  ```
   34:24  error  oas3-valid-media-example  "cpf" property must match pattern "^\d{11}$"
                  paths./elegibilidades.post.requestBody.content.application/json.examples.pedidoValido.value.cpf
  ✖ 1 problem (1 error, 0 warnings, 0 infos, 0 hints)
  ```

Dois achados importantes:

1. **O linter pega o erro antes de qualquer chamada**: o exemplo de mídia é
   validado contra o schema declarado, e um exemplo que não casa com o padrão
   falha na hora do lint. Ou seja, exemplos podem ser executáveis e verificáveis
   por máquina — não são apenas documentação ilustrativa.
2. **Contrato e código concordam**: a mesma regra `^\d{11}$` está na aplicação
   (`src/hospital/api/models.py`, `Field(pattern=r"^\d{11}$")`). O valor `123`
   seria rejeitado com `422` pelo servidor assim como é reprovado pelo linter.
   Isso demonstra que a regra foi implementada em dois planos — documento e código
   — e ambos a aplicam da mesma forma.

### Comparação contrato explícito × contrato gerado

Uma divergência real apareceu ao comparar `contratos/openapi.yaml` (explícito) com
`app.openapi()` (gerado pela implementação) — ver `comparacao-contratos.txt`:

- `POST /elegibilidades`: documentados `202`/`422`, gerados `202`/`422` → alinhados.
- `GET /elegibilidades/{protocolo}`: documentados `200`/`404`, **gerados `200`/`404`/`422`**.

O `422` gerado a mais vem da validação automática de parâmetro de path que o
FastAPI aplica por padrão e o documento explícito não promete. Não é uma
incompatibilidade que quebra consumidores (todo status prometido existe na
implementação); é uma diferença de **cobertura de documentação** — a implementação
faz mais do que o contrato descreve. `PedidoElegibilidade.required` é idêntico nos
dois documentos, reforçando que o núcleo do contrato está alinhado.

## Responsabilidade arquitetural relacionada

A oficina sustenta a separação **interface / contrato / implementação**:

1. **Spectral examina o documento.** Ele prova que o arquivo `openapi.yaml` é
   estruturalmente válido e que os exemplos obedecem aos schemas — mas não prova
   que o servidor se comporta assim. A falha deliberada só foi percebida porque o
   lint verifica o exemplo de mídia.
2. **`TestClient` examina a implementação.** Os sete testes de
   `test_api_contract.py` bateram na aplicação real e provaram comportamento
   (`202`+`Location`, recuperação por `GET`, `422` estruturado, `404`, exemplos do
   contrato aceitos). O `TestClient` não abre porta nem depende de rede — uma
   execução manual no Bruno nunca substitui essa regressão.
3. **Bruno/curl examinam a experiência do consumidor.** As respostas reais
   (`post-ok`, `get-ok`, `post-422`) mostram o que um cliente vê na fronteira.

Há uma lacuna que a ferramenta não decide sozinha: o `422` a mais no `GET` é um
exemplo de **regra semântica** que o linter não poderia julgar — saber se esse
status deveria estar documentado é decisão de arquiteto, não de regra sintática.
Isso reforça o mantra da oficina: use as três perspectivas (documento,
implementação e consumidor) porque nenhuma cobre as outras duas.

## Nota sobre o número de testes

O enunciado da oficina diz procurar **seis** testes em `test_api_contract.py`.
Neste repositório o arquivo contém **sete** funções de teste e o resultado real foi
`7 passed` (a contagem do documento está defasada em relação ao estado do clone).
Registrei esse descompasso em vez de omitir, para a evidência corresponder à
execução real.
