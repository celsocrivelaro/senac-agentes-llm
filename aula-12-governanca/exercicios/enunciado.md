# Exercício 12 — Governança: o *model card* do agente, com evidência

## Contexto

Um *model card* é fácil de escrever e difícil de sustentar. O formulário se
preenche em minutos; o que ele afirma — "o agente nunca aprova acima de tal
valor", "o dado pessoal não sai do país", "toda recusa passa por um humano" —
só vale se o sistema se comportar assim e se houver como provar.

Este exercício pede o *model card* do agente do grupo, nos doze campos da
[nota sobre provar a decisão](../notas-de-aula/02-provar-a-decisao.md) (§4),
com uma exigência a mais: **toda afirmação verificável do documento aponta
para a evidência que a sustenta** — a linha de código, o teste, o registro de
uma execução, o procedimento executado. O documento deixa de ser texto sobre
o sistema e passa a ser o índice das provas.

> **Questão a ser respondida ao final:** um auditor que nunca falou com o
> grupo, com o *model card* e o repositório na mão, consegue decidir se uma
> decisão concreta do agente estava dentro da autoridade dele?

## Objetivo

No repositório do trabalho:

1. escrever o *model card* do agente nos doze campos;
2. especificar a **autoridade** do agente e mostrar **onde cada limite é
   aplicado**;
3. **inventariar os dados pessoais** que o sistema trata;
4. **declarar as lacunas**: toda afirmação sem evidência.

## Requisitos

### 1. O *model card*, nos doze campos

Crie `docs/model-card.md` com os doze campos abaixo. Cada campo tem duas
partes:

- **a afirmação**: o que o sistema é ou faz;
- **a evidência**: o caminho no repositório (arquivo e linha, teste, registro
  de execução, saída de comando) que a sustenta — ou a marca **LACUNA**, com o
  que falta para sustentá-la.

Os campos:

- **Identificação** — nome do sistema, versão, data e os responsáveis
  nomeados: quem responde pelo agente quando algo dá errado.
- **Finalidade** — o que o agente decide ou executa, para quem, e os usos
  **fora de escopo**: o que ele não deve ser usado para fazer.
- **Autoridade** — o que o agente está autorizado a fazer e os limites dessa
  autorização: escopo, ações proibidas, credencial, limite de gasto, limiar de
  aprovação humana, quem pode suspendê-lo. É o campo detalhado no requisito 2.
- **Ferramentas e integrações** — cada ferramenta e servidor MCP que o agente
  usa, com a permissão que cada um concede (leitura, escrita, envio a
  terceiros).
- **Modelo e provedor** — o modelo, com a versão datada e não o *alias*
  ("-latest"); o provedor; a região onde os dados são processados; e os termos
  de retenção e de uso dos dados para treino.
- **Dados** — as fontes de dados, o dado pessoal tratado, a base legal, onde
  cada dado fica e por quanto tempo. É o campo detalhado no requisito 3.
- **Avaliação** — a suíte que mede o agente, a versão dela, os resultados
  agregados e por fatia, e a concordância do juiz com a avaliação humana.
- **Limites conhecidos** — os modos de falha já observados e os casos em que o
  agente não deve ser usado, mesmo dentro da finalidade.
- **Riscos aceitos** — cada risco que o grupo decidiu aceitar, com a razão e o
  que o limita; aceitar um risco é uma decisão registrada, e não um
  esquecimento.
- **Supervisão humana** — o que pausa para aprovação, quem aprova, em quanto
  tempo, e como se interrompe o agente.
- **Registro e retenção** — o que o sistema registra a cada decisão (entrada,
  contexto recuperado, versões, ferramentas, saída), por quanto tempo guarda e
  quem tem acesso ao registro.
- **Histórico de mudanças** — cada versão do sistema, o que mudou e quem
  aprovou a mudança.

Afirmação sem evidência e sem marca de lacuna é o erro mais grave do
exercício: é o documento dizendo do sistema algo que ninguém verificou.

### 2. A autoridade, com o ponto de aplicação

No campo **Autoridade**, especifique o agente como a
[nota sobre governar a ação](../notas-de-aula/01-governar-a-acao.md) (§8)
faz com o auditor de despesas: escopo, ações proibidas, credencial e seus
privilégios, limite de gasto, limiar de aprovação humana, comportamento quando
um controle falha, e quem pode suspender o agente.

Para **cada linha**, declare **onde o limite é aplicado**:

| Onde | O que vale como evidência |
|---|---|
| ausência de ferramenta | o agente não tem a ferramenta que executaria a ação |
| código | a linha que bloqueia, e um teste que a exercita |
| credencial | o privilégio da chave ou do usuário de serviço, e o que ela não alcança |
| plataforma | a configuração (limite de gasto do provedor, permissão do banco) |
| prompt | a instrução no prompt de sistema |

Limite aplicado **só no prompt** é declarado como tal, e conta como lacuna: o
modelo pode desobedecê-lo, e nenhum controle fora do modelo o garante.

Revise a lista de ferramentas e credenciais contra o que o agente **precisa**:
o que sobra é privilégio a remover, ou a justificar no campo de riscos aceitos.

### 3. Os dados pessoais, inventariados

No campo **Dados**, mapeie onde o dado pessoal **entra**, onde **fica** e para
onde **vai**: prompt, índice, memória, registro de execução e provedor
externo. Para cada lugar: o dado, a finalidade, a base legal (LGPD, art. 7º) e
o prazo de guarda. Aponte o que fere a minimização — dado guardado sem
finalidade, ou por mais tempo do que ela exige.

### 4. As lacunas, numa lista

No fim do *model card*, uma seção **Lacunas** reúne toda marca LACUNA do
documento, cada uma com a correção proposta e a prioridade. Lacuna declarada
é resposta correta; lacuna escondida não é.

## Critérios

- toda afirmação verificável do *model card* tem evidência, ou está marcada
  como lacuna;
- a evidência existe no repositório e sustenta o que o campo afirma;
- cada limite da autoridade tem o ponto de aplicação declarado, e os
  aplicados só no prompt estão marcados como lacuna;
- o inventário cobre todos os lugares por onde o dado pessoal passa, com
  finalidade, base legal e prazo.

## Entrega

No repositório do trabalho:

- `docs/model-card.md`, com os doze campos, as evidências e a seção de
  lacunas;
- as evidências referenciadas: código, testes, registros de execução e saídas
  de comando.

> **Onde isto reaparece:** a **Parte 3 do trabalho** defende o sistema do
> grupo como produto. O *model card* é a peça que um terceiro lê primeiro.

## Dicas

- Comece por uma execução real do sistema e veja o que o registro guarda dela:
  isso responde, em meia hora, a metade dos campos — e mostra o que falta.
- Evidência boa é a que outra pessoa consegue abrir: caminho de arquivo com
  linha, nome de teste, identificador da execução no registro, comando com a
  saída copiada.
- "O modelo foi instruído a não fazer X" descreve o prompt, não o sistema. Se
  é só isso que existe, o campo é lacuna.
- Um *model card* gerado por IA a partir do código costuma afirmar mais do que
  o sistema faz. Confira cada afirmação contra a evidência antes de entregar.
