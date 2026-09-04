# Trabalho da disciplina — Parte 1: Escolher e provar o terreno

> Leiam antes a **[visão geral do trabalho](00-visao-geral.md)**: esta é a primeira de três entregas sobre o **mesmo** sistema.

## Contexto

Vocês já sabem chamar um modelo, ajustar parâmetros, escrever prompt com contrato de saída, declarar ferramentas e montar o laço de um agente com orçamento e salvaguardas. E acabaram de ver o que o mercado fez com isso — o que funcionou, o que quebrou e por quê.

Agora vocês escolhem **o seu problema**.

Esta entrega tem um objetivo que não é o óbvio. Ela não existe para vocês mostrarem um agente impressionante — existe para vocês **descobrirem cedo se o tema é viável**, enquanto ainda dá tempo de ajustar. Um tema que se revela impossível na Parte 3 é um semestre perdido; na Parte 1, é uma tarde de conversa.

Por isso o agente desta parte é **deliberadamente simples**. O que se avalia aqui é a **qualidade da escolha** e a **honestidade da análise**, não o tamanho do código.

---

## O que entregar

Quatro coisas:

0. **O grupo** — até 4 alunos
1. **O tema** escolhido
2. **O detalhamento do tema**: o contexto de onde o agente vai ser usado
3. **A análise de modelos** que justifica a escolha do modelo
4. **Um agente simples rodando**, com prompt engineering e arquitetura básica

---

## 0. O grupo

**Até 4 alunos.** Grupos menores são permitidos, com a mesma exigência de profundidade.

Criem o repositório Git do grupo e coloquem os nomes no `README.md`. Todos os integrantes devem commitar — o histórico é evidência de participação, e ele será olhado.

---

## 1 e 2. O tema e o contexto

Este é o item mais importante da entrega, e o que mais decide o semestre de vocês.

Entreguem um documento chamado `case.md` respondendo aos campos abaixo. Ele é a evolução da ficha de case preenchida em aula — se vocês já a preencheram, comecem dela.

### 2.1 O problema

**O problema em uma frase.** Se não couber em uma frase, o tema está grande demais. O tamanho da frase é o teste.

**Quem sofre com ele hoje.** Um cargo, uma pessoa concreta — "o analista de suporte do plantão noturno", e não "as empresas" ou "os usuários". Um tema sem usuário identificável não tem como ter critério de sucesso.

**O contexto de onde o agente vai ser usado.** Esta é a parte que os grupos costumam escrever mal, então seja específico:

- **onde** o sistema roda: dentro de que processo, em que momento, acionado por quem;
- **o que existe antes e depois dele**: de onde vem a entrada, para onde vai a saída, quem consome o resultado;
- **o que acontece hoje sem ele**: como a pessoa resolve isso atualmente, e quanto tempo leva;
- **quais são as regras do domínio**: as políticas, prazos, limites e exceções que o sistema precisa respeitar;
- **o que dá errado hoje**: os casos difíceis, as divergências, as exceções que quebram o processo.

O último item é o mais valioso. **Os casos difíceis do domínio são o que vai definir o seu sistema** — e são o que vocês vão usar para testá-lo.

### 2.2 A interação com o usuário

A disciplina recomenda um caso com **complexidade real de interação**. Descrevam:

- o que o usuário **não** informa de primeira, e que o sistema precisa descobrir;
- o que acontece quando o que o usuário diz **contradiz** o que o sistema encontra;
- como o sistema decide que já sabe o suficiente para agir.

Se as respostas forem "o usuário informa tudo no formulário", reconsiderem o tema.

### 2.3 O sistema

**O que o sistema faz**, em 3 a 5 linhas.

**O nível de autonomia pretendido** — workflow, roteador ou agente — **e por que este e não o de baixo**. A segunda metade é o que vale: a regra da disciplina é *use a menor autonomia que resolve*, então vocês precisam explicar por que o nível anterior não resolvia.

**As ferramentas**, numa tabela:

| Ferramenta | O que faz | Leitura ou escrita? | Reversível? | Contra o que ela conversa |
|---|---|---|---|---|

A última coluna é requisito desta parte — ver o item 4.2.

### 2.4 O verificador

> **Como vocês vão saber que a saída está certa?**

**Este campo reprova mais que todos os outros juntos.** Respostas que valem:

- um **conjunto rotulado à mão** — 40 casos com a resposta que um humano daria;
- uma **regra de negócio** que confere o resultado;
- um **teste** que passa ou falha;
- a comparação com uma **decisão humana registrada**.

Resposta que não vale: *"dá para ver que está certo"*.

Verificador **construído** conta. O que não conta é não ter nenhum.

### 2.5 O critério de sucesso

**Um número, com denominador.** "Acerta em 80%" não diz nada; "acerta a categoria em 32 de 40 casos rotulados" diz.

Se o custo do erro for **assimétrico** — se errar para um lado for muito pior que para o outro —, digam isso e usem duas métricas. Exemplo: *"acerta a categoria em ≥32/40 **e** não deixa passar nenhum caso urgente"*.

### 2.6 Dados

**De onde vêm:** reais, públicos ou simulados.

Se simulados — e é o caso mais comum, e é legítimo —, expliquem **como vocês preservam a dificuldade do problema**. Dado de mentira fácil demais produz um sistema que só funciona no seu conjunto de teste. Nomeiem explicitamente:

- qual é o caso de **divergência** (o sistema diz uma coisa, o usuário diz outra);
- qual é o **registro inexistente** (o identificador que não existe);
- qual é o caso que **não deve** disparar a ação principal.

Os dados dos laboratórios da disciplina foram montados exatamente assim. Façam igual.

### 2.7 Dado sensível

Que dado sensível este tema toca — pessoal, financeiro, de saúde, sigiloso? Se não toca nenhum, digam isso; é uma vantagem do tema, não uma omissão.

Se toca, a regra é: **dado sensível não entra no repositório nem no contexto do modelo.** Ele é simulado.

### 2.8 Espaço para o que ainda vem

Marquem e **escrevam o quê**:

- [ ] **RAG** (Parte 2) — que conhecimento de domínio os agentes vão consultar, e em que formato ele existe hoje?
- [ ] **MCP** (Parte 2) — qual integração vai virar servidor MCP?
- [ ] **LangChain** (Parte 2) — que parte da orquestração?
- [ ] **Multiagente** (Parte 3) — quais seriam os agentes, e por que mais de um?

Um `[x]` sem texto não conta. "RAG: o regulamento interno, 40 páginas em PDF" conta.

### 2.9 O maior risco

O risco de verdade, não o de fachada. *"Pode ser que o modelo erre"* não é risco — é a premissa. *"A nossa única fonte é um PDF escaneado e o OCR pode não funcionar"* é risco, e vem com plano B.

---

## 3. A análise de modelos

Um documento `modelos.md`. O objetivo é **justificar uma escolha**, não catalogar o mercado.

### 3.1 Os candidatos

Escolham **três** modelos candidatos e compare-os nos eixos que importam **para o seu caso** — e diga por que esses eixos, e não outros:

| Eixo | Por que pode importar no seu caso |
|---|---|
| janela de contexto | o seu caso tem documento longo? histórico grande? |
| suporte a **tool calling** e a **saída estruturada** | é pré-requisito: sem isso não há agente |
| suporte a multimídia | só se o caso realmente pedir |
| capacidade de raciocínio | tarefa com passos de inferência, ou classificação direta? |
| custo por milhão de tokens (entrada e saída) | multiplicado pelo número de chamadas por execução |
| latência | há alguém esperando na frente da tela? |
| onde roda | API pública, nuvem contratada, local |
| política de dados | quem pode ver o que vocês enviam |

**Podem escolher um modelo diferente do Mistral usado nos laboratórios.** Se escolherem, esta seção é onde vocês defendem a decisão — e onde vocês assumem o custo dela.

### 3.2 A conta

Estimem o custo de **uma execução** do seu sistema:

```
tokens de entrada por chamada  × nº de chamadas por execução × preço de entrada
+ tokens de saída por chamada  × nº de chamadas por execução × preço de saída
= custo por execução
```

E depois: custo por 100 execuções, e custo estimado do semestre. Um sistema com 6 chamadas por execução custa seis vezes o que vocês imaginaram olhando o preço por token.

### 3.3 A verificação mínima

**Não confiem em leaderboard.** Rodem uma verificação própria, mesmo pequena: **cinco casos do seu domínio** nos três candidatos, com o mesmo prompt, e mostrem o resultado numa tabela.

Cinco casos não são uma medição estatística — e não é isso que se pede. O que se pede é que vocês tenham **olhado a saída dos três modelos no seu problema** antes de escolher. É o embrião do benchmark próprio que a Parte 3 vai cobrar de verdade.

### 3.4 A decisão

Qual modelo, e **em que condições vocês mudariam de ideia**. A segunda parte é o que separa uma escolha de um chute.

---

## 4. O agente simples

Aqui vocês escrevem código. **Pouco código.**

O objetivo é provar que o caminho existe: que as ferramentas são implementáveis, que o modelo escolhido dá conta, e que o problema responde ao tratamento. Não é para estar bonito nem completo.

### 4.1 O mínimo exigido

| Requisito | Detalhe |
|---|---|
| **2 a 4 ferramentas** | mais que isso é escopo da Parte 2 |
| **Ao menos uma ferramenta de escrita** | com confirmação, se a ação for irreversível |
| **Laço com estado explícito** | não guardem tudo na lista de mensagens |
| **Orçamento** | teto de passos, tokens e — se houver custo — de dinheiro |
| **Terminação registrada** | o programa diz **por que** parou |
| **Log da trajetória** | ferramenta, argumentos, resultado e erro de cada passo |
| **Prompts em arquivo** | em `prompts/`, versionados, com `prompt × modelo × parâmetros` carimbado |
| **Erro de ferramenta como dado** | retorno de erro que **ensina** o modelo a se corrigir, não exceção que derruba |

### 4.2 A integração com software tradicional

**Requisito desta parte:** pelo menos uma ferramenta do agente precisa conversar com **software tradicional** — não com o modelo, e não com um dicionário Python no meio do arquivo.

Vale qualquer um destes:

- uma **API HTTP** — sua, de terceiro, ou um servidor local que vocês subam;
- um **banco de dados** — SQLite serve, Postgres serve;
- um **serviço** próprio, mesmo trivial, rodando em outro processo;
- um **arquivo estruturado** lido por uma camada de acesso separada (o mais fraco dos quatro, aceito se a camada existir de verdade).

**Operação mock é permitida e esperada.** O ponto não é ter o sistema real — é o agente atravessar a fronteira do próprio processo e lidar com o que isso traz: erro de rede, dado ausente, formato inesperado, latência.

Na Parte 2 esta integração é **reescrita como servidor MCP**. Escolham algo que faça sentido reescrever.

### 4.3 O prompt engineering

Ainda não é o plano completo da Parte 2, mas já não é improviso. Para cada etapa em que o modelo é chamado, digam **no código, em comentário**:

- qual **técnica** está sendo usada (zero-shot, few-shot, CoT…) e **por que ela**;
- qual é o **contrato de saída** — formato, e o que é proibido;
- o que aquela etapa **impede** de dar errado.

E a regra que vale desde a aula de prompt engineering: **se vocês apagarem uma frase do prompt e não souberem dizer o que ela impedia, ela não estava fazendo nada.**

### 4.4 A arquitetura básica

Um diagrama — ASCII serve, e é preferido — mostrando: a entrada, as etapas, onde o modelo decide, onde o seu código decide, as ferramentas e as condições de parada.

E um parágrafo respondendo: **onde neste desenho está a decisão que justifica um agente em vez de um workflow?** Se não houver nenhuma, é um workflow — o que é aceitável na Parte 1, desde que vocês digam onde a decisão vai entrar na Parte 2.

### 4.5 A demonstração

O programa roda contra **pelo menos 4 casos** do seu domínio, escolhidos para serem diferentes:

1. o caso simples, que deve funcionar;
2. o caso de **divergência** (o sistema diz uma coisa, o usuário diz outra);
3. o caso do **registro inexistente** (erro de ferramenta que o modelo tem de contornar);
4. o caso que **não** deve disparar a ação principal.

O log das quatro execuções vai no repositório.

---

## Como esta parte será avaliada

Em ordem de peso:

| O que se avalia | O que se espera |
|---|---|
| **A qualidade da escolha do tema** | problema em uma frase · usuário concreto · **verificador que existe** · critério de sucesso com denominador · espaço declarado para o que vem |
| **A honestidade da análise de modelos** | três candidatos comparados nos eixos **do caso** · a conta de custo feita · os cinco casos rodados de verdade · a condição de mudar de ideia |
| **O agente rodando** | atende os requisitos de 4.1 · a integração de 4.2 é real · os 4 casos executados, com log |
| **A justificativa da arquitetura** | o nível de autonomia é o menor que resolve, **e vocês explicam por que o de baixo não servia** |
| **O rigor do prompt** | prompts em arquivo, versionados, com técnica e contrato declarados |

E o que **não** conta: quantidade de código, quantidade de ferramentas, sofisticação visual. Nada disso é o assunto desta entrega.

---

## Entrega

No repositório do grupo:

```
README.md          nomes do grupo · o problema em uma frase · como rodar
case.md            os itens 1 e 2
modelos.md         o item 3
prompts/           os prompts, versionados
src/               o agente
logs/              as 4 execuções demonstradas
dados/             os dados simulados (com os casos difíceis nomeados)
```

`README.md` precisa dizer **como rodar** em menos de cinco minutos, do zero, por alguém que nunca viu o projeto.

---

## Dicas

- **Comecem pelo verificador.** Se vocês souberem como medir que a saída está certa, o resto do tema se organiza sozinho. Se não souberem, nenhum outro campo salva o trabalho.
- **Escrevam os dados difíceis antes do agente.** Os quatro casos de 4.5 são o seu conjunto de teste, e escrevê-los primeiro força vocês a entender o domínio antes de escrever prompt.
- **Não escolham o tema mais impressionante. Escolham o que vocês conseguem medir.** O impressionante que ninguém consegue avaliar vira, na apresentação final, uma demonstração que funciona uma vez.
- **Leiam os quatro anti-padrões da [visão geral](00-visao-geral.md) procurando o seu tema neles** — e não procurando motivo para ele não estar lá.
- **Se o tema é o do TCC**, aproveitem: descrevam o contexto do item 2 com a profundidade que vocês já têm. É a parte em que vocês largam na frente.
- E lembrem da calibragem vista em aula: o melhor agente medido em tarefas profissionais longas concluiu **cerca de um terço** delas sozinho. **Um tema que só funciona se o agente acertar quase sempre, sem ajuda, não é viável.**
