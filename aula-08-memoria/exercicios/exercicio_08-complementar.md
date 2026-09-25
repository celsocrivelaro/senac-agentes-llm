# Exercício 8 (complementar) — O que não merece memória

> **Este é um dos dois exercícios complementares da aula 08.** O que vale
> nota é o [enunciado.md](exercicio_08.md), que é de projeto. Este trabalha sobre
> uma execução já dada e exercita uma decisão específica — **o que NÃO
> entra** —, que o exercício de código
> ([08-complementar-codigo.md](exercicio_08-complementar-codigo.md)) cobra no requisito
> 4 e que o de entrega cobra na Parte 2.

## Contexto

A taxonomia usual da memória de agentes tem três categorias: episódica,
semântica e procedural. Ela está incompleta, e a categoria que falta é a que
mais afeta o resultado: **não guardar**.

A maior parte do que uma execução produz é ruído — latência, contagem de
tokens, número de passos, tentativas descartadas. Guardar isso não é apenas
inútil: o ruído **compete por espaço na janela** com o que seria útil, e
memória preenchida com ruído é pior que memória vazia.

Classificar é fácil quando a informação é claramente um fato ou claramente
uma métrica. O exercício existe pelos casos em que não é.

## Parte 1 — Classificar as dez

Uma execução do agente de prestação de contas produziu as dez informações
abaixo. Classifique **cada uma** em `episódica`, `semântica`, `procedural` ou
`não guardar`.

| # | Informação |
|---|---|
| 0 | O funcionário F-088 viajou a Lisboa em março e em agosto de 2026. |
| 1 | Em 12/03/2026 a despesa D-4102 foi aprovada por estar dentro do teto. |
| 2 | A consulta ao histórico levou 240 ms. |
| 3 | Quando o recibo diverge do valor declarado, devolver antes de analisar o mérito. |
| 4 | O id do funcionário tem o formato F seguido de três dígitos. |
| 5 | A resposta da API veio com 1.820 tokens de entrada. |
| 6 | F-091 já teve uma despesa de transporte reprovada por falta de justificativa. |
| 7 | O modelo tentou consultar F-88 antes de acertar F-088. |
| 8 | O teto de refeição em viagem internacional é de R$ 260,00. |
| 9 | O agente concluiu a análise no terceiro passo. |

**Para cada item, justifique em uma linha.** A justificativa das classificadas
como `não guardar` é a parte que importa: indique **qual pergunta futura** essa
informação responderia — e por que essa pergunta nunca é feita.

> Não classifique por formato. "Levou 240 ms" e "custa R$ 260,00" são os dois
> números, e não têm o mesmo destino.

## Parte 2 — A que gera divergência

O item **7** é o que divide a sala, e é o mais instrutivo dos dez.

A tentativa fracassada de consultar `F-88` é ruído de **uma** execução: nunca
mais alguém vai perguntar o que o modelo tentou naquela terça-feira. Mas a
**generalização** dela — *"identificadores de funcionário têm o formato F
seguido de três dígitos"* — é memória procedural legítima, e é o item 4.

Responda:

1. Qual a diferença entre **registrar o incidente** e **extrair a regra**?
2. Quem faz essa extração no seu desenho — o agente, o código ou o humano? (As
   três políticas estão no `03-quem-escreve.py`.)
3. Escreva **mais dois pares** incidente → regra, do seu próprio domínio. Um
   deles deve ser um par em que a regra extraída seria **errada** se derivada
   de uma única ocorrência.

O item 3 é a armadilha: a memória procedural é a mais valiosa e a mais
perigosa, porque uma instrução ruim aprendida se aplica a **todas** as
execuções seguintes, sem que ninguém a revise.

## Parte 3 — O mesmo, no case do trabalho

Tome **uma** execução real (ou plausível) do agente do trabalho e liste
tudo o que ela produz — sem filtrar, inclusive o que obviamente é lixo. Mire
em doze a quinze itens.

Depois:

1. Classifique nas quatro categorias.
2. Calcule a **proporção de `não guardar`**. Proporção menor que metade indica
   filtragem durante a listagem: volte e inclua o que foi descartado.
3. Para as que sobraram, indique **em qual das três estruturas** cada uma seria
   gravada, e **por qual chave ou consulta** ela voltaria. Informação cuja
   recuperação posterior não se sabe fazer é, na prática, `não guardar`.

O critério do item 3 é o que fecha o exercício: **não existe "guardar por via
das dúvidas"**. Ou há uma pergunta futura que a informação responde, ou ela é
ruído com aparência de dado.

## Entrega

No repositório do trabalho, em `exercicios/aula-08-o-que-nao-guardar.md`:

- a tabela das dez, classificada, com a justificativa de uma linha em cada;
- os dois pares incidente → regra da Parte 2, incluindo o par em que a
  generalização seria errada;
- a lista da sua própria execução, com a proporção de `não guardar` e, para
  cada item guardado, a estrutura e a chave ou consulta que o recupera.

> **Onde isto reaparece:** o requisito 4 do [enunciado.md](exercicio_08.md) — *o
> que NÃO entra* — pede essa lista de exclusão aplicada ao agente do case. Este
> exercício é o rascunho dela.

## Dicas

- Comece pelas que são obviamente ruído. Elas calibram o critério para as
  duvidosas.
- Um teste rápido para `não guardar`: tente escrever a pergunta que a
  informação responderia, na voz de quem usa o sistema. Se a pergunta soar
  absurda — *"quantos milissegundos a consulta de março levou?"* —, a
  informação é ruído.
- Informação que muda a cada execução e não é sobre o **domínio** é quase
  sempre ruído de execução. A exceção são as escritas executadas, que precisam
  ser lembradas justamente para não se repetirem.
