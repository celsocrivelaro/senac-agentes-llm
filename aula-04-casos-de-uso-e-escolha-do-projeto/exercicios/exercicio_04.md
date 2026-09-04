# Exercício 4 — A escolha do case

## Contexto

Este é o único exercício da disciplina **sem código**. Não é por ser mais leve — é porque a decisão que ele cobra vale mais do que qualquer implementação que vocês fariam nesta semana.

Vocês vão **escolher o tema do trabalho da disciplina**. O que sair daqui é o sistema que vocês vão construir até o fim do semestre, em três entregas.

> **Leiam antes:**
> - [Visão geral do trabalho](../../trabalho/00-visao-geral.md) — o que é, as três partes, as regras
> - [Enunciado da Parte 1](../../trabalho/01-primeira-entrega.md) — o que a primeira entrega cobra
>
> Este exercício é o **começo da Parte 1**. O que vocês produzirem aqui vira o `docs/case.md` do trabalho — não é um documento descartável.

> ⚠️ **A escolha precisa ser validada comigo.** Nenhum grupo começa a construir antes disso. É por isso que o exercício existe agora, e não na semana em que o código começa: um tema ruim corrigido esta semana custa uma conversa; corrigido na semana 12, custa o semestre.

---

## O que entregar

Um documento `docs/case.md` no repositório do grupo, com **três seções** — e só estas três. Os demais campos do enunciado da Parte 1 (verificador, dados, riscos, análise de modelos) vêm depois, com o tema já aprovado.

---

## 1. O case — indústria e problema

**O setor.** Em que indústria isso acontece: saúde, logística, educação, varejo, jurídico, financeiro, indústria, governo, uma área interna de empresa. Escolham o setor que vocês **conhecem por dentro** — o que vocês já viram funcionar vale mais que o que parece moderno.

**O problema, em uma frase.** Se não couber em uma frase, o tema está grande demais. O tamanho da frase é o teste.

**O contexto de onde o problema vive.** Aqui é onde se separa um tema pensado de um tema chutado:

- **o que acontece hoje sem o sistema** — quem faz, como faz, quanto tempo leva;
- **as regras do domínio** — políticas, prazos, limites, alçadas, exceções que o sistema vai ter que respeitar;
- **o que dá errado hoje** — os casos difíceis, as divergências, aquilo que quebra o processo.

O último item é o mais valioso da seção. **Os casos difíceis do domínio são o que vai definir o sistema de vocês**, e são o que vocês vão usar para testá-lo.

**O que a indústria já faz com agentes nesse problema.** Procurem **dois ou três casos reais** do setor de vocês — no [catálogo do Google Cloud](https://cloud.google.com/transform/101-real-world-generative-ai-use-cases-from-industry-leaders), em blogs de engenharia, na imprensa setorial. Para cada um, meia página:

- o que a empresa fez e que número ela divulgou;
- **que padrão de arquitetura provavelmente está por trás** — com o vocabulário da nota 01 desta aula (§2.1) e o espectro da Aula 01;
- **o que a divulgação não conta** (qual a linha de base? como "resolvido" foi definido? quanto custou?).

Não é enfeite: é assim que vocês descobrem se o problema de vocês já foi resolvido, e como. E vai direto para o `docs/` do trabalho.

---

## 2. Os usuários, e como será a interação

Um sistema tem mais de um usuário, e eles querem coisas diferentes. O formato completo está no [§2.2 do enunciado da Parte 1](../../trabalho/01-primeira-entrega.md); o mínimo aqui é:

**A tabela de perfis** — normalmente de dois a quatro:

| Perfil | O que ele quer | O que ele sabe | O que ele **pode** fazer |
|---|---|---|---|

A última coluna é a que os grupos esquecem, e é a que importa: **quem pode aprovar uma ação irreversível?**

**O usuário principal** — aquele para quem o sistema é desenhado quando os interesses conflitam.

**A interação com ele**, concretamente: por onde (chat, formulário, terminal, e-mail); **quem começa** a conversa; quantas trocas até resolver; o que o sistema devolve e em que formato; e o que o usuário vê quando o sistema **não** consegue resolver.

> **O jeito mais rápido de fazer esta seção é escrever um diálogo de exemplo**, com as falas dos dois lados, do início ao fim. Vale mais que três parágrafos de descrição — e vocês vão reaproveitá-lo como caso de teste.

E a pergunta que decide se o tema serve à disciplina: **o que o usuário não informa de primeira, e que o sistema precisa descobrir?** Se a resposta for "nada, ele preenche o formulário", reconsiderem o tema.

---

## 3. Os ganhos esperados

Vocês vão ter que **vender o sistema** a alguém que não vai ler o código de vocês. A tabela de eixos e os exemplos estão no [§2.5 do enunciado da Parte 1](../../trabalho/01-primeira-entrega.md); o mínimo aqui é:

**Por que um agente, e não software comum.** Duas frases: o que na tarefa exige decisão em tempo de execução. Se o problema se resolve com um formulário, uma consulta e três `if`, **a resposta certa é o formulário**.

**Um ou dois eixos de ganho**, com a conta à vista:

| Eixo | Linha de base (**medida**) | Alvo | Ganho | Volume |
|---|---|---|---|---|

Escolham entre: tempo por tarefa · velocidade de processo · produtividade · cobertura (% sem humano) · redução de carga · erro e retrabalho · custo por transação · receita.

**A linha de base é medida, não estimada.** Cronometrem dez casos, contem uma fila real, perguntem a quem faz. É o item que mais some das entregas, e sem ele a venda é opinião.

**O ganho para o usuário**, que não é o mesmo do negócio — e, se houver tensão entre os dois, digam a tensão.

Duas regras que separam uma venda boa de uma ruim:

- **número específico e feio vence número redondo e bonito** — *"de 12 para 2 minutos em 200 chamados/dia"* tem denominador; *"+50% de eficiência"* não tem;
- **prometam o eixo que vocês conseguem medir.** Promessa que não dá para conferir reprova pelo mesmo motivo que a falta de verificador.

---

## Os quatro anti-padrões

Antes de entregar, leiam a lista da [visão geral do trabalho](../../trabalho/00-visao-geral.md) **procurando o seu tema nela** — e não procurando motivo para ele não estar:

| Anti-padrão | Sintoma |
|---|---|
| **Sem verificador** | vocês não conseguem responder "como sei que está certo" sem escrever "dá para ver" |
| **Dado que vocês não têm** | o plano contém a frase "vou pedir acesso" |
| **Grande demais** | não cabe em uma frase; mais de dez ferramentas |
| **Produto de terceiro** | o esforço é interface e infraestrutura, não arquitetura de agente |

---

## Entrega

No repositório do grupo, até a próxima aula:

```
README.md          nomes do grupo e o problema em uma frase
docs/
  case.md          as três seções deste exercício
  fontes.md        os casos da indústria consultados, com link
```

Markdown, versionado. Nada de `.docx` nem `.pdf` — a pesquisa do case vive em `docs/` e precisa de `git diff`, como o enunciado do trabalho exige.

E a regra que vale a partir daqui: **o tema pode ser refinado até a entrega da Parte 2. Trocado, não.** Refinar é mudar escopo, ferramenta, nível de autonomia ou fonte de dados — e é esperado. Começar de novo depois disso não dá tempo.

---

## Dicas

- **Comecem pelo diálogo de exemplo.** Escrever a conversa do início ao fim expõe em dez minutos o que três parágrafos de descrição escondem — inclusive que o sistema não precisava conversar.
- **Meçam a linha de base esta semana**, enquanto ninguém está escrevendo código. Depois que o desenvolvimento começa, ninguém volta para cronometrar.
- **Escolham o setor que vocês conhecem.** O tema que parece moderno e que ninguém do grupo entende vira um semestre pesquisando domínio em vez de construindo sistema.
- **Não escolham o tema mais impressionante. Escolham o que vocês conseguem medir.** O impressionante que ninguém consegue avaliar vira, na apresentação final, uma demonstração que funciona uma vez.
- E lembrem da calibragem vista em aula: o melhor agente medido em tarefas profissionais longas concluiu **cerca de um terço** delas sozinho. **Um tema que só funciona se o agente acertar quase sempre, sem ajuda, não é viável.**
