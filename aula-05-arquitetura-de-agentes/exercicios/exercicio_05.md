# Exercício 5 — O analista de prestação de contas

## Contexto

O atendente do exercício 3 funciona. E, se vocês fizeram a autópsia em sala, já sabem o que há de errado com ele: **quatro das cinco etapas estavam no código**. A sequência estava numerada no enunciado, e mesmo assim o programa inteiro foi escrito como um laço em que o modelo decide tudo.

Aquilo foi certo para aprender o mecanismo. Agora vocês têm que escolher a arquitetura.

O problema deste exercício foi montado para que a escolha errada **apareça na conta**. Ele é um lote de despesas para conferir contra uma política de reembolso, e a maioria esmagadora dos itens é decidida por uma regra de três linhas. Se vocês mandarem tudo para o modelo, vai funcionar — e vai custar dez vezes mais, com resultado pior justamente nos casos fáceis.

> A pergunta que vocês devem conseguir responder no fim: **quantos dos 8 itens precisaram de LLM?**

## Objetivo

Criar um programa `07-analista.py` que processa um lote de despesas e produz um parecer por item, usando **quatro padrões diferentes**, cada um no lugar certo:

```
 ┌────────────────────────────────────────────────────────────────┐
 │  lote de 8 despesas                                            │
 │         │                                                      │
 │         ▼                                                      │
 │  ┌─────────────┐   dentro da política ──> REGRA EM CÓDIGO      │
 │  │  ROTEADOR   │   ambíguo ─────────────> AGENTE COM ESTADO    │
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

Repare no desenho: **a autonomia cara está confinada a um caminho estreito**. O volume passa por código.

## O sistema (dados de mentira, para o exercício rodar)

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

Os oito itens criam situações **diferentes de propósito**:

| Despesa | A situação | O que se espera do sistema |
| --- | --- | --- |
| `D-4471` | R$ 84, uma pessoa, com nota | **regra pura** — dentro do teto. Não deve chamar o modelo |
| `D-4472` | táxi R$ 45, sem nota (não exige) | **regra pura** — dentro. Não deve chamar o modelo |
| `D-4473` | R$ 312, **três pessoas**, com nota | **ambíguo** — o teto é *por pessoa*: R$ 104/pessoa. Precisa ler a política e o número de comensais |
| `D-4474` | R$ 1.240 | **acima da alçada** — nem o agente nem a regra decidem. Humano |
| `D-4475` | declarado R$ 96, **recibo diz R$ 196** | **divergência** — o teste do raciocínio |
| `D-4476` | material R$ 50, **sem nota**, e a categoria exige | **ambíguo** — está no teto e viola outro requisito |
| `D-4477` | funcionário `F-88` — **este id não existe** | **erro de ferramenta recuperável**: o certo é `F-088` |
| `D-4478` | táxi R$ 130, teto R$ 90, com nota e justificativa | **ambíguo** — viola o teto, mas a descrição pede análise |

## Requisitos

### 1. A triagem — o roteador

Antes de qualquer coisa, cada despesa passa por uma triagem que devolve **uma rota**, com saída estruturada e `enum` (Aula 02, nota 02 §7):

```python
ROTAS = ["dentro_da_politica", "ambiguo", "acima_da_alcada", "nenhuma"]
```

Regras:

- **A rota `acima_da_alcada` é decidida em código, antes de qualquer chamada de LLM.** Valor acima de R$ 500 não é assunto de modelo — é regra da empresa. Deixar o modelo decidir isso é abrir mão de uma garantia por nada.
- **A rota `dentro_da_politica` deve ser resolvida por código sempre que a regra bastar** — categoria, teto, presença de nota. Um item que a regra resolve **não deve gerar nenhuma chamada de LLM**.
- **A rota `nenhuma` é obrigatória** e vai para uma fila de revisão. Nada de forçar uma classificação que não serve.
- O programa **conta e imprime**, ao final: quantos itens foram resolvidos por regra, quantos pelo agente, quantos foram para humano e quantos para a fila.

> Vocês podem implementar a triagem inteira em código, ou usar o modelo apenas onde a regra não alcança. As duas escolhas são defensáveis — **o que não é defensável é não saber justificar a sua**. Escreva a justificativa em comentário, no código.

### 2. O lote — orquestrador-trabalhador

O parecer do lote não é a concatenação dos pareceres individuais. Quando o lote termina, um orquestrador decide **quais análises agregadas fazem sentido** para *este* lote — e essas subtarefas não estão no seu código.

Exemplos do que ele pode decidir (não são obrigatórios, e é esse o ponto): agrupar as violações por funcionário, cruzar com o histórico de quem já foi reprovado antes, destacar o padrão de uma categoria específica.

Requisitos:

- o plano do orquestrador vem em **saída estruturada**, com uma lista de subtarefas;
- **teto de subtarefas** — se o plano vier com mais que o teto, aborte com erro claro. Um orquestrador sem teto é uma conta aberta;
- as subtarefas são executadas e sintetizadas num parecer único do lote.

### 3. Os ambíguos — o agente com estado

Só os itens roteados como `ambiguo` viram agente. E este agente **não** é o laço da aula 03: ele opera sobre um objeto de estado (nota 02).

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

- **exposição por fase**: `registrar_parecer` **não** pode estar declarada enquanto o agente ainda está analisando. Use `ferramentas_ativas` (nota 02, §6);
- a função devolve o **`Estado`**, não uma string;
- todo passo é registrado, com ferramenta, argumentos e erro em **campos** — não em texto.

### 4. O parecer final — avaliador-otimizador

O texto do parecer do lote passa por um avaliador antes de ser dado como pronto.

O critério **tem que estar escrito** e ser verificável item a item — não peça "avalie se está bom". No mínimo:

```python
{"cita_artigo": bool,      # o parecer cita o artigo da política aplicado?
 "cita_valores": bool,     # menciona os valores exatos?
 "conclui": bool,          # termina com veredito explícito por item?
 "o_que_corrigir": str}
```

E:

- **teto de rodadas** (3 é suficiente);
- se sair por teto, o retorno precisa **dizer** que saiu por teto — não entregue melhor esforço como se fosse aprovação.

### 5. Confiabilidade — tudo verificável no log

Estes cinco itens são obrigatórios e vão ser conferidos no log que vocês entregam:

**5.1 Orçamento nas quatro moedas** — passos, tokens, reais e tempo de parede. Passado como **parâmetro**, não como constante no meio do arquivo. Justifique os valores escolhidos em comentário (use a conta da Aula 02, nota 04).

**5.2 As quatro formas de terminar**, registradas no estado: `RESPONDEU`, `ORCAMENTO`, `ERRO_FATAL`, `HUMANO`. O log de cada execução diz **qual** ocorreu e por quê.

**5.3 Erro recuperável × fatal.** A despesa `D-4477` tem um id de funcionário inválido, de propósito. O retorno de erro precisa **ensinar**: o que estava errado, qual era o formato certo e o que fazer agora. Um `{"erro": "não encontrado"}` não cumpre o requisito.

**5.4 Chave de idempotência** em `registrar_parecer`, derivada do **conteúdo** (não `uuid4()`), e o retorno avisando `ja_existia` quando a ação já havia ocorrido.

**5.5 Detector de chamada repetida** ligado, com limite configurável. Ao detectar, o programa deve **intervir antes de abortar** — injetar a observação e dar mais uma chance ao agente. Registre no log quando disparou.

### 6. O contexto — medir antes de otimizar

O programa registra, para cada passo do agente, **quantos tokens foram enviados**. Ao final, imprime a curva por passo e o total acumulado.

Não é preciso implementar compaction (está nos desafios). É preciso **ter o número**: sem ele, qualquer otimização de contexto é chute.

### 7. Prompts em arquivo, versionados — e agora a arquitetura também

Continua valendo o requisito da aula 03: os prompts vivem em `prompts/`, versionados, e cada etapa declara a combinação `prompt × modelo × parâmetros`, carimbada no início da execução.

**O que muda nesta aula:** a **arquitetura entra no carimbo**.

```
etapa=triagem      arquitetura=regra_em_codigo   prompt=—              modelo=—
etapa=ambiguo      arquitetura=agente_react      prompt=analise-v3     modelo=... temp=0
etapa=lote         arquitetura=orq_trabalhador   prompt=orquestra-v1   modelo=... temp=0
etapa=parecer      arquitetura=avaliador_otim    prompt=avaliador-v2   modelo=... temp=0
```

Trocar workflow por agente numa etapa é mudança de versão **tanto quanto** trocar o prompt, e invalida a comparação com as execuções anteriores. Se vocês mudarem a arquitetura de uma etapa depois de rodar, incrementem a versão e rodem as oito despesas de novo.

## O que deve sair na tela

Para cada despesa, uma linha de roteamento; para os ambíguos, a trajetória; no fim, o parecer do lote e o resumo:

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

Os números acima são ilustrativos — os de vocês serão outros. O que importa é que **todos eles existam**.

## Desafios opcionais

1. **Compaction.** Implemente *tool clearing* no agente e compare a curva de tokens por passo com e sem. Mostre o que o resumo perdeu.
2. **Checkpoint.** Salve o estado a cada passo e faça o item `D-4474` (o de alçada humana) **pausar de verdade**: gravar, terminar o processo, e um segundo comando retomar com a aprovação e concluir sem repetir passo nenhum.
3. **Suíte de regressão.** Escreva 8 casos com critério por propriedade (nota 03 da aula 03) e rode a suíte contra duas versões da mesma etapa. Critério k/N, não igualdade de string.
4. **Detector de progresso nulo.** O detector do requisito 5.5 pega repetição idêntica. Implemente também o caso em que as chamadas variam e nada avança.

## Entrega

No repositório de vocês:

- `07-analista.py` (e `prompts/`, com as versões);
- `aula05-log.txt` — a execução completa das oito despesas, com o resumo final;
- os comentários no código respondendo às três justificativas obrigatórias: **por que cada etapa tem a arquitetura que tem**, **por que o orçamento tem os valores que tem** e **por que a triagem usa (ou não usa) o modelo**.

Sem relatório à parte. Como nos exercícios anteriores, **as justificativas moram no código**.

## Dicas

- Comece pela triagem em código puro, sem nenhuma chamada de LLM, e rode. Você vai descobrir quantos itens já estão resolvidos antes de escrever a primeira linha de agente — e essa é a lição do exercício.
- Escreva o `Estado` **antes** do laço. Se você começar pelo laço, vai acabar com `mensagens[]` de novo, e todos os requisitos do item 5 ficam impossíveis.
- Provoque os erros de propósito: rode com o detector desligado uma vez, para ver o laço acontecer, e ligue depois. O log dos dois é mais instrutivo que o código.
- Um retorno de erro bom economiza mais tokens que qualquer compaction. Antes de otimizar contexto, releia os textos de erro que você escreveu.
