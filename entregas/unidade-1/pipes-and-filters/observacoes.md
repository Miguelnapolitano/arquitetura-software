# Observações — Experimento 2: Pipes and Filters (triagem de currículos)

## Condição alterada

Em `main.py`, a vaga era criada com `experiencia_minima=3`. Alterei para `experiencia_minima=4`. Nenhum filtro foi modificado e nenhum currículo foi adicionado ou removido — mudei apenas um parâmetro do critério usado pelo tester `FiltroPorExperienciaMinima`.

## O que a saída revelou

- `saida-antes.txt`: 3 aprovados (Ana Lima, Elena Souza, Diego Faria). Elena (3 anos) passava porque atendia ao mínimo de 3.
- `saida-depois.txt`: surge uma nova linha `[REPROVADO] Elena Souza: 3 ano(s) < mínimo 4`, o relatório final passa de "3 candidato(s) aprovado(s)" para **2**, e Elena desaparece do ranking encaminhado à entrevista.

A comparação mostra três efeitos encadeados no fluxo: (1) o **tester** descartou mais um item logo após a normalização; (2) os filtros seguintes (`CalculadorDeScore`, `RelatorioDeTriagem`) nunca viram esse item; (3) o **consumer** refletiu automaticamente o novo conjunto na contagem e no ranking. Um único critério, alterado no meio do pipe, propagou-se até o fim sem que nada downstream fosse editado.

## Responsabilidade arquitetural relacionada

No estilo Pipes and Filters, cada filtro tem uma única responsabilidade e se comunica apenas pela mensagem que produz. A evidência mostra isso em ação:

1. O descarte é responsabilidade exclusiva do **tester** — foi nele que o comportamento mudou.
2. O **transformer** (`CalculadorDeScore`) e o **consumer** (`RelatorioDeTriagem`) permaneceram intactos e corretos: eles processam o que chega, sem saber quantos nem quais itens existiam antes. Isso explica por que o ranking mudou sem qualquer edição neles.
3. Reorganizar os filtros mudaria o resultado: se o teste de experiência ocorresse depois do score, gastaríamos computo com candidatos já reprovados — a ordem do pipe é parte da responsabilidade da composição feita em `main.py`.
