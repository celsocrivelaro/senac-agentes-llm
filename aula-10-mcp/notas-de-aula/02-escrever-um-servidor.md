# IA Aplicada com LLMs — Aula 10: MCP — Escrever um servidor

## Introdução

A nota anterior estabeleceu que MCP é transporte de ferramenta, e que consumir um servidor de terceiro altera uma linha do agente. Esta nota trata do lado oposto: **publicar** uma ferramenta que já existe, de modo que qualquer agente a consuma — inclusive agentes que o autor não escreveu e não pode inspecionar.

A operação é mecanicamente pequena. A ferramenta `consultar_politica` da Aula 05 vira servidor MCP em cerca de vinte linhas, e o corpo da função não muda. O que muda é o **estatuto do contrato**: um schema que antes era detalhe interno de um repositório passa a ser interface pública, e alterá-lo passa a quebrar consumidores desconhecidos.

Essa mudança de estatuto leva ao limite uma frase da Aula 03 que até aqui parecia figura de linguagem: *a descrição da ferramenta é prompt*. Num servidor MCP, ela é prompt **entregue a terceiros**.

> **Pré-requisitos:** Aula 03, [nota 01](../../aula-03-prompt-engineering/notas-de-aula/01-anatomia-e-tecnicas.md) e [nota 03](../../aula-03-prompt-engineering/notas-de-aula/03-prompt-como-codigo.md) · Aula 05, [nota 02](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md) — erro recuperável × fatal, e a mensagem de erro como prompt · nota 01 desta aula.
>
> **Código:** [`aula10-mcp/`](https://github.com/celsocrivelaro/senac-llm-code/tree/main/aula10-mcp) — o script `00-servidor.py` corresponde a esta nota.

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Implementar** um servidor MCP que exponha uma ferramenta de escrita e um recurso de leitura.
- **Explicar** por que a descrição de uma ferramenta publicada é um artefato de prompt com consumidores externos.
- **Redigir** uma descrição de ferramenta que reduza chamada incorreta, e justificar cada elemento dela.
- **Classificar** um erro de servidor entre recuperável e fatal, e formatar a mensagem correspondente.
- **Definir** uma política de versionamento para um contrato de ferramenta com consumidores desconhecidos.

---

## Desenvolvimento teórico

### 1. A ferramenta vira servidor

A camada de alto nível do SDK de Python — conhecida como **FastMCP** — deriva a declaração a partir da própria assinatura da função. A *docstring* vira a descrição; as anotações de tipo viram o `inputSchema`; o valor de retorno é serializado.

```python
from mcp.server.fastmcp import FastMCP

servidor = FastMCP("politica-de-despesas")

@servidor.tool()
def consultar_politica(categoria: str) -> dict:
    """Devolve o teto de reembolso vigente para uma categoria de despesa."""
    return POLITICA[categoria]
```

O corpo é o mesmo da Aula 05. Isso é intencional e vale ser dito em voz alta: **MCP não pede reescrita de lógica**. Ele pede que a fronteira seja declarada.

A consequência é que a qualidade da declaração passa a ser trabalho de primeira classe. Enquanto a ferramenta era função interna, uma *docstring* fraca custava a legibilidade do repositório. Publicada, ela custa chamadas incorretas de agentes que ninguém do time controla.

### 2. A descrição é prompt — e agora é prompt de terceiros

A Aula 03 (nota 04) estabeleceu que a descrição de uma ferramenta é o texto pelo qual o modelo decide **se** e **como** chamá-la, e que portanto ela é prompt sujeito às mesmas regras: contrato explícito, ausência de ambiguidade, exemplo quando o formato não é óbvio.

Num servidor MCP, três propriedades novas se somam:

1. **O consumidor é desconhecido.** O texto será lido por modelos diferentes, com prompts de sistema diferentes, dentro de agentes com objetivos diferentes. Suposições implícitas sobre o contexto de uso deixam de valer.
2. **O contrato é público.** Mudar o nome de um parâmetro é mudança quebrante para todo cliente, e nenhum compilador avisa.
3. **A mudança é silenciosa.** Um cliente que passa a chamar errado não lança exceção: ele produz resultado pior. A falha aparece como degradação de qualidade, não como erro.

Disso decorre a regra de escrita para descrição publicada: **diga o que a ferramenta faz, o formato exato dos argumentos, e o que ela não faz**. A última parte é a mais esquecida e a que mais evita chamada indevida.

```python
@servidor.tool()
def consultar_politica(categoria: str) -> dict:
    """Devolve o teto de reembolso vigente para uma categoria de despesa.

    categoria: um de "refeicao", "transporte", "hospedagem", "outros".
               Valores fora dessa lista devolvem erro com a lista válida.

    NÃO decide se uma despesa específica é aprovada, e NÃO considera
    exceções de viagem internacional — para isso, consulte o regulamento
    pelo recurso regulamento://despesas/{artigo}.
    """
```

### 3. Recurso e ferramenta, na prática

A nota anterior estabeleceu o critério: ferramenta é o que o **modelo** decide invocar; recurso é o que a **aplicação** decide incluir. O servidor da aula aplica os dois ao mesmo domínio.

```python
@servidor.resource("regulamento://despesas/{artigo}")
def artigo(artigo: str) -> str:
    """Texto integral de um artigo do regulamento de despesas."""
    return REGULAMENTO[artigo]

@servidor.tool()
def registrar_parecer(despesa: str, veredito: str, justificativa: str,
                      chave: str) -> dict:
    """Registra o parecer de uma despesa. Idempotente pela chave.

    chave: identificador estável derivado de (despesa, analista, dia).
           Chamadas repetidas com a mesma chave devolvem o parecer
           existente com ja_existia=true, sem criar um segundo registro.
    """
    if (existente := PARECERES.get(chave)):
        return {**existente, "ja_existia": True}
    ...
```

Duas observações sobre esse par.

A primeira é que o **regulamento é recurso** porque a aplicação sabe, antes do laço começar, que o agente precisará dele. Deixá-lo como ferramenta transferiria ao modelo a decisão de ler — e o modelo pode não ler, ou ler quatro vezes.

A segunda é que a **idempotência atravessou a fronteira**. Na Aula 05 ela protegia contra retry, retomada de checkpoint e laço detectado. Num servidor MCP ela protege contra algo novo: **clientes que o autor do servidor não controla**. Um agente mal escrito do outro lado da conexão vai repetir chamadas, e a chave de idempotência é o que impede que essa repetição vire dano.

### 4. O erro, atravessando o protocolo

A Aula 05 (nota 02, §4) estabeleceu que a mensagem de erro é o único texto que o modelo lê para decidir **como se corrigir**, e que ela deve responder três coisas: o que errou, qual era o certo, e o que fazer agora. A classificação entre recuperável e fatal cabe ao **código**, nunca ao modelo.

Num servidor MCP, essa classificação passa a ter duas naturezas distintas, e confundi-las é o defeito mais comum:

| Natureza | Exemplo | Quem trata |
|---|---|---|
| **Erro de protocolo** | método inexistente, JSON malformado | o cliente, como falha de transporte |
| **Erro de domínio** | categoria inválida, id inexistente | **o modelo**, como observação |

Um erro de domínio devolvido como falha de protocolo desaparece do contexto do modelo: ele nunca fica sabendo que errou o argumento, e repete. Um erro de infraestrutura devolvido como observação produz o oposto — o modelo reformula educadamente até o orçamento acabar, exatamente o cenário que a Aula 05 descreveu.

```python
@servidor.tool()
def consultar_politica(categoria: str) -> dict:
    if categoria not in POLITICA:
        # Erro de domínio: volta como CONTEÚDO, e ensina.
        return {"erro": "categoria desconhecida",
                "recebido": categoria,
                "validos": sorted(POLITICA),
                "sugestao": "chame novamente com um dos valores de 'validos'"}
    return POLITICA[categoria]
```

### 5. Versionamento de um contrato público

A Aula 03 (nota 03) estabeleceu o carimbo: uma execução só é comparável a outra se prompt, modelo e parâmetros forem os mesmos. A Aula 06 acrescentou a estratégia de *chunking*; a Aula 08, o estado da memória. Esta aula acrescenta **a revisão da especificação e a versão do servidor**, e a razão é a mesma de sempre: são entradas que mudam o comportamento observado.

Mas há aqui uma diferença de grau que vale nomear. Nos casos anteriores, quem muda o carimbo é quem roda o experimento. Num servidor MCP de terceiro, **quem muda o carimbo é outra pessoa**, sem aviso e sem que o resultado deixe de ser sintaticamente válido.

A política mínima que decorre disso:

- **Fixar a versão** do servidor consumido, como se fixa a de qualquer dependência.
- **Tratar a descrição como código sob revisão**: mudança de descrição é mudança de comportamento e passa pela suíte da Aula 11.
- **Não remover nem renomear** parâmetro de ferramenta publicada; acrescentar opcional, e depreciar com prazo.

O protocolo passou a ajudar nisso na revisão de 2026-07-28, que introduziu política formal de depreciação. Antes dela, cada servidor decidia sozinho — e a Aula 14 mostra a classe de ataque que essa ausência de garantia habilitou.

---

## Exemplos

### Exemplo 1 — O servidor mínimo, completo

```python
from mcp.server.fastmcp import FastMCP
from dados import POLITICA

REGULAMENTO = {"art-7": "Art. 7º O reembolso de despesa com refeição observará o teto de R$ 120,00 por refeição, mediante nota fiscal.", "art-12": "...", "art-19": "..."}

servidor = FastMCP("politica-de-despesas")

@servidor.resource("regulamento://despesas/{artigo}")
def artigo(artigo: str) -> str:
    """Texto integral de um artigo do regulamento de despesas."""
    return REGULAMENTO.get(artigo, f"[artigo {artigo} inexistente]")

@servidor.tool()
def consultar_politica(categoria: str) -> dict:
    """Devolve o teto de reembolso vigente para uma categoria de despesa."""
    if categoria not in POLITICA:
        return {"erro": "categoria desconhecida", "recebido": categoria,
                "validos": sorted(POLITICA)}
    return {"categoria": categoria, **POLITICA[categoria]}

if __name__ == "__main__":
    servidor.run(transport="streamable-http")   # processo próprio, em porta
```

Vinte e duas linhas, e a ferramenta da Aula 05 passou a ser consumível por qualquer agente.

### Exemplo 2 — Duas descrições, e a diferença medida

```
DESCRIÇÃO A: "Consulta a política."
  -> o modelo chama com categoria="almoço"      (não existe)
  -> o modelo chama sem argumento                (schema rejeita)
  -> 3 chamadas até acertar

DESCRIÇÃO B: "Devolve o teto de reembolso vigente para uma categoria de
              despesa. categoria: um de "refeicao", "transporte",
              "hospedagem", "outros"."
  -> o modelo chama com categoria="refeicao"
  -> 1 chamada
```

O ganho não é de elegância. São duas chamadas a menos por execução, em todo cliente do servidor, para sempre.

### Exemplo 3 — A idempotência protegendo contra o cliente alheio

```
cliente A (bem escrito)   registrar_parecer(D-4612, ..., chave=k1) -> P-1188
cliente B (com retry)     registrar_parecer(D-4612, ..., chave=k1) -> P-1188
                                                    {"ja_existia": true}
```

O segundo cliente não é do time. A chave é o que impede que o `retry` dele produza dois pareceres para a mesma despesa — e o `ja_existia` é informação **para o modelo** do cliente B, que assim descobre que a ação já estava feita.

---

## Fontes e leituras

MODEL CONTEXT PROTOCOL. **Specification**: revisões 2025-06-18 (saída estruturada) e 2026-07-28 (política de depreciação).

MODEL CONTEXT PROTOCOL. **Python SDK**: documentação da camada FastMCP.

**Material da disciplina.** Aula 03, [nota 04](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) — a descrição da ferramenta como prompt; [nota 03](../../aula-03-prompt-engineering/notas-de-aula/03-prompt-como-codigo.md) — versionamento e carimbo. Aula 05, [nota 02](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md) — erro recuperável × fatal, mensagem acionável e idempotência.
