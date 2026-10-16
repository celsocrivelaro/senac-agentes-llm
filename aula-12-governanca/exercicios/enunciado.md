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
> decisão concreta do agente estava dentro da autoridade dele — e consegue
> atender a um pedido de revisão e a um de eliminação?

## Objetivo

No repositório do trabalho:

1. escrever o *model card* do agente nos doze campos;
2. especificar a **autoridade** do agente e mostrar **onde cada limite é
   aplicado**;
3. **reconstruir uma decisão real** a partir do registro do sistema;
4. **inventariar os dados pessoais** e testar a **revisão** e a
   **eliminação**;
5. **declarar as lacunas**: toda afirmação sem evidência.

## Requisitos

### 1. O *model card*, nos doze campos

Crie `docs/model-card.md` com os doze campos da nota: identificação,
finalidade, autoridade, ferramentas e integrações, modelo e provedor, dados,
avaliação, limites conhecidos, riscos aceitos, supervisão humana, registro e
retenção, histórico de mudanças.

Cada campo tem duas partes:

- **a afirmação**: o que o sistema é ou faz;
- **a evidência**: o caminho no repositório (arquivo e linha, teste, registro
  de execução, saída de comando) que a sustenta — ou a marca **LACUNA**, com o
  que falta para sustentá-la.

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

### 3. Uma decisão real, reconstruída

Escolha **uma execução real** do sistema — uma decisão que afeta alguém — e
responda, **só com o que o registro guarda**:

1. qual versão do modelo, do prompt e do índice produziu a saída;
2. que contexto foi recuperado e usado;
3. que ferramentas foram chamadas, com que argumentos, e que política estava
   em vigor;
4. que dado pessoal entrou no contexto e para onde foi enviado;
5. se houve revisão humana, e de quem.

Cada pergunta que o registro não responde é uma **lacuna**, com a correção
proposta: o que passar a registrar, e com que identificador. O campo
**Registro e retenção** do *model card* aponta para esta reconstrução.

### 4. Os dados pessoais, inventariados

No campo **Dados**, mapeie onde o dado pessoal **entra**, onde **fica** e para
onde **vai**: prompt, índice, memória, registro de execução e provedor
externo. Para cada lugar: o dado, a finalidade, a base legal (LGPD, art. 7º) e
o prazo de guarda. Aponte o que fere a minimização — dado guardado sem
finalidade, ou por mais tempo do que ela exige.

### 5. Revisão e eliminação, executadas

Execute os dois pedidos de um titular e registre o resultado:

- **revisão de decisão automatizada** (LGPD, art. 20): a partir da
  reconstrução do requisito 3, produza a explicação da decisão que seria
  entregue ao titular — critérios e procedimentos, sem expor segredo de
  negócio — e diga quem faria a revisão;
- **eliminação**: elimine os dados de um usuário de teste e mostre o
  **antes e o depois** em cada lugar do inventário.

Declare o conflito entre eliminar e provar: o que foi eliminado, o que foi
**preservado como evidência** da decisão, e com que justificativa e por
quanto tempo.

### 6. As lacunas, numa lista

No fim do *model card*, uma seção **Lacunas** reúne toda marca LACUNA do
documento, cada uma com a correção proposta e a prioridade. Lacuna declarada
é resposta correta; lacuna escondida não é.

## Critérios

- toda afirmação verificável do *model card* tem evidência, ou está marcada
  como lacuna;
- a evidência existe no repositório e sustenta o que o campo afirma;
- cada limite da autoridade tem o ponto de aplicação declarado, e os
  aplicados só no prompt estão marcados como lacuna;
- a decisão reconstruída é real, e as perguntas sem resposta viram lacunas
  com correção;
- a eliminação foi executada, com o antes e o depois, e o conflito com a
  preservação de evidência está justificado.

## Desafios opcionais

**A. Dois limites aplicados no código.** Implemente fora do modelo dois
limites da autoridade — por exemplo, um limite de gasto por janela que bloqueia
a execução e um limiar que pausa para aprovação humana — e escreva testes que
mostram que eles funcionam **mesmo quando o prompt pede para ignorá-los**. Os
testes entram no *pipeline* de integração contínua do grupo como portão.

**B. O auditor externo.** Troque o *model card* com outro grupo. Cada grupo,
sem falar com o outro, escolhe uma decisão do sistema alheio e responde, só
com o documento e o repositório: a decisão estava dentro da autoridade do
agente? Registre o que conseguiu verificar e onde o documento não bastou.

## Entrega

No repositório do trabalho:

- `docs/model-card.md`, com os doze campos, as evidências e a seção de
  lacunas;
- as evidências referenciadas: código, testes, o registro da execução
  reconstruída e a saída da eliminação (antes e depois).

> **Onde isto reaparece:** a **Parte 3 do trabalho** defende o sistema do
> grupo como produto. O *model card* é a peça que um terceiro lê primeiro.

## Dicas

- Comece pelo requisito 3, e não pelo 1: reconstruir uma decisão mostra, em
  meia hora, o que o registro guarda e o que falta — e isso decide metade dos
  campos.
- Evidência boa é a que outra pessoa consegue abrir: caminho de arquivo com
  linha, nome de teste, identificador da execução no registro, comando com a
  saída copiada.
- "O modelo foi instruído a não fazer X" descreve o prompt, não o sistema. Se
  é só isso que existe, o campo é lacuna.
- O prazo de guarda do registro e a eliminação puxam em sentidos opostos.
  Decida por finalidade: o que é preciso guardar para responder a uma
  contestação, e por quanto tempo.
- Um *model card* gerado por IA a partir do código costuma afirmar mais do que
  o sistema faz. Confira cada afirmação contra a evidência antes de entregar.
