# Exercício 11 — A avaliação do agente do seu trabalho

## Contexto

O seu agente já mudou muitas vezes desde a Aula 05: de prompt, de corte, de
memória, de framework. Cada mudança foi defendida com um exemplo que funcionou
ou com um argumento — e nenhuma foi conferida contra as anteriores. Esta aula
mostrou por que isso não basta: a saída é uma distribuição, acertar uma vez
não é acertar, e a média pode esconder uma classe inteira de casos que
quebrou.

O exercício é decidir **como o seu agente vai ser avaliado** — que casos, que
critério para cada coisa que importa, que juiz, e que regra separa melhora de
ruído. No papel, antes da primeira linha de código.

> **Não se escreve código aqui.** A entrega é um texto.
>
> O [enunciado.md](enunciado.md) é o par prático: lá a suíte que você desenhar
> aqui é implementada e executada contra duas versões do sistema. Desenhar
> antes evita o erro mais comum desta aula — escolher o que medir pelo que é
> fácil de medir.

---

## O que responder

Cinco perguntas. Responda na ordem — cada uma depende da anterior.

### 1. O que significa "acertar" no seu case

Liste as **propriedades** que precisam estar certas para que uma execução do
seu agente sirva a quem a pediu. Não "a resposta é boa": coisas que se
conferem uma a uma.

| Tipo de propriedade | Exemplo |
|---|---|
| **o resultado** | o veredito, a classificação, o valor calculado, o registro criado |
| **a fidelidade** | o que foi afirmado está no documento recuperado |
| **o formato** | a saída respeita o contrato que outro sistema consome |
| **o caminho** | consultou a fonte certa, não repetiu ferramenta, não pulou a confirmação |
| **a recusa** | disse "não sei" ou "não posso" quando devia |
| **o tom e a forma** | o que sobra, e só o que sobra, quando nada acima cobre |

Para cada propriedade, escreva **o que é falhar** com um exemplo concreto do
seu domínio. Se você não consegue descrever a falha, a propriedade ainda não
está definida — e nenhum avaliador vai defini-la por você.

> **Comece pelo que custa caro errar.** Um agente que aprova o reembolso
> errado e um agente que responde num tom frio não falham da mesma forma. Diga
> quais propriedades são as que, se falharem, alguém perde dinheiro, tempo ou
> confiança.

### 2. Os casos — o *golden dataset* v1

Monte uma tabela com **quinze a vinte casos** do seu agente. Para cada um:

- a **entrada**;
- o **esperado**: a saída exata, um rótulo, ou o critério que a resposta deve
  satisfazer;
- a **origem**: erro observado durante o desenvolvimento, caso sintético
  (escrito por você ou gerado por modelo **e revisado**), ou incidente real, se
  o agente já tiver sido usado por alguém;
- a **classe**: o agrupamento que vai permitir ler o resultado por fatia;
- **por que o caso existe** — o que ele pega que os outros não pegam.

Você não parte do zero. Procure no próprio repositório os **embriões** que a
[nota 01](../notas-de-aula/01-por-que-e-diferente.md) nomeou: os exemplos de *few-shot* da Aula 03, as perguntas com
resposta conhecida da Aula 06, o par `recall` × fidelidade da Aula 07, as tarefas
fixas da comparação da Aula 09 e os casos demonstrados na Parte 2. Diga quais
deles entram.

**Exigências de composição**, que valem mais que o tamanho:

- **casos de recusa** — perguntas fora do escopo ou sem resposta na base;
- **casos de borda** — o valor no limite, o documento ambíguo, a entrada
  malformada;
- **pelo menos três classes**, com mais de um caso em cada.

> **Cinquenta falhas reais revisadas valem mais que quinhentos casos
> sintéticos não revisados.** Se quase todos os seus casos são sintéticos,
> diga isso — e diga de onde virão os reais quando o serviço estiver no ar, na
> Aula 12.

### 3. O nível de cada propriedade

Para cada propriedade da pergunta (1), escolha **o nível mais barato que a
responde** — determinístico, trajetória, resultado ou juiz LLM — e justifique.

| Propriedade | Nível | Como se confere | Por que não um nível mais barato |
|---|---|---|---|
| … | … | … | … |

A última coluna é a que vale. Veredito enumerado se confere com `==`; formato,
com *schema*; citação, conferindo se o artigo foi de fato recuperado; registro
criado, olhando o estado do mundo depois da execução. **Toda propriedade que
for para o juiz precisa dizer por que nada disso resolveria** — o juiz é o
nível mais caro e o único que também precisa ser avaliado.

Diga também se o seu agente precisa de avaliação de **trajetória**. Um agente
que age sobre o mundo e acerta pelo caminho errado vai errar quando o caminho
mudar; se o seu agente só responde texto, talvez o resultado baste — diga qual
é o caso.

### 4. O juiz, e como ele vai ser validado

Escolha **uma** propriedade sem gabarito, das que ficaram com o juiz, e
escreva:

- a **rubrica**: o que conta como sim e o que conta como não, em critério
  **binário**;
- **um exemplo de cada lado**, tirado dos seus casos;
- **como o juiz vai ser validado**: quantas saídas serão rotuladas por
  pessoas, quem rotula, e qual a **concordância mínima** abaixo da qual o juiz
  não é adotado — decidida agora, antes de medir.

> **Quem rotula importa.** Se o único rotulador é o grupo, a validação mede o
> quanto o juiz concorda com vocês — o que é útil, e é diferente de medir o
> quanto ele está certo. Diga quem, no domínio do case, poderia rotular uma
> amostra.

Liste também, para este juiz, **qual dos vieses documentados mais o ameaça**
— posição, verbosidade ou auto-preferência — e o que a rubrica faz contra ele.

### 5. O ruído, e a regra que separa melhora de acaso

- **De onde vem o não determinismo do seu agente?** Parâmetros de amostragem,
  ferramentas externas que mudam, memória que cresce entre execuções. Para
  cada fonte presente no seu case, diga se a suíte vai **controlá-la**
  (temperatura fixa, ferramenta simulada, memória congelada) ou **medi-la**;
- **Quantas repetições por caso**, e por quê;
- **`pass@k` ou `pass^k`?** Qual das duas é a medida certa para o seu agente,
  e por quê. Um agente que age sobre o mundo precisa acertar **todas** as
  vezes, não **alguma** vez;
- **A regra de promoção**: uma mudança no sistema só é aceita se… Escreva a
  regra com número — o ganho mínimo acima do ruído, e a queda máxima tolerada
  em qualquer fatia —, datada, antes de existir a primeira comparação.

> **A regra escrita depois do resultado não é regra.** É a descrição do
> resultado que você queria.

---

## Entrega

Um documento em `exercicios/aula-11-avaliacao.md`, no repositório do trabalho.

Texto corrido, com as cinco perguntas como seções e a tabela de casos da
pergunta (2) inteira. Tabelas onde couberem. **Três a cinco páginas** — se
passar muito disso, provavelmente há descrição de técnica onde deveria haver
decisão sobre o seu agente.

Este desenho vai mudar quando a suíte rodar pela primeira vez — é esperado, e é
o mesmo que aconteceu com a arquitetura da Aula 05 e com a base da Aula 06.
Vale versionar este documento em vez de sobrescrevê-lo.

> **Onde isto reaparece:** a **Parte 3 do trabalho** começa pela suíte (§0) e
> pede `docs/evals.md`, com os níveis justificados, a matriz de concordância do
> juiz e os desacordos lidos. O que você escreve aqui é o rascunho dele, e o
> [enunciado.md](enunciado.md) é a primeira implementação.

---

## Critérios

| | |
|---|---|
| **Definiu o acerto** | cada propriedade tem a falha descrita com um exemplo do domínio |
| **Declarou a origem** | todo caso tem origem e classe, e há casos de recusa e de borda |
| **Escolheu o nível mais barato** | toda propriedade no juiz justifica por que os níveis abaixo não bastam |
| **Planejou validar o juiz** | rubrica binária, exemplos dos dois lados, e o limiar de adoção decidido antes |
| **Escreveu a regra antes** | a regra de promoção tem número, considera as fatias e está datada |

---

## Dicas

- **Comece pelas falhas que você já viu.** Toda vez que o seu agente errou
  durante o desenvolvimento e você corrigiu, aquele erro é um caso — e é o
  mais valioso que você tem, porque não depende de imaginação.
- **Um caso que todas as versões acertam não ensina nada.** Se a tabela da
  pergunta (2) só tem casos fáceis, a suíte vai aprovar qualquer mudança.
- **Desconfie da propriedade "a resposta é boa".** Quase sempre ela se
  desmonta em duas ou três propriedades que se conferem sem juiz.
- **Escreva a rubrica como se fosse para uma pessoa nova na equipe.** Se uma
  pessoa não conseguiria aplicá-la sem perguntar, o juiz também não consegue.
- **Não tente acertar.** É a `v1`, e ela existe para estar errada de um jeito
  que você consiga enxergar depois.
