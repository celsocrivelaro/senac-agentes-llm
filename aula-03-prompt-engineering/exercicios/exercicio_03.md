# Exercício 3 - O atendente que pergunta, pensa e age

## Contexto

O abridor de chamados que vocês fizeram na aula 02 funciona — mas só quando o cliente escreve uma mensagem completa. Na vida real ele escreve assim:

> *"meu pedido não chegou"*

E pronto. Sem número, sem data, sem contexto. O programa da aula 02 não tem o que extrair, e abre um chamado inútil ou nenhum.

Falta o que um atendente humano faria: **perguntar o que falta, consultar o sistema, pensar sobre o que os dados dizem, e só então agir.**

É isso que vocês vão construir. E, sem nenhuma cerimônia, isso é um **agente**: um programa em que o modelo decide o que fazer a seguir, em vez de seguir uma sequência que você escreveu.

## Objetivo

Criar um programa `06-atendente.py` que conduz um atendimento completo em cinco etapas:

```
 ┌──────────────────────────────────────────────────────────────┐
 │ 1. COLETA        conversa com o cliente até ter o número do  │
 │                  pedido e o que aconteceu. PERGUNTA o que    │
 │                  falta, uma coisa de cada vez                │
 ├──────────────────────────────────────────────────────────────┤
 │ 2. CONSULTA      ferramenta: consultar_pedido(numero)        │
 │                  -> status real, previsão, transportadora    │
 ├──────────────────────────────────────────────────────────────┤
 │ 3. RACIOCÍNIO    com os dados reais em mãos, decidir qual é  │
 │                  o caso e qual a ação. Aqui vai o CoT        │
 ├──────────────────────────────────────────────────────────────┤
 │ 4. AÇÃO          ferramenta: abrir_chamado(...)              │
 │                  ESCRITA — exige confirmação do cliente      │
 ├──────────────────────────────────────────────────────────────┤
 │ 5. CONSULTA      ferramenta: consultar_chamado(protocolo)    │
 │    FINAL         confirma que foi criado e informa o cliente │
 └──────────────────────────────────────────────────────────────┘
```

As etapas 2, 4 e 5 são **ferramentas** — o modelo decide quando chamá-las, o seu código executa. A 1 e a 3 são **prompt**.

> Repare no que vocês vão montar: a etapa 4 é o programa da aula 02, virado ferramenta. É exatamente o que aquele exercício prometia no fim.

## O sistema (dados de mentira, para o exercício rodar)

```python
from datetime import date

HOJE = date(2026, 9, 8)          # data fixa, para o exercício ser reproduzível

PEDIDOS = {
    "48219": {"situacao": "em transporte",     "previsao": "2026-09-02",
              "transportadora": "RápidoLog",   "cliente": "Ana Souza"},
    "77310": {"situacao": "entregue",          "previsao": "2026-08-19",
              "transportadora": "RápidoLog",   "cliente": "Bruno Lima"},
    "90455": {"situacao": "aguardando coleta", "previsao": "2026-09-15",
              "transportadora": "TransBrasil", "cliente": "Ana Souza"},
    "31002": {"situacao": "em transporte",     "previsao": "2026-09-11",
              "transportadora": "TransBrasil", "cliente": "Célia Rocha"},
}

CHAMADOS = {}                    # preenchido por abrir_chamado()

CATEGORIAS = ["entrega_atrasada", "endereco_errado", "produto_avariado",
              "duvida", "elogio"]
```

Repare que os quatro pedidos criam situações **diferentes**, e é de propósito:

| Pedido | A situação | O que o raciocínio precisa concluir |
| --- | --- | --- |
| `48219` | previsão venceu há 6 dias, ainda em transporte | atraso real — abrir chamado, urgência alta |
| `77310` | consta **entregue** | o cliente diz que não recebeu, mas o sistema diz que sim. **Divergência** |
| `90455` | previsão só daqui a uma semana | **não** está atrasado — não é caso de chamado |
| `31002` | previsão daqui a 3 dias | idem |

## Requisitos

### 1. Etapa de coleta — o programa pergunta

O programa começa com uma mensagem do cliente que **não tem informação suficiente**, e conversa até ter o que precisa:

```
Cliente: meu pedido não chegou
Bot:     Sinto muito pelo transtorno. Você tem o número do pedido?
Cliente: acho que é 48219
Bot:     Obrigado. Você chegou a receber alguma tentativa de entrega?
Cliente: não, ninguém apareceu
```

Regras:

- **uma pergunta por vez.** Um bot que pergunta três coisas de uma vez recebe resposta de uma só;
- **não peça o que já foi dito.** Se o cliente já deu o número na primeira mensagem, não pergunte de novo — isso exige que o histórico esteja no contexto (nota 02);
- o programa decide **sozinho** quando tem o suficiente para seguir. Não use um contador de turnos fixo;
- **teto de turnos** mesmo assim, para o laço não ficar aberto se o cliente não colaborar.

### 2. Consulta ao sistema — a primeira ferramenta

```python
def consultar_pedido(numero: str) -> dict:
    ...
```

Escreva a **descrição** da ferramenta lembrando que ela é prompt (nota 04, §4): o que faz, **quando usar** e **quando não usar**. E devolva erro como **dado**, não como exceção — se o número não existir, o modelo precisa poder pedir de novo ao cliente.

### 3. Raciocínio — aqui, e só aqui, vai o CoT

Com o status real em mãos, o programa decide **qual é o caso**. Isso não é uma classificação de um passo: exige comparar a previsão com a data de hoje, confrontar o que o cliente disse com o que o sistema diz, e concluir.

Peça o raciocínio explícito, e faça o programa **imprimir**:

```
--- raciocínio ---
A previsão era 02/09 e hoje é 08/09, então o prazo venceu há 6 dias.
O sistema diz "em transporte", e o cliente diz que não houve tentativa
de entrega. Não é divergência: é atraso com o pedido ainda em trânsito.
Categoria: entrega_atrasada. Urgência: alta, porque passou de 5 dias.
Ação: acionar a transportadora e dar nova previsão ao cliente.
```

Duas exigências:

- **use CoT nesta etapa, e não nas outras.** Na coleta ele não ajuda (não há etapas a percorrer) e ainda deixa o bot prolixo. Escreva no código, em comentário, por que ele entra aqui;
- o raciocínio termina numa **conclusão estruturada** (categoria, urgência, ação) que o seu código consegue ler para chamar a próxima ferramenta.

> O caso do pedido `77310` existe para testar isso: o cliente diz que não recebeu, o sistema diz "entregue". Um programa que classifica sem raciocinar chama de `entrega_atrasada`. O caso é outro — e o seu raciocínio tem que dizer qual.

### 4. Abrir o chamado — ferramenta de **escrita**

```python
def abrir_chamado(pedido: str, categoria: str, urgencia: str,
                  descricao: str, acao_sugerida: str) -> dict:
    ...  # devolve {"protocolo": "2026-0842"}
```

Esta é diferente das outras: ela **muda o estado do sistema**. Ferramenta de escrita exige (nota 04, §7):

- **confirmação do cliente antes de executar.** O programa mostra o que vai abrir e pergunta;
- se o cliente disser que não, o programa **não abre** — e continua a conversa;
- **registro**: imprima o que foi aberto, com o protocolo.

E o campo `acao_sugerida` é obrigatório, como no exercício anterior: específico da situação, não genérico.

### 5. Consulta final — confirmar antes de encerrar

Depois de abrir, **consulte o chamado recém-criado** e informe o cliente com o protocolo:

```
Bot: Abri o chamado 2026-0842 para o pedido 48219. Categoria: atraso na
     entrega, urgência alta. A transportadora será acionada e retornaremos
     com uma nova previsão. Você pode acompanhar pelo protocolo.
```

Parece redundante — você acabou de criar o chamado, por que consultar? Porque **não é o mesmo programa que grava e que confirma**: o `abrir_chamado` pode ter falhado silenciosamente, gravado num campo errado, ou o protocolo pode não ser o que você acha. Confirmar lendo é o que separa "eu mandei criar" de "está criado".

> Esse hábito tem nome em sistemas distribuídos e vale para agentes: **não confie no retorno da sua própria escrita — leia de volta.**

### 6. Os parâmetros ficam no código

Como no exercício anterior: cada etapa tem a sua configuração, num bloco no topo do script, com um comentário dizendo **por quê**.

E aqui a diferença entre as etapas é maior que na aula 02 — são três naturezas distintas:

| Etapa | O que ela é | Variação na saída |
| --- | --- | --- |
| **Coleta** | conversa com humano | aceitável, até desejável |
| **Raciocínio** | decisão sobre dados | **defeito** |
| **Redação final** | texto para humano | aceitável |

Se as três saírem com a mesma configuração, algo está errado.

### 7. Robustez

- **teto de passos** no laço do agente, e teto de turnos na coleta;
- checar `finish_reason` — `"length"` não é resposta;
- `429` com **backoff exponencial**;
- ferramenta desconhecida ou argumento inválido devolvem erro **como dado**;
- ao final, imprimir o número de chamadas ao modelo e de chamadas a ferramentas.

## Os quatro atendimentos que você deve entregar

Rode o programa uma vez para cada situação, com estas aberturas:

1. **"meu pedido não chegou"** → o cliente informa `48219` quando perguntado. Deve terminar com chamado aberto, urgência alta.
2. **"o 77310 nunca chegou aqui"** → o sistema diz **entregue**. O raciocínio tem que perceber a divergência, e o chamado (se for aberto) tem que refleti-la.
3. **"cadê meu pedido 90455?"** → **não** está atrasado. O programa deve explicar a previsão e **não abrir chamado**.
4. **"o 99999 sumiu"** → pedido inexistente. A ferramenta devolve erro, e o programa pede o número de novo em vez de quebrar.

## Desafios opcionais

- **Suíte de regressão.** Escreva 5 casos com o resultado esperado (categoria e se abre ou não chamado) e rode 3 vezes cada, com critério k/N (nota 03, §5). Depois mude uma palavra do prompt de raciocínio e rode de novo.
- **Roteamento de modelo.** Use um modelo pequeno para a coleta (conversa simples) e o modelo bom só para o raciocínio. Compare a qualidade.
- **A terceira divergência.** Acrescente ao `PEDIDOS` um caso em que o cliente reclama de avaria mas o pedido consta "aguardando coleta" — ou seja, ele nem saiu. O que o seu raciocínio faz?

## Entrega

- `06-atendente.py`
- **Os quatro atendimentos**, com a conversa completa, o bloco `--- raciocínio ---` de cada um e o log das chamadas de ferramenta
- Um print de erro tratado: pedido inexistente, `429` com backoff, ou `finish_reason="length"` detectado

## Dicas

- **Comece pelo caminho feliz** (atendimento 1), com uma ferramenta só. Depois acrescente as outras duas, depois os casos difíceis.
- **Você vai ver o ReAct rodando** (nota 04, §6.1): a cada volta, o modelo decide (*Thought*), o seu código executa (*Action*), o resultado volta (*Observation*). Imprima os três — fica muito mais fácil de depurar.
- Se o bot perguntar coisas que o cliente já respondeu, o problema é de **contexto**, não de prompt: confira se o histórico está indo inteiro na chamada (nota 02, §1).
- Se ele abrir chamado sem perguntar, o problema é o **system prompt**: ele não disse que ação de escrita precisa de confirmação.
- Se o raciocínio sair curto e genérico, confira se você pediu CoT **e** deu espaço para ele — `max_tokens` apertado corta o raciocínio antes da conclusão (nota 01, §7.2).
