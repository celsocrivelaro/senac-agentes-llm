# IA Aplicada com LLMs — Aula 07: RAG e Documentos — A resposta que cita e que recusa

## Introdução

A [nota anterior](04-o-pipeline-e-os-quatro-modos-de-falha.md) deixou dois problemas em aberto. O primeiro é que uma resposta sem fonte declarada não é auditável: o modo de falha 1 produz respostas confiantes sobre o assunto errado, e nada no formato de saída permite detectá-lo. O segundo é o modo de falha 3, em que o modelo responde a uma pergunta cuja resposta não está no corpus.

Os dois se fecham com o mesmo instrumento: um **contrato de saída**. E o instrumento não é novo — é a saída estruturada da Aula 02 (nota 02, §7) combinada com o contrato de saída da Aula 03 (nota 01), aplicados agora ao caminho da resposta.

A segunda metade da nota trata da decisão oposta: **quando não recuperar**. É a tática zero da Aula 05 (nota 03, §4) num contexto novo, e leva a uma conclusão incômoda — para uma classe inteira de perguntas, a ferramenta determinística da Aula 05 supera o sistema construído nesta aula.

> **Pré-requisitos:** [nota 04](04-o-pipeline-e-os-quatro-modos-de-falha.md) desta aula · Aula 02, [nota 02 §7](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/02-configuracoes-da-chamada.md) · Aula 03, [nota 01](../../aula-03-prompt-engineering/notas-de-aula/01-anatomia-e-tecnicas.md) · Aula 05, [nota 01-2](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-2-router.md) e [nota 03 §8](../../aula-05-arquitetura-de-agentes/notas-de-aula/03-context-engineering-dinamica.md).
>
> **Código:** [`06-citacao-e-recusa.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula07-rag-e-documentos/06-citacao-e-recusa.py).

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Especificar** o contrato de saída de uma resposta com fonte citada e recusa explícita.
- **Verificar a citação em código**, sem envolver o modelo na verificação.
- **Justificar a obrigatoriedade da recusa** por analogia com a rota de escape de um roteador.
- **Determinar quando a recuperação não se aplica**, e defender a decisão pelo custo.

---

## Desenvolvimento teórico

### 1. O contrato

```python
SCHEMA_RESPOSTA = {
    "type": "object",
    "properties": {
        "resposta": {"type": "string"},
        "fontes": {"type": "array", "items": {"type": "string"}},
        "suficiente": {"type": "boolean"},
    },
    "required": ["resposta", "fontes", "suficiente"],
    "additionalProperties": False,
}
```

Três campos, e cada um fecha uma lacuna identificada na nota anterior:

| Campo | Fecha |
|---|---|
| `resposta` | nada — é o que já existia |
| `fontes` | modo de falha 1: torna a recuperação incorreta **inspecionável** |
| `suficiente` | modo de falha 3: torna a ausência de resposta **declarável** |

O schema é imposto por decodificação restrita, como na Aula 02 (nota 02, §7). O modelo não tem como omitir um campo obrigatório nem devolver `suficiente` fora do domínio booleano.

#### 1.1 O prompt que acompanha o schema

Schema restringe a **forma**; o prompt determina o **conteúdo**. O laboratório usa:

```python
PROMPT_COM_CONTRATO = """Responda a pergunta do colaborador usando EXCLUSIVAMENTE
os trechos da política de reembolso abaixo.
...
Responda no schema, observando:
- `fontes`: os rótulos entre colchetes dos trechos efetivamente usados na
  resposta, exatamente como aparecem. Não invente rótulo.
- `suficiente`: false quando os trechos não contêm a informação necessária
  para responder. Nesse caso `resposta` diz o que falta, e não tenta
  responder assim mesmo.

Não use conhecimento que não esteja nos trechos. Não conclua além do que
os trechos permitem."""
```

Três elementos correspondem diretamente ao que a Aula 03 estabeleceu: contrato explícito por campo, **exemplos negativos** (*"não invente rótulo"*, *"não tenta responder assim mesmo"*) e delimitação da fonte de conhecimento.

O último parágrafo merece destaque. Sem ele, o modelo complementa os trechos com conhecimento próprio — e o faz de maneira plausível, porque política de reembolso é um domínio sobre o qual ele viu muitos exemplos durante o treinamento. A resposta resultante mistura o corpus da empresa com a prática de mercado, sem que nada no formato indique onde uma termina e a outra começa.

---

### 2. A recusa é a rota `nenhuma`

O campo `suficiente` não é uma cortesia. É o mesmo mecanismo que a Aula 05 (nota 02, §2) estabeleceu como obrigatório em um roteador:

> Sem rota de escape, o modelo é forçado a escolher entre opções que não se aplicam — e escolhe, com alta confiança declarada.

A correspondência é exata:

| Roteador (Aula 05) | RAG (esta aula) |
|---|---|
| enumeração de rotas | trechos recuperados |
| rota `nenhuma` | `suficiente: false` |
| fila de revisão | destino da recusa |
| erro silencioso evitado | resposta inventada evitada |

E vale a mesma advertência das duas aulas: **recusar sem destino é falhar de maneira mais educada**. Um sistema que devolve `suficiente: false` e encerra transferiu o problema para o usuário. A recusa precisa levar a algum lugar — fila de revisão humana, solicitação de reformulação, ou nova consulta com outra estratégia, que é o assunto da [nota 03](03-medir-o-rag-e-a-recuperacao-como-ferramenta.md).

---

### 3. A citação precisa ser verificável

Uma fonte só tem valor se puder ser conferida. A verificação é determinística e não envolve o modelo:

```python
def citacao_verificavel(fontes: list[str], trechos: list[dict]) -> dict:
    disponiveis = [t["artigo"] for t in trechos]
    validas = [f for f in fontes if _contem(disponiveis, f)]
    return {"citadas": fontes, "disponiveis": disponiveis,
            "validas": validas,
            "inventadas": [f for f in fontes if f not in validas]}
```

A função compara o que foi citado com o que foi recuperado. Custa zero chamadas, roda em toda resposta e é executável em produção.

Sua viabilidade depende de uma decisão tomada duas aulas antes: o corte por **estrutura** da Aula 06 produz identificadores que correspondem à numeração do documento (`Art. 4º §2º`). Sob corte por contagem de caracteres, os identificadores seriam `c7` e `s12`, e não haveria o que citar — a resposta teria de referenciar o texto por paráfrase, e paráfrase não é conferível por comparação de cadeias.

> **Uma fonte inventada é pior que nenhuma fonte.** Ela transmite ao leitor a impressão de que a resposta foi verificada. A checagem acima existe justamente para que essa impressão seja verdadeira.

---

### 4. Quando não recuperar

Esta seção trata da condição em que a recuperação **não** deve ocorrer, e do custo que a recuperação incondicional impõe.

O sistema construído até aqui tem custo por pergunta: uma chamada de embedding, uma consulta ao índice e um prompt maior — porque os trechos recuperados entram no contexto. Esse custo se paga em **toda** pergunta, e não em algumas.

Há três situações em que ele não se justifica:

#### 4.1 A resposta é uma regra determinística

*"Qual o teto da categoria refeição?"* é uma consulta a uma tabela. A Aula 05 a resolvia assim:

```python
POLITICA = {
    "refeicao": {"artigo": "Art. 4", "teto_por_pessoa": 120.00,
                 "exige_nota": True},
    ...
}
```

Duas linhas, zero chamadas, resultado sempre correto. Comparada ao RAG desta aula, a estrutura acima é superior em custo, latência e confiabilidade — e inferior apenas em abrangência: ela responde às perguntas previstas por quem a escreveu.

O critério de escolha decorre disso. Quando o conjunto de perguntas é fechado e as respostas cabem numa tabela, a tabela vence. Quando o conjunto é aberto e o conhecimento é um documento que ninguém vai estruturar — que é a premissa da Aula 06 —, a recuperação vence.

#### 4.2 O corpus cabe no prompt

Quarenta artigos curtos somam alguns milhares de tokens. Um modelo com janela ampla os recebe integralmente, e a resposta obtida tem **fidelidade maior**, porque nenhuma etapa de recuperação pode omitir o trecho relevante.

O cálculo é direto: se o corpus íntegro couber no orçamento por pergunta com folga, o índice é infraestrutura sem retorno. O ponto em que a conta se inverte é o volume, e ele se calcula, não se estima.

#### 4.3 A pergunta contém identificador ou valor

É a conclusão da [nota 02](02-o-que-o-embedding-nao-ve.md) desta aula (§6) aplicada ao roteamento. Uma pergunta que contenha `D-4471` ou uma faixa de valor não é caso de busca semântica: o vetor é cego para as duas coisas.

O padrão adequado é o roteador da Aula 05, decidindo antes de qualquer chamada qual mecanismo se aplica:

```
   pergunta ──> [ triagem ] ──┬──> contém id            ──> consulta direta
                              ├──> pede valor de tabela ──> lookup
                              └──> é sobre assunto      ──> RAG
```

> É a **tática zero** da Aula 05 (nota 03, §4) num contexto novo: antes de otimizar o mecanismo caro, verificar se ele precisa ser acionado.

---

## Exemplos

### Exemplo 1 — A mesma pergunta, sem e com o campo `suficiente`

```
  P: Em quantos dias o reembolso é depositado na conta do colaborador?

     SEM contrato: O reembolso é depositado em até 30 dias corridos,
                   conforme o prazo estabelecido na política.

     COM contrato: [RECUSOU] Os trechos disponíveis tratam do prazo de
                   submissão do pedido (Art. 8º §1º), não do prazo de
                   depósito. A política fornecida não trata desse ponto.
                   fontes=[]
```

*(Saída ilustrativa; a execução do `06-citacao-e-recusa.py` produz a do modelo em uso.)*

A diferença entre as duas execuções é um campo de schema e um parágrafo de prompt. A primeira resposta é o modo de falha mais caro da nota anterior; a segunda é o comportamento correto — e note-se que ela **não se limita a recusar**: informa o que os trechos de fato contêm, o que permite ao usuário reformular.

### Exemplo 2 — Uma fonte inventada, detectada em código

```
  [FONTE INVENTADA] Qual o teto para hospedagem em viagem internacional?
      citou=['Art. 6º §2º', 'Art. 2º §3º']
      disponíveis=['Art. 6º §2º', 'Art. 6º §1º', 'Art. 6º §3º']
      inventadas=['Art. 2º §3º']
```

O modelo citou o Art. 2º §3º, que define viagem internacional e é pertinente ao raciocínio — mas **não estava entre os trechos recuperados**. A citação procede do conhecimento prévio do modelo sobre o documento, não do contexto fornecido.

O caso é instrutivo porque a citação inventada está **correta**: o Art. 2º §3º existe e define o que a resposta afirma. A verificação em código a rejeita mesmo assim, e deve rejeitar: o requisito é que a resposta se apoie no que foi recuperado, e não que acerte por outros meios. Um sistema que aceita citações não verificadas aceitará também as que estiverem erradas.

### Exemplo 3 — O contraste que fecha a aula

Pergunta: *"Qual o teto da categoria refeição?"*

| | RAG desta aula | Ferramenta da Aula 05 |
|---|---|---|
| Chamadas | 1 embedding + 1 geração | 0 |
| Latência | ~1 s | microssegundos |
| Custo | ~R$ 0,002 | 0 |
| Pode errar? | sim — modos de falha 1 e 4 | não |
| Cobre pergunta imprevista? | sim | não |

Para esta pergunta, a tabela vence em quatro dos cinco critérios. Reconhecê-lo não invalida a aula: estabelece a fronteira. O RAG existe para a quinta linha, e apenas para ela.

O erro que a comparação previne é o de substituir a tabela pelo RAG por parecer mais moderno. É o mesmo erro que a Aula 05 combateu ao mostrar que a rota mais valiosa de um roteador é a que não chama o modelo — aqui, a consulta mais valiosa é a que não recupera nada.

---

## Fontes e leituras

BARNETT, S. et al. **Seven failure points when engineering a retrieval augmented generation system**. arXiv:2401.05856, 2024.

ES, S. et al. **RAGAS**: automated evaluation of retrieval augmented generation. arXiv:2309.15217, 2023.

LEWIS, P. et al. **Retrieval-augmented generation for knowledge-intensive NLP tasks**. arXiv:2005.11401, 2020.

**Material da disciplina.** Aula 02, [nota 02 §7](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/02-configuracoes-da-chamada.md) — saída estruturada com decodificação restrita. Aula 03, [nota 01](../../aula-03-prompt-engineering/notas-de-aula/01-anatomia-e-tecnicas.md) — contrato de saída e exemplos negativos. Aula 05, [nota 01-2](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-2-router.md) — a rota de escape obrigatória; [nota 03 §8](../../aula-05-arquitetura-de-agentes/notas-de-aula/03-context-engineering-dinamica.md) — a tática zero. Desta aula, [nota 02 §6](02-o-que-o-embedding-nao-ve.md) — o que não é caso de busca semântica.
