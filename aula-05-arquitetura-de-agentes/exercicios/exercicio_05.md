# Exercício 5 — Analista de prestação de contas

## Contexto

O exercício 3 produziu um atendimento em cinco etapas implementado como agente.
A autópsia da Aula 05 estabeleceu que quatro das cinco etapas constavam do
código: tratava-se de um *workflow* com dois momentos de decisão. Aquela escolha
era adequada ao objetivo de então — o mecanismo de *tool calling*. Este exercício
tem por objeto a escolha da arquitetura.

O problema foi construído de modo que a escolha incorreta se manifeste no custo.
Trata-se de um lote de despesas a conferir contra uma política de reembolso, em
que a maioria dos itens é decidida por regra determinística. Encaminhar todos os
itens ao modelo produz resultado correto a um custo aproximadamente dez vezes
maior, e com qualidade inferior justamente nos casos triviais.

> **Questão a ser respondida ao final:** quantos dos 8 itens exigiram chamada ao
> modelo?

## Objetivo

Implementar o programa `07-analista.py`, que processa um lote de despesas e
produz um parecer por item, empregando **quatro padrões de arquitetura**, cada um
na etapa adequada:

```
 ┌────────────────────────────────────────────────────────────────┐
 │  lote de 8 despesas                                            │
 │         │                                                      │
 │         ▼                                                      │
 │  ┌─────────────┐   dentro da política ──> REGRA EM CÓDIGO      │
 │  │  ROUTER     │   ambíguo ─────────────> AGENTE COM ESTADO    │
 │  │             │   acima da alçada ─────> HUMANO (pausa)       │
 │  └─────────────┘   nenhuma ─────────────> fila de revisão      │
 │         │                                                      │
 │         ▼                                                      │
 │  ┌──────────────────────────┐                                  │
 │  │ ORQUESTRADOR-TRABALHADOR │  monta o parecer do lote         │
 │  └──────────────────────────┘                                  │
 │         │                                                      │
 │         ▼                                                      │
 │  ┌──────────────────────────┐                                  │
 │  │  AVALIADOR-OTIMIZADOR    │  critica e revisa o texto final  │
 │  └──────────────────────────┘                                  │
 └────────────────────────────────────────────────────────────────┘
```

No desenho, **a autonomia cara está confinada a um caminho estreito**; o volume
trafega por código determinístico.

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

### 1. A triagem — o *Router*

Cada despesa passa por uma triagem que devolve **uma rota**, em saída estruturada com `enum` (Aula 02, nota 02, §7):

```python
ROTAS = ["dentro_da_politica", "ambiguo", "acima_da_alcada", "nenhuma"]
```

Regras:

- **A rota `acima_da_alcada` é decidida em código, antes de qualquer chamada ao modelo.** Valor acima de R$ 500 é regra da empresa, não objeto de inferência: delegá-la ao modelo suprime uma garantia sem contrapartida.
- **A rota `dentro_da_politica` é resolvida por código sempre que a regra bastar** — categoria, teto, presença de nota fiscal. Item resolvido por regra **não deve gerar chamada ao modelo**.
- **A rota `nenhuma` é obrigatória** e encaminha à fila de revisão. Classificação forçada entre opções inaplicáveis é erro silencioso.
- O programa **conta e imprime**, ao final: quantos itens foram resolvidos por regra, quantos pelo agente, quantos foram para humano e quantos para a fila.

> A triagem admite implementação integralmente em código ou emprego do modelo apenas onde a regra não alcança. Ambas as escolhas são defensáveis; a ausência de justificativa não é. **Registrar a justificativa em comentário, no código.**

### 2. O lote — orquestrador-trabalhador

O parecer do lote não é a concatenação dos pareceres individuais. Concluído o lote, um orquestrador determina **quais análises agregadas se aplicam** a *este* lote — e essas subtarefas não constam do código.

Exemplos de decisões possíveis — nenhuma obrigatória, e é esse o ponto: agrupar violações por funcionário, cruzar com o histórico de reprovações anteriores, destacar o padrão de uma categoria específica.

Requisitos:

- o plano do orquestrador vem em **saída estruturada**, com uma lista de subtarefas;
- **teto de subtarefas** — plano que exceda o teto aborta com erro explícito. Orquestrador sem teto é conta aberta;
- as subtarefas são executadas e sintetizadas num parecer único do lote.

### 3. Os ambíguos — o agente com estado

Apenas os itens roteados como `ambiguo` são processados por agente. Esse agente **não** é o laço da Aula 03: opera sobre um objeto de estado (nota 02).

O `Estado` precisa ter, no mínimo:

```python
objetivo · passos (ferramenta, argumentos, resultado, erro, tokens)
tokens_gastos · custo_estimado · ferramentas_ativas · termino · resposta
```

Ferramentas disponíveis ao agente (todas de leitura, exceto a última):

| Ferramenta | O que faz |
|---|---|
| `buscar_despesa(id)` | devolve o registro de `DESPESAS` |
| `consultar_politica(categoria)` | devolve o artigo, o teto e se exige nota |
| `consultar_historico(funcionario)` | devolve os pareceres anteriores |
| `ler_recibo(despesa_id)` | devolve o valor do recibo digitalizado |
| `registrar_parecer(...)` | **escrita** — idempotente, ver requisito 5 |

Requisitos:

- **exposição por fase**: `registrar_parecer` **não** pode constar das declarações enquanto o agente está em análise. Empregar `ferramentas_ativas` (nota 02, §6);
- a função devolve o **`Estado`**, não uma string;
- todo passo é registrado, com ferramenta, argumentos e erro em **campos** — não em texto.

### 4. O parecer final — avaliador-otimizador

O texto do parecer do lote é submetido a um avaliador antes de ser dado como concluído.

O critério **deve estar escrito** e ser verificável item a item; formulações como "avalie se está bom" não atendem ao requisito. No mínimo:

```python
{"cita_artigo": bool,      # o parecer cita o artigo da política aplicado?
 "cita_valores": bool,     # menciona os valores exatos?
 "conclui": bool,          # termina com veredito explícito por item?
 "o_que_corrigir": str}
```

E:

- **teto de rodadas** (3 é suficiente);
- saída por teto deve ser **declarada** como tal no retorno; melhor esforço apresentado como aprovação não atende ao requisito.

### 5. Confiabilidade — tudo verificável no log

Os cinco itens a seguir são obrigatórios e serão conferidos no log entregue:

**5.1 Orçamento nas quatro moedas** — passos, tokens, unidade monetária e tempo de parede, passado como **parâmetro** e não como constante no interior do arquivo. Justificar os valores em comentário, a partir da conta da Aula 02, nota 04.

**5.2 As quatro formas de terminar**, registradas no estado: `RESPONDEU`, `ORCAMENTO`, `ERRO_FATAL`, `HUMANO`. O log de cada execução diz **qual** ocorreu e por quê.

**5.3 Erro recuperável × fatal.** A despesa `D-4477` contém identificador de funcionário inválido, deliberadamente. O retorno de erro deve informar o que estava incorreto, qual o formato esperado e qual a próxima ação. `{"erro": "não encontrado"}` não atende ao requisito.

**5.4 Chave de idempotência** em `registrar_parecer`, derivada do **conteúdo** (não `uuid4()`), e o retorno avisando `ja_existia` quando a ação já havia ocorrido.

**5.5 Detector de chamada repetida** ativo, com limite configurável. Ao detectar, o programa deve **intervir antes de abortar**: injetar a observação e permitir nova tentativa. Registrar no log o instante de disparo.

### 6. O contexto — medir antes de otimizar

O programa registra, para cada passo do agente, **o número de tokens enviados**, e ao final imprime a curva por passo e o total acumulado.

A implementação de *compaction* não é exigida (consta dos desafios). A obtenção do número é: sem ele, qualquer otimização de contexto é conjectura.

### 7. Prompts em arquivo, versionados — e agora a arquitetura também

Permanece válido o requisito da Aula 03: os prompts residem em `prompts/`, versionados, e cada etapa declara a combinação `prompt × modelo × parâmetros`, registrada no início da execução.

**O que muda nesta aula:** a **arquitetura entra no carimbo**.

```
etapa=triagem      arquitetura=regra_em_codigo   prompt=—              modelo=—
etapa=ambiguo      arquitetura=agente_react      prompt=analise-v3     modelo=... temp=0
etapa=lote         arquitetura=orq_trabalhador   prompt=orquestra-v1   modelo=... temp=0
etapa=parecer      arquitetura=avaliador_otim    prompt=avaliador-v2   modelo=... temp=0
```

Substituir *workflow* por agente numa etapa constitui mudança de versão **tanto quanto** alterar o prompt, e invalida a comparação com execuções anteriores. Alterada a arquitetura de uma etapa após a execução, incrementa-se a versão e reexecutam-se as oito despesas.

## O que deve sair na tela

Para cada despesa, uma linha de roteamento; para os itens ambíguos, a trajetória; ao final, o parecer do lote e o resumo:

```
D-4471  rota=dentro_da_politica   [regra]     0 chamadas
D-4472  rota=dentro_da_politica   [regra]     0 chamadas
D-4473  rota=ambiguo              [agente]    4 passos  1.980 tokens  R$ 0,0048  -> RESPONDEU
...
D-4477  rota=ambiguo              [agente]    3 passos  1.510 tokens  R$ 0,0036  -> RESPONDEU
        passo 0: consultar_historico(F-88) -> ERRO RECUPERÁVEL (esperado F-XXX)
        passo 1: consultar_historico(F-088) -> ok

RESUMO DO LOTE
  por regra: 2   |   por agente: 4   |   humano: 1   |   fila: 1
  chamadas de LLM: 17     tokens: 24.310     custo: R$ 0,058
  se tudo tivesse ido para o agente: ~32 chamadas (estimativa)
```

Os números acima são ilustrativos. O requisito é que **todos eles existam** na saída.

## Desafios opcionais

1. **Compaction.** Implementar *tool clearing* no agente e comparar a curva de tokens por passo com e sem a tática. Identificar a informação perdida no resumo.
2. **Checkpoint.** Persistir o estado a cada passo e fazer o item `D-4474` (alçada humana) suspender efetivamente a execução: gravar, encerrar o processo e, em segundo comando, retomar com a aprovação e concluir sem repetição de passos.
3. **Suíte de regressão.** Redigir 8 casos com critério por propriedade (Aula 03, nota 03) e executar a suíte contra duas versões da mesma etapa. Critério k/N, não igualdade de cadeia de caracteres.
4. **Detector de progresso nulo.** O detector do requisito 5.5 captura repetição idêntica. Implementar também o caso em que as chamadas variam sem que o estado avance.

## Entrega

No repositório do grupo:

- `07-analista.py` (e `prompts/`, com as versões);
- `aula05-log.txt` — a execução completa das oito despesas, com o resumo final;
- comentários no código com as três justificativas obrigatórias: **a arquitetura adotada em cada etapa**, **os valores do orçamento** e **o emprego ou não do modelo na triagem**.

Não há relatório em separado. Como nos exercícios anteriores, **as justificativas residem no código**.

## Dicas

- Iniciar pela triagem em código puro, sem chamada ao modelo, e executar. O número de itens já resolvidos antes da primeira linha de agente é a lição central do exercício.
- Escrever o `Estado` **antes** do laço. Iniciar pelo laço reconduz a `mensagens[]` como estrutura de estado, e todos os requisitos do item 5 tornam-se inviáveis.
- Provocar os erros deliberadamente: executar uma vez com o detector desativado, observar o laço, e ativá-lo em seguida. O log das duas execuções é mais instrutivo que o código.
- Um retorno de erro adequado economiza mais tokens que qualquer *compaction*. Antes de otimizar contexto, revisar os textos de erro redigidos.
