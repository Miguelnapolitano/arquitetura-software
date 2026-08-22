# Observações — Experimento 1: Camadas (agenda clínica)

## Condição alterada

Em `main.py`, o cenário 2 tentava agendar para a Dra. Ana das **09:15 às 09:45**, que se sobrepõe à consulta #1 (09:00–09:30) e por isso retornava HTTP 409. Alterei a entrada para **09:30 às 10:00** — um intervalo livre, encaixado exatamente entre as consultas já existentes (09:00–09:30 e 10:00–10:30). Nenhuma regra de negócio foi alterada; mudei apenas o dado de entrada do cenário.

## O que a saída revelou

- `saida-antes.txt`: `HTTP 409 CONFLICT → {'erro': 'Dr(a). Dra. Ana Silva já tem consulta das 09:00 às 09:30.'}`
- `saida-depois.txt`: o mesmo endpoint retorna `HTTP 201 CREATED` com a nova consulta (id=4, 09:30–10:00), e a agenda do dia passa a listar três consultas ordenadas, incluindo a nova.

A comparação mostra que o conflito não é uma propriedade fixa do código, mas o resultado da **avaliação da regra sobre os dados**: como os intervalos 09:30–10:00 e 09:00–09:30 apenas se tocam na fronteira (sem sobreposição), a invariante não é violada e o agendamento é aceito.

## Responsabilidade arquitetural relacionada

A evidência sustenta a separação em camadas:

1. A **regra de conflito** vive na camada de domínio (`Horario.conflita_com`, em `dominio.py`) e é aplicada pela camada de serviço (`AgendamentoServico.agendar`, em `servicos.py`) — foi ela que produziu resultados diferentes diante de entradas diferentes.
2. A **camada de apresentação** (`apresentacao.py`) não mudou nada: continuou traduzindo a exceção de negócio em HTTP 409 e o sucesso em HTTP 201. Ou seja, mudar o comportamento observável exigiu mexer só nos dados que atravessam as fronteiras de camada, não nas responsabilidades.
3. O armazenamento continua em memória, mas o serviço só depende do contrato (`RepositorioConsulta`), reforçando que cada camada colabora sem conhecer os detalhes da vizinha de baixo.
