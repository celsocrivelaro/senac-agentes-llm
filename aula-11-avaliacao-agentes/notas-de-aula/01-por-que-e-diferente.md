# IA Aplicada com LLMs — Aula 11: Evals — Por que avaliar isto é diferente

> **Pré-requisitos:** Aula 02, [nota 01](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) e [nota 04](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/04-custo-latencia-e-decisao.md) · Aula 03, [nota 03](../../aula-03-prompt-engineering/notas-de-aula/03-prompt-como-codigo.md) · Aula 06, [nota 02](../../aula-06-embeddings-e-rag/notas-de-aula/02-chunking-e-a-medida-da-busca.md) · Aula 08, [nota 04](../../aula-08-memoria/notas-de-aula/04-esquecer-e-proteger.md).
>
> **Código:** [`aula11-evals/`](https://github.com/celsocrivelaro/senac-llm-code/tree/main/aula11-evals) — os scripts `02-golden-set.py` e `03-pass-k.py` correspondem a esta nota.

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Identificar** as três fontes de não determinismo de um sistema com LLM e distingui-las por natureza.
- **Explicar** por que a métrica de um sistema desses é uma distribuição, e não um valor.
- **Distinguir** `pass@k` de `pass^k`, e justificar por que a segunda é a métrica adequada a agentes.
- **Construir** um *golden dataset* versionado, declarando a origem de cada caso.
- **Justificar** o tamanho e a composição do conjunto, e reconhecer o que o número médio esconde.

---

## Desenvolvimento teórico

### 1. O contraste com o teste de software convencional

Em software determinístico, a mesma entrada produz a mesma saída, e o teste é uma igualdade: `assert soma(2, 3) == 5`. Quando falha, falhou; quando passa, passou. A suíte é um conjunto de proposições verdadeiras ou falsas.

Aqui não é assim, e o problema não é de precisão numérica nem de tolerância de ponto flutuante. É que **o sistema sob teste não tem uma saída para cada entrada** — tem uma distribuição de saídas.

Disso decorre a primeira consequência prática, e ela é desconfortável: **rodar uma vez e reportar o número é reportar uma amostra de tamanho um**. A maioria dos relatórios de avaliação de sistemas com LLM que circulam publicamente faz exatamente isso.

### 2. As três fontes, e elas são diferentes em natureza

Confundi-las leva a decisões erradas, porque cada uma se combate de um jeito.

**Parâmetro de amostragem** (Aula 02). `temperature`, `top_p` e afins introduzem variação deliberada. É configurável, e pode ser zerada. É a fonte mais fácil, e é a única que a maioria das pessoas lembra.

**O provedor.** Mesmo com `temperature=0`, não há garantia de reprodutibilidade. Ordem de operações em ponto flutuante, distribuição de carga entre réplicas, e a própria versão do modelo servida podem variar. Esta fonte **não** é configurável: nenhum parâmetro a elimina.

**A memória** (Aula 08). Esta é a pior das três, e é a mais interessante, porque não é acidente: é **consequência de projeto**. A nota 03 daquela aula estabeleceu que memória torna o agente não reprodutível — a mesma entrada, no mesmo modelo, com os mesmos parâmetros, produz saída diferente amanhã, porque a memória mudou entre as duas execuções.

| Fonte | Controlável? | Como se lida |
|---|---|---|
| amostragem | sim | zerar, e declarar no carimbo |
| provedor | não | repetir e medir a dispersão |
| **memória** | **não, por projeto** | **fixar o estado da memória no carimbo** |

A terceira linha explica por que o carimbo da Aula 08 acrescentou o estado da memória: **duas execuções com memórias diferentes não são comparáveis**, e tratá-las como comparáveis produz números que variam sem que nada tenha sido alterado.

### 3. A métrica é uma distribuição

Se a saída varia, a pergunta "o sistema acerta esta tarefa?" está mal formulada. A pergunta correta é: **com que probabilidade ele acerta esta tarefa?**

E, para um agente que age sobre o mundo, há uma pergunta ainda mais dura: **com que probabilidade ele acerta esta tarefa todas as vezes?**

Duas métricas separam essas perguntas:

- **`pass@k`** — a probabilidade de **ao menos uma** de *k* tentativas acertar. É a métrica de geração de código: se uma das dez sugestões compila e passa nos testes, o desenvolvedor usa aquela.
- **`pass^k`** — a probabilidade de **todas** as *k* tentativas acertarem. É a métrica de confiabilidade.

Para um sistema que registra pareceres, envia mensagens ou movimenta dinheiro, `pass@k` é a métrica errada: não existe um humano escolhendo a melhor das dez execuções. Existem dez usuários, cada um recebendo uma.

O τ-bench (Yao et al., 2024) formalizou `pass^k` e produziu o número que serve de calibração para esta aula: agentes de ponta com chamada de função ficam **abaixo de 50% de sucesso** nas tarefas, e **abaixo de 25% em `pass^8`** no domínio de varejo. Passar uma vez não é passar.

> **Esta é a dívida da Aula 02 sendo paga.** A nota 04 daquela aula afirmou que "a média esconde a variância" e que, para agentes, "acertar uma vez é muito diferente de acertar sempre". Aqui a afirmação ganha nome e fórmula.

### 4. O *golden dataset*

**Definição.** Um *golden dataset* é um conjunto **revisado** e **versionado** de entradas representativas, cada uma acompanhada da saída esperada, de um rótulo, de uma rubrica ou de um contexto de referência.

As três palavras em destaque fazem trabalho. *Revisado* exclui casos gerados e aceitos sem inspeção. *Versionado* é o que torna duas execuções comparáveis. E *representativas* é a exigência mais difícil, porque só se sabe o que é representativo depois de operar.

### 5. Os três embriões que já existem

Este objeto já foi construído três vezes no curso, sem o nome:

| Aula | O que construiu | O que faltava |
|---|---|---|
| 03 | conjunto de teste para medir *few-shot* | escala e origem declarada |
| 06 | dez perguntas com o artigo-resposta conhecido | além da busca, a geração |
| 07 | o par `recall` × fidelidade | repetição, e o juiz validado |

Os três são o mesmo objeto em escalas diferentes. Projetá-los lado a lado, antes de qualquer definição nova, é o modo de estabelecer que esta aula não introduz um conceito: ela **dá método ao que já foi feito à mão**.

### 6. De onde vêm os casos

Três origens, e a ordem importa porque a qualidade decresce.

**Incidentes de produção.** Todo comportamento errado observado em operação vira caso permanente. Esta é a fonte mais valiosa, porque é a única que não depende de imaginação: alguém encontrou aquilo de verdade. A Aula 12 fecha esse ciclo, alimentando o *dataset* a partir da avaliação online.

**Erros observados no desenvolvimento.** O que quebrou enquanto o sistema era construído. Menos valioso que o anterior, e muito mais disponível.

**Casos sintéticos.** Gerados por modelo, para cobrir o que a operação ainda não exercitou. São legítimos, e têm uma condição: **revisão com o mesmo rigor dos reais**. Um conjunto sintético não revisado mede a capacidade do gerador de imaginar problemas, e não a do sistema de resolvê-los.

### 7. Quanto é suficiente

A pergunta é frequente e a resposta desagrada: **menos do que se imagina, se forem os casos certos**.

Cinquenta falhas reais revisadas por um especialista de domínio valem mais que quinhentos casos sintéticos não revisados. A régua da literatura para calibrar um juiz automático trabalha com 100 a 200 saídas avaliadas por dois ou três avaliadores independentes — e é da concordância entre eles, antes de qualquer automação, que sai a confiança no critério.

Duas exigências de composição valem mais que o tamanho:

- **cobrir as classes que importam**, e não apenas as frequentes. Um conjunto com noventa casos fáceis e dez difíceis produz uma média que se move quando os fáceis mudam;
- **incluir os casos de recusa**. Um sistema que deveria dizer "não sei" e diz outra coisa falha de um jeito que nenhum caso positivo detecta. A Aula 07 já havia construído esse tipo de caso.

---

## Exemplos

### Exemplo 1 — A mesma entrada, vinte vezes

```
tarefa: "A despesa D-4613 está dentro da política? Foi em Lisboa."
        (exige o art. 19 — exceção de viagem internacional)

  acertos ....... 13 / 20
  pass@1 ........ 0.65
  pass^5 ........ 0.12       <- probabilidade de acertar as 5 vezes
  pass^8 ........ 0.03

  distribuição das trajetórias:
    despesa -> politica -> regulamento   13x   (correta)
    despesa -> politica                   6x   (não buscou a exceção)
    despesa                               1x   (concluiu cedo demais)
```

Reportar "65%" para esse sistema é verdade e é enganoso. O número que o usuário sente é o `pass^k`.

### Exemplo 2 — Um caso do dataset, com origem declarada

```python
{
  "id": "C-005",
  "origem": "producao",           # producao | desenvolvimento | sintetico
  "incidente": "2026-08-14, ticket #331",
  "entrada": "A despesa D-4613 está dentro da política? Foi em Lisboa.",
  "esperado": {"veredito": "aprovado", "cita": "art-19"},
  "classe": "excecao_internacional",
  "por_que_existe": "o agente aprovou pelo teto nacional e não citou a exceção; o parecer foi contestado pelo funcionário"
}
```

O campo `por_que_existe` não é documentação: é o que permite decidir, meses depois, se o caso ainda faz sentido quando a política mudar.

### Exemplo 3 — Um *golden dataset* pequeno, inteiro

Seis casos do agente de despesas, um de cada tipo que a composição exige. A versão do conjunto vai no topo, porque é ela que torna duas execuções comparáveis.

```python
VERSAO_DATASET = "1.2.0"

CASOS = [
    # simples — a referência: se este falhar, nada mais importa
    {"id": "C-001", "origem": "desenvolvimento", "classe": "simples",
     "entrada": "A despesa D-4612 está dentro da política?",        # refeição, R$ 138
     "esperado": {"veredito": "reprovado", "cita": "art-7"},
     "por_que_existe": "acima do teto, sem exceção"},

    # exceção — veio de um incidente, e por isso tem data e ticket
    {"id": "C-006", "origem": "producao", "classe": "excecao_internacional",
     "incidente": "2026-08-22, ticket #349",
     "entrada": "Uma refeição de R$ 180 em viagem internacional é reembolsável?",
     "esperado": {"veredito": "aprovado", "cita": "art-19"},
     "por_que_existe": "120 x 1,6 = 192; o agente reprovou por comparar com 120"},

    # exceção negativa — sintético, e por isso tem revisor
    {"id": "C-008", "origem": "sintetico", "classe": "excecao_internacional",
     "revisor": "professor",
     "entrada": "Hospedagem de R$ 900 em capital e em viagem internacional. Os acréscimos somam?",
     "esperado": {"veredito": "reprovado", "cita": "art-19"},
     "por_que_existe": "o art. 19 veda a cumulação"},

    # recusa — fora do corpus: a resposta certa é não responder
    {"id": "C-009", "origem": "producao", "classe": "recusa",
     "incidente": "2026-07-30, ticket #302",
     "entrada": "Qual é o procedimento para solicitar férias coletivas?",
     "esperado": {"veredito": "recusa", "cita": None},
     "por_que_existe": "o agente inventou um procedimento"},

    # recusa — o erro da ferramenta precisa virar recusa, não palpite
    {"id": "C-010", "origem": "desenvolvimento", "classe": "recusa",
     "entrada": "A despesa D-9999 está dentro da política?",
     "esperado": {"veredito": "recusa", "cita": None},
     "por_que_existe": "id inexistente"},

    # escrita — o esperado inclui o estado do mundo depois
    {"id": "C-013", "origem": "producao", "classe": "escrita",
     "incidente": "2026-08-05, ticket #318",
     "entrada": "Registre o parecer da despesa D-4612.",
     "esperado": {"veredito": "reprovado", "cita": "art-7", "pareceres_criados": 1},
     "por_que_existe": "dois pareceres para a mesma despesa após um retry"},
]
```

Seis casos não medem taxa nenhuma com precisão, e não é para isso que servem aqui: cada um ocupa um lugar que nenhum outro cobre. Retirar C-009 deixa o conjunto sem recusa; retirar C-013 deixa a escrita sem verificação de estado. O conjunto completo do laboratório tem 14 casos em `casos.py`, com a mesma estrutura.

### Exemplo 4 — As quatro formas do "esperado"

A definição da seção 4 admite quatro formas de referência. O agente de despesas usa a primeira; outros sistemas pedem as outras.

```python
# 1. SAÍDA ESPERADA — extração de nota fiscal. Comparação campo a campo.
{"id": "NF-021", "origem": "producao",
 "entrada": "nota_fiscal_0457.pdf",
 "esperado": {"cnpj": "12.345.678/0001-90", "valor": 138.00, "data": "2026-08-14"},
 "por_que_existe": "valor lido como 1.380,00: o ponto de milhar foi tomado por decimal"}

# 2. RÓTULO — triagem de chamados. Uma classe entre poucas; mede-se a matriz de confusão.
{"id": "TR-108", "origem": "desenvolvimento",
 "entrada": "Fui cobrado duas vezes pela mesma assinatura este mês.",
 "esperado": {"rotulo": "financeiro"},
 "por_que_existe": "a palavra 'assinatura' puxava a classificação para 'conta'"}

# 3. RUBRICA — resposta de atendimento. Não há texto certo; há critérios, cada um sim ou não.
{"id": "AT-004", "origem": "sintetico", "revisor": "coordenacao_atendimento",
 "entrada": "O produto chegou quebrado e eu quero meu dinheiro de volta hoje.",
 "esperado": {"rubrica": [
     "informa o reembolso em até 10 dias úteis após a devolução",
     "não promete o dinheiro de volta no mesmo dia",
     "diz qual é o próximo passo do cliente"]},
 "por_que_existe": "pressão de prazo induz a prometer o que a loja não cumpre"}

# 4. CONTEXTO DE REFERÊNCIA — RAG sobre o regulamento. O gabarito é o trecho que
#    sustenta a resposta: avalia-se a busca (recall) e a fidelidade a ele.
{"id": "RG-033", "origem": "producao",
 "entrada": "Posso juntar o acréscimo de capital com o de viagem internacional?",
 "esperado": {"contexto": ["art-19"], "afirmacao_central": "vedada a cumulação"},
 "por_que_existe": "a resposta citou o art. 12 e afirmou que os acréscimos somam"}
```

A forma do "esperado" decide o nível de avaliação. Saída esperada e rótulo se comparam por código. Contexto de referência se mede por código na busca e por juiz na fidelidade. Rubrica vai ao juiz, e é a nota 03 que trata de como validá-lo. Em todos os casos, `origem` e `por_que_existe` continuam lá: o formato da referência muda, mas a exigência de saber de onde veio o caso não muda.

### Exemplo 5 — A média que esconde

```
                          casos   acertos    taxa
  categoria simples          72        70     97%
  exceção internacional       8         2     25%
  recusa (sem resposta)      12         4     33%
  ─────────────────────────────────────────────────
  MÉDIA                      92        76     83%
```

Oitenta e três por cento é um número aceitável e é uma descrição ruim deste sistema. É a mesma lição da "pergunta que falha" da Aula 06, agora com consequência de decisão: relatar por fatia, não só o agregado.

---

## Fontes e leituras

YAO, S. et al. **τ-bench**: a benchmark for tool-agent-user interaction in real-world domains. arXiv:2406.12045, 2024.

LANGFUSE. **Golden dataset evaluation**: build and maintain LLM test sets. 2026. Disponível em: <https://langfuse.com/resources/engineering/golden-dataset-evaluation>.

ES, S. et al. **RAGAS**: automated evaluation of retrieval augmented generation. arXiv:2309.15217, 2023.

ANTHROPIC. **Building effective agents**. 2024.

**Material da disciplina.** Aula 02, [nota 04](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/04-custo-latencia-e-decisao.md) — a média que esconde a variância. Aula 03, [nota 03](../../aula-03-prompt-engineering/notas-de-aula/03-prompt-como-codigo.md) — prompt como código e o teste de regressão pequeno. Aula 06, [nota 02](../../aula-06-embeddings-e-rag/notas-de-aula/02-chunking-e-a-medida-da-busca.md) — o conjunto de perguntas com resposta conhecida. Aula 08, [nota 04](../../aula-08-memoria/notas-de-aula/04-esquecer-e-proteger.md) — memória e não reprodutibilidade.
