# IA Aplicada com LLMs — Aula 10: MCP — O que MCP padroniza

## Introdução

A Aula 03 (nota 04, §9) encerrou com uma lista de pendências. Depois da Aula 05, três permaneciam de pé; a Aula 08 riscou a primeira. Esta nota abre o penúltimo item.

O problema que a motiva é concreto e vem da Aula 05. O agente construído lá tem ferramentas que são funções Python: cada uma exigiu um schema JSON escrito à mão, validação de argumentos, classificação de erro entre recuperável e fatal, e uma política de versionamento. Esse custo é aceitável quando a ferramenta é do próprio domínio. Deixa de ser quando a ferramenta é **um sistema de terceiro** — um repositório de código, um banco, um serviço de tíquetes —, porque nesse caso o mesmo cliente está sendo reescrito, com pequenas variações, por todo grupo que constrói um agente.

O **Model Context Protocol (MCP)** é a resposta padronizada a esse desperdício. Esta nota estabelece o que ele padroniza, o que ele deliberadamente não padroniza, e o que muda — quase nada — no agente que já existe.

> **Pré-requisitos:** Aula 03, [nota 01](../../aula-03-prompt-engineering/notas-de-aula/01-anatomia-e-tecnicas.md) e [nota 04](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) · Aula 05, [nota 01](../../aula-08-memoria/notas-de-aula/01-o-agente-e-o-estado.md) e [nota 02](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md).
>
> **Código:** [`aula10-mcp/`](https://github.com/celsocrivelaro/senac-llm-code/tree/main/aula10-mcp) — os scripts `01-sessao.py`, `02-agente-com-mcp.py` e `03-agente-langgraph-mcp.py` correspondem a esta nota — os três clientes do mesmo servidor.

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Formular** o problema que MCP resolve em termos de M×N, e reconhecer o precedente do Language Server Protocol.
- **Distinguir** os três primitivos do protocolo — *tool*, *resource* e *prompt* — pelo agente que controla cada um.
- **Descrever** o ciclo de uma sessão MCP: negociação de capacidades, listagem e invocação.
- **Escolher** entre os dois transportes definidos pela especificação, justificando pela topologia do sistema.
- **Demonstrar** que o laço, o estado e as salvaguardas da Aula 05 não são alterados pela adoção do protocolo.

---

## Desenvolvimento teórico

### 1. O problema em M×N

Considere M aplicações que precisam falar com N sistemas externos. Sem um protocolo comum, cada par exige uma integração própria: o número de integrações a escrever e manter cresce como **M × N**. Com um protocolo comum, cada aplicação implementa um cliente e cada sistema implementa um servidor: o número cai para **M + N**.

Esse argumento não é novo, e o precedente mais próximo é conhecido de quem estuda Ciência da Computação. Antes do **Language Server Protocol (LSP)**, cada editor de código precisava de uma integração própria para cada linguagem — autocompletar de Python no editor A, autocompletar de Python no editor B, e assim por diante. O LSP definiu um protocolo entre editor e servidor de linguagem, e a matriz virou soma.

MCP faz a mesma operação sobre um par diferente: **agente** e **sistema de contexto ou ação**. O agente implementa um cliente MCP uma vez; cada sistema expõe um servidor MCP uma vez.

A economia é real, mas convém ser preciso sobre onde ela está. O protocolo não escreve a integração: alguém ainda precisa implementar o servidor que fala com o sistema de tíquetes. O que ele elimina é a **multiplicação** — esse servidor é escrito uma vez e consumido por qualquer agente, inclusive por agentes que o autor do servidor não conhece.

### 2. Os três primitivos, e quem controla cada um

O protocolo define três tipos de coisa que um servidor pode expor. A distinção entre eles não é de conteúdo — é de **quem decide usá-los**, e essa é a chave para entender por que são três e não um.

| Primitivo | O que é | Quem decide invocar |
|---|---|---|
| **Tool** | operação executável, com efeito ou retorno | **o modelo**, durante o laço |
| **Resource** | dado legível, identificado por URI | **a aplicação**, ao montar o contexto |
| **Prompt** | modelo de interação parametrizado | **o usuário**, explicitamente |

Apenas o primeiro é familiar: é exatamente o *tool calling* da Aula 03, com a declaração vindo de outro processo em vez de um dicionário local.

O **resource** é o primitivo novo mais importante, e o mais frequentemente mal usado. Um recurso é conteúdo que a aplicação escolhe incluir no contexto — um arquivo, um registro, uma seção de um documento. A diferença em relação a uma ferramenta de leitura não é técnica: uma ferramenta `ler_regulamento()` faria o mesmo trabalho. A diferença é **de controle**. Quando o regulamento é ferramenta, o modelo decide se e quando lê, e pode decidir errado ou repetidamente. Quando é recurso, a aplicação decide — e essa decisão é código, testável e barato, no espírito da tabela da Aula 05 (nota 02, §3).

O **prompt** é um modelo de interação que o servidor oferece e o usuário aciona: "revisar este arquivo", "resumir esta *issue*". Ele não entra no laço automático. É o primitivo menos usado dos três, e a razão é que ele pressupõe uma interface em que o usuário escolhe algo de uma lista — o que nem todo agente tem.

> **A regra prática que decorre da tabela:** se a decisão de usar deve ser do modelo, é ferramenta; se deve ser do programa, é recurso. Colocar como ferramenta algo que a aplicação sempre precisa incluir transfere ao modelo uma decisão que não deveria ser dele, e a Aula 05 já estabeleceu o custo disso.

### 3. A sessão MCP

MCP é construído sobre **JSON-RPC 2.0**: mensagens com `method`, `params` e `id`, e respostas correlacionadas pelo `id`. Cliente e servidor mantêm uma **sessão MCP**, e ela tem um começo obrigatório.

```
cliente                                    servidor
   │── initialize ──────────────────────────▶│   versão do protocolo,
   │◀───────────────── capacidades ──────────│   capacidades de cada lado
   │── notifications/initialized ───────────▶│
   │                                          │
   │── tools/list ──────────────────────────▶│
   │◀──────────── [ {name, description,       │
   │                 inputSchema}, ... ] ─────│
   │                                          │
   │── tools/call {name, arguments} ────────▶│
   │◀──────────────────── content ───────────│
```

A negociação inicial existe porque as duas pontas evoluem separadamente. Cliente e servidor declaram a versão do protocolo que falam e as capacidades opcionais que oferecem; o que não for declarado não pode ser usado. É a mesma disciplina de contrato explícito que a Aula 03 estabeleceu para saída estruturada, aplicada à camada de transporte.

O que volta de `tools/list` é a **declaração** — nome, descrição e schema de entrada — no mesmo formato que a Aula 03 usou. É por isso que plugar MCP num agente existente é uma operação pequena: a forma da declaração não mudou, apenas a origem.

### 4. Os transportes

A especificação define dois, e a escolha decorre de onde o servidor roda.

**`stdio`** — o cliente inicia o servidor como processo filho e conversa por entrada e saída padrão. Sem rede, sem porta, sem autenticação: o isolamento é o do sistema operacional. É o transporte adequado quando o servidor pertence ao mesmo programa que o consome — e a dependência é literal, porque o servidor morre com o cliente.

**Streamable HTTP** — `POST` para o sentido cliente → servidor, e *Server-Sent Events* opcional para o sentido inverso, quando o servidor precisa enviar mensagens não solicitadas. Serve a servidores remotos, atravessa infraestrutura HTTP existente, e traz junto tudo o que rede traz: autenticação, autorização e uma superfície exposta. A revisão de março de 2025 acrescentou autorização baseada em OAuth 2.1 exatamente por isso.

**É o transporte do laboratório desta aula**, e a escolha é pedagógica: com ele o servidor vira processo independente, iniciado num terminal próprio, e os clientes apenas se conectam. A premissa do protocolo — *a ferramenta é de outro dono* — deixa de ser afirmação e passa a ser a forma de executar o laboratório. O endereço é `127.0.0.1`, de modo que a porta existe mas só a máquina local alcança.

> A escolha do transporte é decisão de topologia, não de gosto. Servidor local que lê arquivos do disco do usuário não deve escutar em endereço que a rede alcance — e a Aula 14 mostra o que acontece com quem troca `127.0.0.1` por `0.0.0.0` sem autenticação.

### 5. A especificação é móvel

Entre novembro de 2024 e julho de 2026 houve cinco revisões:

| Revisão | O que introduziu |
|---|---|
| 2024-11 | modelo cliente-servidor; *tools*, *resources*, *prompts* |
| 2025-03-26 | Streamable HTTP; autorização OAuth 2.1 |
| 2025-06-18 | saída estruturada; *elicitation*; remoção do lote JSON-RPC |
| 2025-11-25 | descoberta de autorização refinada; *tasks* experimentais |
| 2026-07-28 | núcleo sem estado; MCP Apps; política formal de depreciação |

O conteúdo dessa tabela importa menos que a leitura dela. **Um sistema construído sobre MCP está construído sobre algo que muda**, e a mesma frase valeu na Aula 02 para a depreciação de modelos e na Aula 09 para a API de um framework. A consequência prática é de engenharia comum: fixar a versão, ler a nota de migração, e tratar a atualização como trabalho previsto e não como acidente.

A revisão de 2026-07-28 merece nota, porque muda uma premissa: o núcleo passou a ser **sem estado**, e a sessão MCP longa deixou de ser obrigatória. Servidores escritos supondo sessão MCP persistente precisam ser revistos.

### 6. Consumir, e o que não muda

O experimento decisivo desta nota é plugar um servidor MCP no agente da Aula 05 e observar o que precisou mudar.

A resposta é: a origem da lista de ferramentas e o despacho da execução, e nada mais. `EstadoAgente` continua o mesmo; `Orcamento` com os quatro tetos continua o mesmo; `Termino` com os quatro motivos continua o mesmo; o detector de laço continua o mesmo. Uma ferramenta local e uma ferramenta MCP convivem na mesma lista e são indistinguíveis para o modelo, porque ambas chegam a ele como declaração no mesmo formato.

**Há uma terceira mudança, e ela não é do protocolo.** O laboratório consome o servidor pelo cliente do SDK oficial, que é assíncrono — e concorrência atravessa quem chama: o laço precisou virar `async`. A distinção importa porque é a diferença entre o que o protocolo exige e o que a *dependência* exige. O `01-sessao.py`, que fala o mesmo protocolo à mão, é síncrono.

**E há um terceiro cliente, que é o que o aluno vai usar no trabalho.** O `03-agente-langgraph-mcp.py` consome o mesmo servidor pelo adaptador do framework: a conexão, a sessão, a listagem e a tradução do schema MCP para o formato do modelo somem, e sobram um dicionário de configuração e duas chamadas. Os três degraus — à mão, SDK, adaptador — valem como medida do que cada dependência esconde.

O que o adaptador **não** faz é inspecionar o que o servidor declara. A descrição envenenada do §5 chega ao modelo exatamente como chegaria sem ele: o framework confia no servidor e repassa. Ele move a fronteira de confiança de lugar; não a fecha.

Disso decorre a afirmação que organiza a aula:

> **MCP não é uma arquitetura de agente. É transporte de ferramenta.**

O protocolo não fornece orçamento, não detecta laço, não classifica erro entre recuperável e fatal, não decide quando envolver um humano, e não grava *trace*. Tudo o que torna um agente confiável continua sendo responsabilidade de quem escreve o agente. Quem adota MCP esperando confiabilidade adotou a coisa errada pelo motivo errado.

---

## Exemplos

### Exemplo 1 — Uma sessão MCP inteira, sem biblioteca

```python
# 01-sessao.py (trecho): as três mensagens que constituem uma sessão MCP mínima
enviar({"jsonrpc": "2.0", "id": 1, "method": "initialize",
        "params": {"protocolVersion": PROTOCOLO,
                   "capabilities": {},
                   "clientInfo": {"name": "aula10", "version": "1.0"}}})
notificar({"jsonrpc": "2.0", "method": "notifications/initialized"})

enviar({"jsonrpc": "2.0", "id": 2, "method": "tools/list"})
# -> {"tools": [{"name": "consultar_politica",
#                "description": "Devolve o teto de reembolso da categoria.",
#                "inputSchema": {"type": "object",
#                                "properties": {"categoria": {"type": "string"}},
#                                "required": ["categoria"]}}]}
```

A `description` e o `inputSchema` são exatamente o que a Aula 03 escrevia à mão. O protocolo não inventou o formato da declaração — ele padronizou de onde ela vem.

### Exemplo 2 — A troca de uma linha no agente

```python
# ANTES (Aula 05): a declaração é dado local
FERRAMENTAS = {"consultar_politica": consultar_politica, ...}
declaracoes = [schema_de(f) for f in FERRAMENTAS.values()]

# DEPOIS (Aula 10): a declaração vem da sessão MCP
declaracoes = cliente.listar_ferramentas()      # tools/list
resultado   = cliente.chamar(nome, argumentos)  # tools/call

# O laço abaixo não muda nenhuma linha.
while not (motivo := orcamento.excedido(estado)):
    ...
```

### Exemplo 3 — Ferramenta ou recurso

```python
# Recurso: a APLICAÇÃO decide incluir o regulamento no contexto.
@servidor.resource("regulamento://despesas/{artigo}")
def artigo(artigo: str) -> str:
    """Texto do artigo do regulamento de despesas."""
    return REGULAMENTO[artigo]

# Ferramenta: o MODELO decide registrar o parecer, e há efeito no mundo.
@servidor.tool()
def registrar_parecer(despesa: str, veredito: str, chave: str) -> dict:
    """Registra o parecer de uma despesa. Idempotente pela chave."""
    ...
```

A separação não é estética. Escrita com efeito é decisão do modelo dentro do laço, e por isso é ferramenta — com idempotência, como a Aula 05 exige. Leitura que a aplicação sempre precisa fazer não deveria depender de o modelo lembrar de pedir.

---

## Fontes e leituras

ANTHROPIC. **Introducing the Model Context Protocol**. 2024.

MODEL CONTEXT PROTOCOL. **Specification**: revisões 2025-03-26, 2025-06-18, 2025-11-25 e 2026-07-28.

MICROSOFT. **Language Server Protocol Specification**.

LANGCHAIN. **MCP integration**. <https://docs.langchain.com/oss/python/langchain/mcp/index> — o adaptador do framework. Atenção à versão: `langchain.mcp.MCPAdapter` exige `langchain>=1.4`; abaixo disso o caminho é o pacote `langchain-mcp-adapters`, que é o fixado nesta disciplina.

**Material da disciplina.** Aula 03, [nota 04](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) — o ciclo de *tool calling* e a lista de pendências do §9. Aula 05, [nota 01](../../aula-08-memoria/notas-de-aula/01-o-agente-e-o-estado.md) — o estado explícito e as ferramentas escritas à mão; [nota 02](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md) — orçamento, término e classificação de erro, nenhum deles fornecido pelo protocolo.
