# IA Aplicada com LLMs — Aula 03: Prompt engineering — O prompt como código: versionado e testado

## Introdução

As duas notas anteriores trataram o prompt como um artefato que você **escreve**. Esta trata dele como um artefato que **vive em produção** — e que, portanto, precisa sobreviver ao que todo código de produção enfrenta: **alguém vai editar aquele texto.**

A pergunta desta nota é uma só, e ela é mais difícil do que parece:

> **Como você garante que a edição não quebrou nada?**

Em código, a resposta é conhecida: versionar, revisar, testar. O que muda aqui — e é o motivo pelo qual a nota existe — é que **as três coisas funcionam de forma diferente quando o artefato é um prompt**. Versionar exige versionar mais do que o texto. Revisar lendo o diff não pega o que importa. E o teste é *flaky* por natureza, o que assusta quem vem de teste determinístico.

> **Uma delimitação, decidida para esta disciplina:** o prompt também é a superfície por onde o sistema é **atacado** — texto vindo de fora pode agir como instrução, porque o modelo não distingue uma coisa da outra. Isso é **prompt injection**, e é assunto da **aula de segurança**, com o catálogo do OWASP e as defesas de arquitetura. Aqui a gente enuncia o fato (nota 01, §3) e para.

---

## Objetivos de aprendizagem

Ao final desta nota você deve ser capaz de:

- **Explicar** por que prompt, modelo e parâmetros formam **uma** unidade versionável.
- **Justificar** a suíte de regressão pelo **raio de alcance não-local** da mudança de prompt.
- **Escrever** uma suíte de regressão com critério por **propriedade** e tolerância **k de N**.
- **Decidir** entre manter o prompt em arquivo versionado ou em um *prompt registry*.
- **Reconhecer** a suíte como o **embrião do eval** que vem no fim do curso.

---

## Desenvolvimento teórico

### 1. Onde o prompt mora hoje

O prompt determina o comportamento do sistema tanto quanto o código determina. Mas compare como as duas coisas são tratadas no time típico:

| | Código | Prompt |
|---|---|---|
| Onde vive | arquivo versionado | string no meio de um `.py` — ou um campo de texto numa interface |
| Passa por review | sim | não |
| Tem teste | sim | não |
| Qual versão está em produção? | `git log` | *"acho que a Ana mexeu semana passada"* |
| Dá para reverter | `git revert` | — |

Nada disso é por má-fé. É porque prompt **parece** conteúdo, não código. Está em português, qualquer pessoa lê, e mudar parece inofensivo.

---

### 2. Três eixos, uma unidade

Aqui a Aula 02 volta. O comportamento do seu sistema é produto de **três coisas que variam de forma independente**:

```
   comportamento  =  prompt  ×  modelo  ×  parâmetros
                       │         │            │
                       │         │            └─ alguém "ajustou" a temperatura
                       │         └─ o alias -latest mudou sozinho
                       └─ alguém "melhorou" o texto
```

Na Aula 02 (nota 01, §7.2) você viu que `mistral-small-latest` é uma **dependência sem versão** — o provedor atualiza e o seu código chama outra coisa. Esta nota acrescenta: **o prompt é exatamente o mesmo tipo de dependência**, e os parâmetros também.

A consequência é dura e imediata:

> **Se você não versiona os três juntos, não consegue reproduzir bug nenhum.**

O resultado piorou na terça. Foi o prompt que alguém editou, o modelo que o provedor atualizou, ou o `top_p` que entrou num PR de "ajuste fino"? Sem os três registrados, a resposta é um palpite — e a correção também.

Na prática, isso significa que a unidade versionada não é o texto solto:

```yaml
# prompts/extracao-chamado/v3.yaml
versao: 3
modelo: mistral-small-2506          # versão DATADA, não o alias
parametros:
  temperature: 0
  max_tokens: 120
  response_format: {type: json_schema, json_schema: {...}}
template: |
  Classifique a mensagem em: {categorias}.
  ...
```

#### 2.1 "Mas o git já versiona o arquivo — para que o `versao: 3`?"

É a primeira pergunta de quem já usa git, e é uma boa pergunta.

O git versiona **o arquivo**. O que você precisa identificar é outra coisa: **a combinação que produziu um resultado**.

Repare na diferença. O `git log` responde *"como este arquivo mudou ao longo do tempo?"*. A pergunta que aparece em produção é outra:

> Um cliente reclamou de uma resposta errada que o sistema deu **terça-feira às 14h20**.
> Qual prompt, com qual modelo e quais parâmetros, gerou aquilo?

O git sozinho não responde. Ele sabe o que o arquivo continha em cada commit, mas não sabe **qual commit estava rodando** naquele momento — nem que modelo o provedor estava servindo, nem se alguém tinha mexido no `temperature`.

O número de versão resolve isso porque ele é **um nome para a combinação inteira**, e esse nome pode viajar:

```python
resultado = extrair(mensagem)

log.info("extracao concluida",
         extra={"prompt_versao": CONFIG["versao"],    # <- o carimbo
                "modelo": CONFIG["modelo"],
                "chamado_id": chamado.id})
```

Com o carimbo no log de cada requisição, a pergunta de terça vira uma consulta: *"versão 3, `mistral-small-2506`"* — e agora sim o `git log` é útil, porque você sabe **o que procurar**.

**Quando incrementar?** A regra prática é a mesma de qualquer artefato versionado: **quando o comportamento muda**. Corrigir um erro de digitação num comentário não muda; reescrever uma instrução, acrescentar um exemplo ou trocar o modelo mudam. E o teste da §5 é o árbitro: **se a suíte deu um resultado diferente, era mudança de comportamento** — e merecia versão nova.

> Em times pequenos, dá para começar sem o número e usar o **hash do commit** como identificador. Funciona, e tem a vantagem de ser automático. O número explícito ganha quando o prompt passa a ser publicado independentemente do deploy do código — aí ele deixa de acompanhar o commit.

---

### 3. A justificativa técnica: raio de alcance não-local

Até aqui, um engenheiro de software pode responder *"tudo bem, mas isso é só boa prática — eu já versiono e testo tudo"*. Falta a parte que é **específica de LLM** e que muda a natureza do problema:

> **Em código, você raciocina sobre o impacto de uma mudança. Num prompt, não dá.**

Se você altera uma função, sabe quem a chama e consegue delimitar o estrago. Se você acrescenta *"seja conciso"* ao seu prompt, **pode quebrar a extração de um campo que a frase nem menciona** — porque o modelo processa tudo junto e a mudança desloca a distribuição inteira.

Não existe análise estática de prompt. Não existe "encontrar usos". Não existe tipo que garanta nada.

**Só existe medir.** É daí — e não de disciplina de processo — que vem a necessidade da suíte de regressão.

---

### 4. Tirar o prompt do código

O primeiro passo é mecânico: templates em arquivos versionados, com o texto separado das variáveis.

```python
from pathlib import Path

PROMPTS = Path(__file__).parent / "prompts"

def carregar_prompt(nome: str, **variaveis) -> str:
    texto = (PROMPTS / f"{nome}.md").read_text(encoding="utf-8")
    return texto.format(**variaveis)
```

Dez linhas resolvem 80% do problema: o prompt passa a aparecer no `git diff`, entra no review do PR e volta com `git revert`.

**O trade-off honesto** entre as duas opções que existem:

| | Arquivo no repositório | Prompt registry (LangSmith, Langfuse…) |
|---|---|---|
| Review | **entra no PR** | fora do fluxo de review |
| Rollback | `git revert` | pela ferramenta |
| Mudar sem deploy | não | **sim** |
| Quem pode mudar | quem tem acesso ao repositório | qualquer pessoa com acesso à ferramenta |

O registry vende "mudar o prompt sem deploy" como vantagem — e é, para times com pessoas não técnicas ajustando texto. Mas é a **mesma** propriedade que faz a mudança escapar do review e do teste. **Comece pelo arquivo.** Registry é decisão de organização, não de engenharia, e volta na aula de produção.

---

### 5. Teste de regressão de prompt

Aqui a Aula 02 (nota 02, §6) é reaproveitada inteira: **teste por propriedade, não por igualdade**.

```python
CASOS = [
    ("Meu pedido 48219 não chegou",        "entrega_atrasada", "48219"),
    ("vocês entregam no sábado?",          "duvida",           None),
    ("o entregador deixou no nº 45, o meu é 145", "endereco_errado", None),
]

def test_extracao():
    for mensagem, categoria_esperada, pedido_esperado in CASOS:
        saida = extrair(mensagem)
        assert saida["categoria"] == categoria_esperada
        assert saida["pedido"] == pedido_esperado
```

Repare no que **não** está sendo testado: o texto exato da resposta. Isso falharia sem que nada estivesse quebrado.

#### 5.1 O teste é flaky por natureza

Um detalhe que não existe em teste comum e que assusta quem vem de teste determinístico: **`temperature=0` não garante saída idêntica** (Aula 02, nota 02 §6). O mesmo teste pode passar hoje e falhar amanhã sem que nada tenha mudado.

Isso não é defeito da suíte — é propriedade do sistema sob teste. O critério muda de "passou" para **"passou k de N execuções"**:

```python
def taxa_de_acerto(caso, n=5):
    acertos = sum(1 for _ in range(n) if avaliar(caso))
    return acertos / n

def test_suite():
    resultados = [taxa_de_acerto(c) for c in CASOS]
    media = sum(resultados) / len(resultados)
    assert media >= 0.90, f"regressão: {media:.0%} (mínimo 90%)"
```

Duas decisões embutidas, e as duas são de projeto, não de código:

- **o limiar** (aqui 90%) tem que ser definido **antes** de ver o resultado. Senão você ajusta o limiar até passar, e o teste não testa nada;
- **N não sai de graça.** N=5 sobre 20 casos são 100 chamadas por execução da suíte, e cada uma leva alguns segundos. Rode a suíte completa antes de mergear, e uma versão reduzida durante o desenvolvimento.

#### 5.2 O que isso tem a ver com evals

É a mesma coisa, em escala e ambição diferentes:

| | Teste de regressão (aqui) | Eval (fim do curso) |
|---|---|---|
| Tamanho | 5–20 casos | 50–500 |
| Critério | propriedade objetiva | + LLM-as-judge, rubrica |
| Quando roda | antes de mergear | + contínuo, em produção |
| Pergunta | *"eu quebrei algo?"* | *"isto é bom o suficiente?"* |

Escrever a versão pequena agora não duplica a aula de evals — **prepara**. Você chega lá já tendo sentido o problema, e a aula passa a ser sobre o que a versão pequena não resolve.

---

### 6. A demonstração: uma palavra e três casos

Este é o momento em que o argumento deixa de ser teórico.

**Prompt v1:**
```text
Extraia o número do pedido. Responda apenas o número.
```
→ suíte: **8/8**

Alguém abre um PR de limpeza de texto e remove uma palavra:

**Prompt v2:**
```text
Extraia o número do pedido. Responda o número.
```
→ suíte: **5/8**

Três casos passaram a vir assim:

```
O número do pedido é 48219.
```

O `json.loads` do outro lado quebra, ou o campo vai para o banco com texto em volta. **A palavra "apenas" era o contrato inteiro** — e nenhum revisor humano teria apontado isso num diff de uma palavra.

A suíte apontou em quatro segundos.

> **A lição, e ela é o resumo do bloco:** revisar prompt lendo o texto é como revisar código sem rodar os testes. Funciona até não funcionar, e você descobre em produção.

Esta demonstração é reproduzível: [`05-versao-de-prompt.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula03-prompt/05-versao-de-prompt.py) traz as duas versões em `prompts/`, roda a suíte contra as duas e imprime o carimbo de cada uma. Comece rodando `diff` entre os dois arquivos — a diferença é a palavra *apenas*.

---

## Exemplos

### Exemplo 1 — A estrutura mínima que resolve

Não é preciso ferramenta nenhuma para ter versionamento e teste:

```
projeto/
├── prompts/
│   ├── extracao-chamado.md      ← o texto, versionado
│   └── redacao-chamado.md
├── config.py                    ← modelo + parâmetros por etapa
├── casos_de_teste.py            ← os casos com o esperado
└── test_prompts.py              ← a suíte
```

```python
# config.py — os três eixos juntos, num lugar só
EXTRACAO = {
    "prompt": "extracao-chamado",
    "modelo": "mistral-small-2506",   # versão DATADA (Aula 02, nota 01 §7.2)
    "params": {"temperature": 0, "max_tokens": 120},
}
```

Com isso: o prompt aparece no diff, a mudança de modelo aparece no diff, a mudança de parâmetro aparece no diff — e a suíte roda contra os três.

## Exercícios resolvidos

### 1. O prompt que "não mudou"

**Enunciado.** Uma extração de dados funcionava com 97% de acerto. Sem ninguém tocar no código, caiu para 71% em uma semana. O time jura que o prompt não mudou — e o `git log` do arquivo confirma. O que investigar, e em que ordem?

**Resolução.**

O `git log` do prompt confirma **um** dos três eixos. Faltam dois.

**1. O modelo mudou.** É a hipótese mais provável e a mais fácil de checar. Se o código usa um alias `-latest`, o provedor pode ter atualizado o modelo por baixo (Aula 02, nota 01 §7.2). Verificação: fixar uma versão datada anterior e rodar a suíte. Se voltar a 97%, está achado.

**2. Os parâmetros mudaram.** Alguém pode ter ajustado `temperature` ou `max_tokens` num PR não relacionado. `git log` do arquivo de configuração, não do prompt.

**3. A entrada mudou.** O menos lembrado e bem comum: o *formato* das mensagens de origem mudou (novo canal de atendimento, novo template de e-mail com assinatura longa, mensagens mais compridas estourando o `max_tokens`). O prompt está igual; o que ele recebe, não.

**O que fazer depois de achar:**

- se foi o modelo: **fixar a versão datada** e migrar deliberadamente, com a suíte rodando antes;
- se foi parâmetro: trazer para o arquivo de configuração versionado junto do prompt;
- se foi a entrada: acrescentar os novos casos à suíte — ela não pegou porque não os conhecia.

**E a lição de processo:** os 26 pontos foram perdidos ao longo de uma semana, sem ninguém notar. Uma suíte rodando no CI teria falhado no dia. **O valor da suíte não está em achar o culpado — está em reduzir o tempo entre a quebra e a descoberta.**

## Síntese

- **Prompt determina comportamento como código determina** — mas costuma viver sem versão, sem review, sem teste e sem rollback.
- **Três eixos, uma unidade:** `comportamento = prompt × modelo × parâmetros`. Versionar um sem os outros **impede reproduzir bug**. Em produção, versão de modelo **datada**, nunca alias.
- A justificativa técnica da suíte é o **raio de alcance não-local**: não há análise estática de prompt — acrescentar "seja conciso" pode quebrar um campo que a frase nem menciona. **Só dá para medir.**
- Tirar o prompt do código são **dez linhas**, e já entrega diff, review e `git revert`. Registry é decisão de organização — e a mesma propriedade que o torna conveniente (mudar sem deploy) é a que faz escapar do review.
- **Teste por propriedade, nunca por igualdade.** E o teste é **flaky por natureza**: critério **k de N**, com o limiar definido **antes** de ver o resultado.
- A suíte é o **embrião do eval**: mesma natureza, escala e ambição diferentes.
- Uma palavra removida ("apenas") derrubou a suíte de 8/8 para 5/8. **Revisar prompt lendo o texto é como revisar código sem rodar os testes.**

---

## Fontes e leituras

**Engenharia**

- Aula 02, [nota 02 §6](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/02-configuracoes-da-chamada.md) — teste por propriedade e a não-reprodutibilidade do `temperature=0`. É a base da §5.
- Aula 02, [nota 01 §7.2](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) — o alias `-latest` como dependência sem versão.
- Documentação de *prompt registry* (LangSmith, Langfuse) — para quando o time tiver pessoas não técnicas ajustando texto; volta na aula de produção.

**Nesta disciplina**

- [Nota 01 desta aula](01-anatomia-e-tecnicas.md) — o output contract e os delimitadores, que são o que a suíte desta nota testa.
- [Nota 02 desta aula](02-context-engineering.md) — o contexto também muda entre versões, e também precisa entrar na suíte.
- [Nota 04 desta aula](04-tool-calling.md) — a descrição da ferramenta é prompt, logo entra no mesmo versionamento e na mesma suíte.
- **Aula de segurança** — prompt injection, direto e indireto, e as defesas de arquitetura. O fato de que o modelo não distingue instrução de dado fica plantado aqui; as consequências são lá.
