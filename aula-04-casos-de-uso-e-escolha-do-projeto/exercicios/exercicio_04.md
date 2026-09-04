# Exercício 4 — A autópsia e a ficha

## Contexto

Este é o único exercício da disciplina **sem código**. Não é por ser mais leve — é porque a decisão que ele cobra vale mais que qualquer implementação que vocês fariam nesta semana.

Ele tem duas partes, e elas se alimentam:

- a **Parte 1** é a leitura crítica de um caso real de indústria. Ela existe para vocês descobrirem, na prática, o quanto um *case study* de fornecedor **não conta**;
- a **Parte 2** é a ficha do case de vocês. Ela existe porque é o contrato do resto do semestre.

Façam a Parte 1 primeiro. Ela muda a Parte 2.

## Parte 1 — A autópsia de um caso real

Cada grupo escolhe **um** caso público de agente em produção e escreve uma análise de **duas páginas**.

### Onde procurar

| Fonte | O que tem |
|---|---|
| [Catálogo do Google Cloud](https://cloud.google.com/transform/101-real-world-generative-ai-use-cases-from-industry-leaders) | mais de mil casos, por 11 setores × 6 tipos de agente |
| [*awesome-agent-failures*](https://github.com/vectara/awesome-agent-failures) | casos de **falha**, com análise — vale tanto quanto |
| Blogs de engenharia de empresas | os melhores: quem publica arquitetura, e não só resultado |
| Imprensa setorial | útil para o que a empresa **não** publicou |

**Regra de escolha:** não pode ser Klarna, Air Canada nem Replit — os três já foram dissecados em aula. E, de preferência, escolham um caso do **setor do case de vocês** — a Parte 1 vira pesquisa de campo para a Parte 2.

### O que a análise deve conter

**1. O que a empresa fez** (≈ meia página)
O problema que ela tinha, o sistema que construiu, quem é o usuário final.

**2. A arquitetura provável** (≈ meia página) — *o item central*
Que padrão de arquitetura está por trás? (vocabulário: espectro de autonomia da Aula 01 e §2.1 da nota 01 desta aula) Desenhe o diagrama, em ASCII mesmo, no formato dos que vocês viram em aula. **Liste as pistas** que sustentam a sua inferência — a percentagem que não é 100%, a menção a transbordo humano, o volume, o tipo de ação.

É inferência, e pode estar errada. O que se avalia é o **raciocínio**, não o acerto. Uma análise que diz *"provavelmente roteador, porque X, Y e Z — mas poderia ser um agente com ferramentas restritas se W"* vale mais que uma afirmação categórica sem pistas.

**3. Os números divulgados** (≈ quarto de página)
Quais foram, e **o que exatamente cada um mede**. Para cada número, responda: **quem publicou** (fornecedor tem interesse no número), e **o que foi perguntado ou contado** — "resolvido", "com sucesso" e "eficiência" são réguas da própria empresa, e é preciso procurar a definição.

**4. O que a divulgação NÃO conta** (≈ meia página) — *o item mais valioso*
Todo *case study* de fornecedor omite algo. Nomeie as omissões:
- qual é a **linha de base**? ("4× melhor" em relação a quê?)
- como "resolvido" / "com sucesso" está definido?
- quanto custou construir e quanto custa rodar?
- o que acontece nos casos que o sistema **não** atende?
- houve reversão, ajuste ou redução de escopo depois da publicação?

Não é preciso responder — é preciso **saber perguntar**. Uma lista de cinco perguntas que a empresa não respondeu é uma boa resposta a este item.

**5. As salvaguardas que este caso exige** (≈ quarto de página)
Com a tabela de modos de falha da Aula 01 (nota 03 §4) na mão: que ações são irreversíveis? onde precisaria de confirmação humana? o que aconteceria se o agente entrasse em laço? que dado sensível ele toca? Não é preciso saber implementar as salvaguardas — é preciso saber **quais faltam**.

### Formato

Markdown, no repositório do grupo, em **`docs/autopsia.md`** — toda a pesquisa sobre o case vive em `docs/`, como o enunciado do trabalho exige. Cite todas as fontes com link. Duas páginas — análise curta e afiada vale mais que resumo longo.

---

## Parte 2 — A ficha de case, versionada

A ficha preenchida à mão em sala, agora digital, em `docs/case.md` no repositório do grupo.

### O modelo

```markdown
# Case — [nome curto]
**Grupo:** [nomes] · **Versão:** v1 · **Data:** [data]

## O problema em uma frase
[se não couber em uma frase, o case está grande demais]

## Quem sofre com ele hoje
[um cargo, uma pessoa concreta — não "as empresas", não "os usuários"]

## O que o sistema faz
[3 a 5 linhas]

## Tipo de agente
[customer · employee · code · data · creative · security]

## Nível de autonomia pretendido
[workflow · roteador · agente]
**E por que este e não o de baixo:** [a justificativa é o que vale]

## Por que um agente, e não software comum
[duas frases: o que na tarefa exige decisão em tempo de execução]

## O ganho esperado — a venda
| Eixo | Linha de base (MEDIDA) | Alvo | Ganho | Volume |
|---|---|---|---|---|
| | | | | |

**A conta:** [de X para Y = Z%, sobre N casos por dia]
**É uma estimativa** — e será conferida na Parte 3.

## O ganho para o usuário
[não é o mesmo do negócio; se houver tensão entre os dois, diga]

## O outro lado da conta
[quanto custa rodar · quanto custa construir · o que o sistema vai
 errar, e quem paga por isso]

## As ferramentas (3 a 6)
| Ferramenta | Leitura ou escrita? | Reversível? |
|---|---|---|
| | | |

## O verificador
[COMO você vai saber que a saída está certa?
 Se não souber responder, o case não passa.]

## O critério de sucesso
[um número, com denominador]

## De onde vêm os dados
[reais · públicos · simulados — e, se simulados, como você preserva
 a DIFICULDADE do problema: qual é o caso divergente? qual é o
 registro inexistente? qual é o caso que NÃO deve disparar a ação?]

## Que dado sensível este case toca
[pessoal? financeiro? de saúde? "nenhum" é resposta válida e é vantagem]

## Espaço para o que ainda vem
- [ ] RAG — que conhecimento externo, e em que formato?
- [ ] Memória — precisa lembrar entre sessões? por quê?
- [ ] MCP — conversa com sistema de terceiro? qual?
- [ ] Evals — o que entra no conjunto de teste?

## O maior risco deste case
[o risco de verdade, não o de fachada — e o plano B]
```

### Os dois requisitos que vão além do preenchimento

**1. A ficha é versionada.** Ela nasce `v1` nesta semana e é a base do `case.md` que a **Parte 1 do trabalho** pede (`trabalho/01-primeira-entrega.md`). **Toda** mudança até a entrega da Parte 2 incrementa a versão e vira um commit cuja mensagem explica **por que** mudou:

```
case: v2 — troca o verificador de "avaliação humana" para conjunto
rotulado, porque a aula de evals mostrou que avaliação subjetiva não
serve de linha de base
```

É o mesmo princípio do versionamento de prompt cobrado desde a Aula 03, aplicado à decisão de projeto. **A trilha de como o case mudou de ideia é parte da avaliação** — ela é o registro do que vocês aprenderam, e é a única evidência disso que sobrevive ao semestre.

**2. Um `README.md` de meia página** dizendo, para quem chegar de fora: qual é o problema, qual a arquitetura pretendida e como se saberá que funcionou.

---

## O que será avaliado

**Parte 1 — a autópsia**

| Critério | O que se espera |
|---|---|
| Inferência de arquitetura | as **pistas** estão listadas e sustentam a conclusão |
| Leitura crítica dos números | procurou a **definição** por trás de cada métrica, em vez de repeti-la |
| **As omissões** | nomeou o que a empresa não contou — este item pesa mais |
| Salvaguardas | nomeou as que faltam usando a tabela de falhas da Aula 01, e não adjetivos |
| Fontes | tudo com link; distingue fonte primária de cobertura de imprensa |

**Parte 2 — a ficha**

| Critério | O que se espera |
|---|---|
| O problema cabe em uma frase | e a frase é específica |
| **A venda tem número e linha de base** | a base foi **medida**, e a conta tem denominador e volume |
| O usuário é uma pessoa | com cargo |
| **O verificador existe e é executável** | **reprova mais que todos os outros juntos** |
| O critério de sucesso é um número | com denominador |
| A autonomia é a menor que resolve | e a justificativa explica por que o nível abaixo não servia |
| Os dados simulados preservam a dificuldade | os três casos difíceis estão nomeados |
| Há espaço para o que vem | pelo menos dois, com o **quê** escrito |
| O risco é real | e tem plano B |

---

## Entrega

No repositório do grupo, até a próxima aula:

- `docs/autopsia.md` — a Parte 1;
- `docs/case.md` — a Parte 2, em `v1`;
- `README.md` — meia página, com o problema e como rodar.

---

## Dicas

- **Comecem pelo verificador.** Se vocês souberem como medir que a saída está certa, o resto do case se organiza sozinho. Se não souberem, nenhum outro campo salva.
- Na Parte 1, procurem o caso que tem **blog de engenharia**, e não só *case study* de marketing. A diferença de material é enorme, e o item 2 fica muito mais fácil.
- Desconfiem do seu próprio entusiasmo com um case. O melhor teste é ler os quatro anti-padrões do enunciado da Parte 1 do trabalho **procurando o seu tema neles**, e não procurando motivo para ele não estar lá.
- Não escolham o case mais impressionante. Escolham o que vocês conseguem **medir**. O impressionante que ninguém consegue avaliar vira, na semana 18, uma demonstração que funciona uma vez.
- E lembrem do teto: **30,3%**. Se o seu case só faz sentido com o agente acertando quase sempre, sozinho, ele não é viável neste semestre — nem, provavelmente, neste ano.
