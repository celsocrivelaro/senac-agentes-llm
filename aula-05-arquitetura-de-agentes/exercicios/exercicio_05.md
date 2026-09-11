# Exercício 5 — O analista de prestação de contas

## Contexto

O exercício 3 produziu um atendimento de cinco etapas implementado como agente. A autópsia desta aula mostrou que quatro daquelas etapas estavam no código: era um *workflow* com dois momentos de decisão. Aquela escolha servia ao objetivo de então — o mecanismo de *tool calling*. **Aqui o objeto é a escolha da arquitetura.**

O domínio é novo de propósito: um lote de despesas a conferir contra uma política de reembolso. Ele foi construído para que **a escolha errada apareça na conta** — a maioria dos itens se decide por regra determinística, e mandar todos ao modelo produz o resultado certo gastando cerca de dez vezes mais, com qualidade pior justamente nos casos triviais.

> **Questão a ser respondida ao final:** quantos dos 8 itens exigiram chamada ao modelo — e quantos você esperava que exigissem?

> **Este exercício é o par prático do [exercicio_05-trabalho.md](exercicio_05-trabalho.md).** Lá você desenha a arquitetura do seu case; aqui você implementa uma arquitetura já desenhada, para sentir no código o que cada padrão cobra.

## Objetivo

Implementar `08-analista.py`, que processa o lote e produz um parecer por item, com **quatro padrões**, cada um na etapa em que ele é o certo:

```
  lote de 8 despesas
        │
        ▼
  ┌─────────────┐   dentro da política ──> REGRA EM CÓDIGO (sem LLM)
  │   ROUTER    │   ambíguo ─────────────> AGENTE COM ESTADO
  │             │   acima da alçada ─────> HUMANO (pausa)
  └─────────────┘   nenhuma ─────────────> fila de revisão
        │
        ▼
  ┌──────────────────────────┐
  │ ORQUESTRADOR-TRABALHADOR │   monta o parecer do lote
  └──────────────────────────┘
        │
        ▼
  ┌──────────────────────────┐
  │   AVALIADOR-OTIMIZADOR   │   critica e revisa o texto final
  └──────────────────────────┘
```

**A autonomia cara está confinada a um caminho estreito**; o volume trafega por código determinístico. É o desenho que a [nota 01-6](../notas-de-aula/01-6-qual-padrao-usar.md) §5 defende — e que o `05-trabalho.md` pede que você justifique para o seu próprio case.

## Dados do problema

```python
from datetime import date

HOJE = date(2026, 9, 15)         # data fixa, para o exercício ser reproduzível

# ---------------------------------------------------------------- política
POLITICA = {
    "refeicao":   {"artigo": "Art. 4", "teto_por_pessoa": 120.00, "exige_nota": True},
    "transporte": {"artigo": "Art. 5", "teto_unitario":   90.00, "exige_nota": False},
    "hospedagem": {"artigo": "Art. 6", "teto_diaria":    380.00, "exige_nota": True},
    "material":   {"artigo": "Art. 9", "teto_unitario":   50.00, "exige_nota": True},
}

ALCADA_ANALISTA = 500.00         # acima disso, decisão é humana. Sempre.

FUNCIONARIOS = {
    "F-088": {"nome": "Ana Souza",   "centro_custo": "COMERCIAL"},
    "F-091": {"nome": "Bruno Lima",  "centro_custo": "TECNOLOGIA"},
    "F-103": {"nome": "Célia Rocha", "centro_custo": "COMERCIAL"},
}

# ---------------------------------------------------------------- o lote
DESPESAS = {
    "D-4471": {"funcionario": "F-088", "categoria": "refeicao",   "valor":   84.00,
               "pessoas": 1, "tem_nota": True,  "descricao": "almoço em visita a cliente"},

    "D-4472": {"funcionario": "F-091", "categoria": "transporte", "valor":   45.00,
               "pessoas": 1, "tem_nota": False, "descricao": "táxi aeroporto-hotel"},

    "D-4473": {"funcionario": "F-088", "categoria": "refeicao",   "valor":  312.00,
               "pessoas": 3, "tem_nota": True,  "descricao": "jantar com equipe do cliente"},

    "D-4474": {"funcionario": "F-103", "categoria": "hospedagem", "valor": 1240.00,
               "diarias": 2, "tem_nota": True,  "descricao": "hotel, congresso setorial"},

    "D-4475": {"funcionario": "F-091", "categoria": "refeicao",   "valor":   96.00,
               "pessoas": 1, "tem_nota": True,  "descricao": "jantar, viagem a trabalho"},

    "D-4476": {"funcionario": "F-103", "categoria": "material",   "valor":   50.00,
               "pessoas": 1, "tem_nota": False, "descricao": "material de escritório"},

    "D-4477": {"funcionario": "F-88",  "categoria": "transporte", "valor":   38.00,
               "pessoas": 1, "tem_nota": False, "descricao": "aplicativo, reunião externa"},

    "D-4478": {"funcionario": "F-091", "categoria": "transporte", "valor":  130.00,
               "pessoas": 1, "tem_nota": True,  "descricao": "táxi, trajeto longo, madrugada"},
}

# ---------------------------------------------- o que o recibo digitalizado diz
# (nem sempre bate com o valor declarado — é de propósito)
RECIBOS = {
    "D-4471": 84.00,  "D-4472": 45.00,  "D-4473": 312.00, "D-4474": 1240.00,
    "D-4475": 196.00,                                     # <-- não bate
    "D-4476": 50.00,  "D-4477": 38.00,  "D-4478": 130.00,
}

HISTORICO = {                    # pareceres de meses anteriores
    "F-088": [{"despesa": "D-4102", "veredito": "aprovado",  "categoria": "refeicao"},
              {"despesa": "D-4188", "veredito": "aprovado",  "categoria": "refeicao"}],
    "F-091": [{"despesa": "D-4210", "veredito": "reprovado", "categoria": "transporte",
               "motivo": "acima do teto unitário, sem justificativa"}],
    "F-103": [],
}

PARECERES = {}                   # preenchido por registrar_parecer()
```

Os oito itens constituem situações deliberadamente distintas:

| Despesa | A situação | O que se espera do sistema |
| --- | --- | --- |
| `D-4471` | R$ 84, uma pessoa, com nota | **regra pura** — dentro do teto. Não deve chamar o modelo |
| `D-4472` | táxi R$ 45, sem nota (não exige) | **regra pura** — dentro. Não deve chamar o modelo |
| `D-4473` | R$ 312, **três pessoas**, com nota | **ambíguo** — o teto é *por pessoa*: R$ 104/pessoa. Exige leitura da política e do número de comensais |
| `D-4474` | R$ 1.240 | **acima da alçada** — nem o agente nem a regra decidem. Humano |
| `D-4475` | declarado R$ 96, **recibo diz R$ 196** | **divergência** — o teste do raciocínio |
| `D-4476` | material R$ 50, **sem nota**, e a categoria exige | **ambíguo** — está no teto e viola outro requisito |
| `D-4477` | funcionário `F-88` — **este id não existe** | **erro de ferramenta recuperável**: o certo é `F-088` |
| `D-4478` | táxi R$ 130, teto R$ 90, com nota e justificativa | **ambíguo** — viola o teto, mas a descrição demanda análise |

## Requisitos

## Requisitos

### 1. A triagem — o *Router*

Saída estruturada com `enum` de quatro rotas, e **a rota `nenhuma` é obrigatória**: sem escape, o modelo escolhe uma rota inaplicável com alta confiança.

A rota de maior valor é a que **não chama o modelo**. `D-4471` e `D-4472` estão dentro da política, e uma regra em código resolve os dois — **se o seu sistema chamar o modelo para eles, é defeito**, e o relatório do fim vai mostrar.

Antes de classificar com o modelo, tente a regra. É a cascata do [`01-router.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula05-agentes/01-router.py).

### 2. O portão — padrão Sequencial

A despesa `D-4475` declara R$ 96 e o recibo diz R$ 196. **Isso não é trabalho do modelo**: é uma comparação entre dois números, e ela vai num `if`, antes de qualquer chamada.

Implemente um portão que confira, para todo item: o funcionário existe, a categoria existe na política, e **o valor declarado bate com o recibo**. Item que não passa **não segue** para a análise — vai com o motivo para a fila.

É a distinção que a [nota 01-1](../notas-de-aula/01-1-sequencial.md) faz entre validação de **forma** e de **fato**. `D-4477` cita `F-88`, que não existe: forma certa, fato errado.

### 3. Os ambíguos — o agente com estado

`D-4473`, `D-4476` e `D-4478` exigem ler a política, o histórico do funcionário e decidir. O agente roda com:

- **objeto de estado** — objetivo, trajetória, ferramentas ativas, término;
- **orçamento nos três tetos** — passos, tokens e tempo, passados como **parâmetro**;
- **término registrado** — o programa diz *por que* parou;
- **detector de laço** ligado.

`registrar_parecer` é **escrita**: chave de idempotência derivada do conteúdo, com `ja_existia` no retorno. E `D-4474` está acima da alçada: o agente **pausa**, grava e devolve o controle — `Termino.HUMANO`, sem `input()` no meio do laço.

### 4. O parecer do lote — orquestrador-trabalhador

As análises que o lote comporta **dependem do que a triagem produziu** — e por isso só existem em execução. Daí o `MAX_SUBTAREFAS`: um orquestrador sem teto é uma conta aberta assinada por um modelo.

### 5. A revisão — avaliador-otimizador

Critério **escrito**, item a item, com schema booleano: *cita o artigo da política? cita o valor? conclui?* Um `{"nota": 8}` não é acionável.

Teto de rodadas, e o retorno **declara** a saída por teto. Melhor esforço apresentado como aprovação é pior que reprovação explícita.

> **Antes de escrever o avaliador, leia a [nota 01-5](../notas-de-aula/01-5-avaliador-otimizador.md) §2.1.** Boa parte deste critério é executável — e o que é executável se verifica em código, não com um LLM.

### 6. O carimbo

Prompts em `prompts/`, versionados, com `prompt × modelo × parâmetros`. E esta aula acrescenta **a arquitetura**: trocar *workflow* por agente numa etapa é mudança de versão tanto quanto trocar o prompt, e invalida a comparação com as execuções anteriores.

## O que deve sair na tela

```
TRIAGEM
  <id>  rota=<qual>  [regra|modelo]

PORTÃO
  <id>  BARRADO — <motivo>

AGENTE (itens ambíguos)
  <id>  termino=<motivo>  passos=<n>  tokens=<n>

CONTA
  itens ................. 8
  resolvidos por REGRA .. <n>   (zero chamadas de LLM)
  enviados ao MODELO .... <n>
  chamadas totais ....... <n>
  se TUDO fosse ao modelo: ~<n> chamadas  (<n>x)
```

## Desafios opcionais

**A.** Rode o lote inteiro **sem** o portão e sem a regra, mandando os 8 itens ao agente. Compare as duas execuções em chamadas, e olhe o parecer da `D-4475` na versão sem portão.

**B.** Persista o *checkpoint* entre execuções: rode, interrompa no item 5, rode de novo e verifique que ele **retoma**. É o gancho da aula de memória.

## Entrega

No repositório do laboratório: `08-analista.py`, a pasta `prompts/` e o log das duas execuções do desafio A, se você o fizer.

**Parâmetros e justificativas no código**, em comentário — sem relatório à parte, como nos exercícios anteriores.

## Dicas

- **Comece pela regra, não pelo agente.** Escreva primeiro o código que decide `D-4471` e `D-4472` sem modelo nenhum. Se isso funcionar, você já tem 25% do lote resolvido e o número que o exercício pede.
- **O portão antes da triagem economiza mais que o portão depois.** Pense onde ele cabe.
- **`D-4477` é erro recuperável**, não fatal: o retorno precisa dizer o que errou, qual era o certo e o que fazer — e o modelo se corrige sozinho na volta seguinte.
- **Se o seu agente gastar mais de 8 passos num item**, o problema não é o teto: é o retorno de alguma ferramenta que não está ensinando nada.
