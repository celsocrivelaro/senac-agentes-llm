# IA Aplicada com LLMs — Aula 08: Memória — Esquecer e proteger

## Introdução

As duas notas anteriores trataram do que a memória é e de como se escreve nela. Esta trata do que sai — e do que ocorre quando alguém que não deveria consegue ler o que ficou.

A primeira metade cobre um assunto que a literatura sobre agentes quase sempre omite. Sistemas de memória são apresentados como acumuladores, e a métrica implícita é quanto retêm. Um sistema que apenas acumula, porém, degrada de três maneiras distintas, e as três exigem tratamento próprio: fatos que são contraditos, fatos que envelhecem, e fatos cujo titular exige remoção.

A segunda metade nomeia o risco e encerra. A Aula 04 (nota 01, §10.2) já havia documentado o caso; aqui ele é retomado com o vocabulário técnico que as três aulas da fase construíram, e entregue à Aula 14.

> **Pré-requisitos:** notas [02](02-checkpoint-nao-e-memoria.md) e [03](03-escrever-e-ler.md) desta aula · Aula 07, [nota 02 §5](../../aula-07-rag-e-documentos/notas-de-aula/02-o-que-o-embedding-nao-ve.md) · Aula 04, [nota 01 §10](../../aula-04-casos-de-uso-e-escolha-do-projeto/notas-de-aula/01-o-que-e-um-agente.md).
>
> **Código:** [`04-esquecer-contradicao.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula08-memoria/04-esquecer-contradicao.py), [`04-esquecer-decaimento.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula08-memoria/04-esquecer-decaimento.py), [`04-esquecer-remocao.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula08-memoria/04-esquecer-remocao.py) e [`05-o-ganho-em-passos.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula08-memoria/05-o-ganho-em-passos.py).

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Demonstrar** que a recuperação por similaridade não desempata fatos contraditórios datados.
- **Implementar** desempate por carimbo de tempo em código determinístico.
- **Distinguir** decaimento de remoção, e justificar por que o primeiro não é apagamento.
- **Verificar** que uma remoção alcançou todas as estruturas de memória.
- **Caracterizar** a memória como superfície de ataque e como vetor de ataque.

---

## Desenvolvimento teórico

As duas primeiras seções tratam do que a literatura chama de *consolidation*: a manutenção da memória de **longo prazo** depois de escrita ([nota 02](02-checkpoint-nao-e-memoria.md), §2.1). A terceira é de outra natureza — obrigação legal, e não refinamento.

### 1. Contradição

Este é o experimento central da aula, e ele colhe uma semente plantada duas semanas antes.

Considere dois fatos, ambos verdadeiros quando registrados:

| Data | Fato |
|---|---|
| 2026-03-01 | "O teto de reembolso para refeição em viagem nacional é de R$ 120,00 por pessoa." |
| 2026-08-01 | "O teto de reembolso para refeição em viagem nacional passou a ser de R$ 150,00 por pessoa." |

O segundo revoga o primeiro, e **nada no texto de nenhum dos dois declara isso**. A informação de qual é posterior está na data, e a data não participa da comparação por similaridade.

Diante da consulta *"qual o teto de refeição em viagem nacional?"*, a recuperação ordena por proximidade semântica. O resultado observado tende a favorecer o fato de março, e por um motivo linguístico: a pergunta usa o presente do indicativo — *"qual **é** o teto"* — e o fato de março é redigido no presente, enquanto o de agosto é redigido no pretérito (*"passou a ser"*).

> A ordenação foi determinada pelo **tempo verbal**, não pela **data**.

É a cegueira de tempo da Aula 07 ([nota 02](../../aula-07-rag-e-documentos/notas-de-aula/02-o-que-o-embedding-nao-ve.md), §5), e ela custa aqui o que não custava lá: naquela nota era uma medição sobre um par de textos; aqui é a resposta que o agente dá a um usuário.

#### 1.1 A correção

Duas decisões, e nenhuma envolve o modelo.

**A data vai como metadado, não como texto.** Data em texto é para o modelo ler; data em metadado é para o código comparar. É a mesma escolha que a [nota 02](02-checkpoint-nao-e-memoria.md), §3.1, aplicou a `funcionario` e `veredito`.

**O desempate é uma linha:**

```python
def mais_recente(candidatos: list[dict], campo: str = "data") -> dict | None:
    return max(candidatos, key=lambda c: c[campo]) if candidatos else None
```

Determinística, testável, sem custo. Delegar o desempate ao modelo é o antipadrão da disciplina: ele erra, cobra uma chamada e não é verificável.

É a mesma conclusão que a Aula 05 estabeleceu para o roteador e a Aula 06 para as cegueiras — **regra determinística onde ela existe**. A terceira ocorrência do mesmo princípio em três aulas consecutivas não é repetição didática: é a constatação de que a fronteira entre o que cabe ao modelo e o que cabe ao código reaparece em cada componente novo.

#### 1.2 O descarte é registrado

```python
def desempatar_por_tempo(recuperados: list[dict]) -> dict:
    vencedor = mais_recente(recuperados)
    return {"vigente": vencedor,
            "descartados": [r for r in recuperados if r is not vencedor]}
```

A função devolve o que foi descartado, e não apenas o vencedor. A razão é a mesma que levou a Aula 05 ([nota 02](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md), §2) a registrar o motivo de término: **um agente que silenciosamente ignora um fato contraditório é indistinguível de um que nunca o teve**.

Quando a decisão for auditada — e decisões sobre reembolso são auditadas —, a diferença entre "o sistema não sabia do teto antigo" e "o sistema sabia e o descartou por ser anterior" é a diferença entre um defeito e um comportamento correto.

#### 1.3 O caso que o carimbo não resolve

Desempate por tempo trata o caso em que um fato **revoga** outro. Não trata o caso em que os dois permanecem válidos em condições distintas.

O teto nacional de R$ 120,00 e o teto internacional de R$ 260,00 não se contradizem: aplicam-se a situações diferentes. Um desempate por data entre eles produziria a resposta errada com a mesma confiança.

Para esse caso, o carimbo é insuficiente: é necessário armazenar a **condição de aplicação** junto com o fato. E o problema é reconhecível — é o mesmo da despesa de Lisboa da Aula 07 ([nota 03](../../aula-07-rag-e-documentos/notas-de-aula/03-medir-o-rag-e-a-recuperacao-como-ferramenta.md), §3), com a mesma solução: a consulta precisa carregar o critério íntegro, e não apenas o assunto.

---

### 2. Decaimento

Um fato que ninguém contradisse pode simplesmente ter envelhecido. Uma preferência registrada há dois anos, um destino frequente que deixou de ser frequente, uma regra procedural extraída de um processo que mudou.

O tratamento **não é apagar por idade**. Um episódio de sete meses atrás pode ser o único relevante para a pergunta corrente — no laboratório, a execução de Lisboa de março é exatamente esse caso.

A implementação usual pondera a distância pela recência no momento da recuperação. PARK et al. (2023) formalizam a combinação de três fatores — recência, importância e relevância — numa pontuação única, e é a formulação mais citada do tema. O laboratório a deixa como desafio opcional, com a recuperação por similaridade pura como linha de base para comparação.

O que se deve evitar está claro:

| Prática | Efeito |
|---|---|
| Apagar por idade | descarta o episódio raro e relevante |
| **Rebaixar por idade** | preserva o episódio e reduz a chance de ele dominar |
| Nada | a memória cresce sem limite e o antigo compete com o recente |

---

### 3. Remoção sob solicitação

O titular dos dados requer o apagamento, e o sistema precisa atendê-lo. É obrigação legal sob a LGPD, e é o requisito que distingue afirmar de cumprir.

O ponto técnico é que o dado está em **mais de uma estrutura**, e a remoção precisa alcançar todas:

```python
removidos_ep  = episodica.esquecer_funcionario(CODIGO_FUNCIONARIO)  # índice vetorial
removidos_sem = semantica.esquecer(CODIGO_FUNCIONARIO)              # chave-valor
# procedural: varredura por conteúdo, ver §3.3
```

> **Um índice vetorial do qual não se sabe remover uma pessoa é passivo jurídico.** A capacidade de remoção precisa ser projetada junto com o índice, e não descoberta quando a solicitação chegar.

#### 3.1 A quarta estrutura, que não é memória

O trecho acima está correto sobre o que faz e **errado sobre o resultado**, e a diferença entre as duas coisas é o assunto desta seção.

As três memórias não são os três únicos lugares onde o dado do titular caiu. O *checkpoint* da [nota 01](01-o-agente-e-o-estado.md), §8, guarda os argumentos de cada passo — e o passo 0 da execução analisada foi `consultar_historico({"funcionario": "F-088"})`. É dado pessoal, num arquivo que ninguém classificou como memória, e exatamente por isso ele escapa.

O [`04-esquecer-remocao.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula08-memoria/04-esquecer-remocao.py) **falha de propósito** na primeira verificação:

```
  removidos: 4 episódios, 2 fatos
  "Apaguei das três memórias." É o ponto em que a maioria para.

  A PROVA — verificar, não afirmar
  FALHOU — 1 vestígio(s) de F-088:
      [checkpoint] exec-0a1f.json
```

> **A lista de lugares a varrer não se deriva da taxonomia.** Ela se deriva de um inventário de **onde o dado cai** — e esse inventário cresce toda vez que alguém acrescenta uma estrutura ao sistema. Varrer "as três memórias" é varrer o modelo mental, não o sistema.

A varredura do *checkpoint* é ainda **textual**, e não por chave: o identificador aparece dentro de `argumentos`, de `objetivo` e de `resposta`, em profundidades diferentes do mesmo documento. Uma busca por campo não o encontraria em todos.

E a remoção é do **arquivo inteiro**, não do campo: um *checkpoint* com o identificador apagado não retoma mais nada, e o que restaria seria um arquivo inútil ocupando espaço.

É a razão pela qual a [nota 02](02-checkpoint-nao-e-memoria.md), §3.1, registra `funcionario` como metadado. Sem esse campo, a remoção de um titular exigiria varredura textual sobre todos os documentos — que é lenta, incompleta e não auditável.

#### 3.2 A verificação é o requisito

```python
vestigios = []
restantes = episodica.recuperar(f"despesas do funcionário {CODIGO_FUNCIONARIO}", k=5)
for r in restantes:
    if (r.get("funcionario") == CODIGO_FUNCIONARIO
            or CODIGO_FUNCIONARIO in r["resumo"]):
        vestigios.append(("episodica", r["id"]))
```

A chamada de exclusão não é a entrega. A entrega é a **busca posterior que não encontra nada**.

A distinção é a mesma que a Aula 05 fez entre executar e registrar: *"apaguei"* é afirmação; *"consultei as quatro estruturas e não localizei"* é prova. E a verificação precisa buscar por dois critérios — o metadado e o **conteúdo textual** —, porque um identificador pode aparecer no corpo de um resumo sem constar do metadado correspondente.

#### 3.3 A memória procedural escapa

A regra *"identificadores de funcionário têm o formato F seguido de três dígitos"* foi extraída de um erro cometido pelo F-088 e **não menciona o F-088**.

Ela não é dado pessoal, e portanto não é objeto da solicitação. Mas o caso ilustra o problema estrutural: se uma regra procedural contivesse um identificador, uma remoção por chave não a alcançaria — e nem a varredura textual da §3.1, porque ninguém pensa em varrer o *system prompt*. Memória procedural é texto livre, e texto livre exige varredura deliberada.

É a terceira razão para manter a lista de regras curta e sob revisão humana — as duas primeiras estão na [nota 03](03-escrever-e-ler.md), §1.3 e §3.

---

### 4. A memória é o alvo

A Aula 04 ([nota 01](../../aula-04-casos-de-uso-e-escolha-do-projeto/notas-de-aula/01-o-que-e-um-agente.md), §10.2) documentou o caso: mais de trinta mil instâncias de assistentes pessoais locais expostas na internet aberta, em boa parte por configuração de rede incorreta. Quem alcançava uma dessas instâncias podia emitir comandos ao agente, extrair as credenciais armazenadas na máquina e **ler a memória e o histórico de conversa**.

Duas propriedades tornam a memória o componente mais sensível de um sistema com agentes:

**É acumulada.** Uma execução isolada expõe uma tarefa. A memória expõe o que o usuário disse ao longo de meses — e a agregação tem valor que nenhuma execução isolada tem.

**Não é supervisionada.** Ninguém revisou o que entrou nos últimos seis meses. Diferentemente de um banco de dados de negócio, cujo esquema é projetado e cujo conteúdo é auditado, a memória de um agente recebe o que a política de escrita da [nota 03](03-escrever-e-ler.md) determinar — e o que essa política captura não é integralmente previsível.

#### 4.1 Vetor, além de alvo

A caracterização acima trata a memória como o que se protege. Há a caracterização inversa, e ela é menos evidente.

Conteúdo escrito na memória é **lido semanas depois, em outro contexto**, sem que nenhum mecanismo relacione as duas ocorrências. Um agente que extraia da execução um "procedimento aprendido" e o insira no *system prompt* das execuções seguintes construiu um canal por onde uma instrução pode ser injetada hoje e executada indefinidamente.

É injeção de prompt com persistência, e ela difere da injeção convencional em dois pontos: o intervalo entre entrada e execução, e o fato de a instrução passar a integrar o *system prompt* — a posição de maior autoridade no contexto.

A defesa segue o princípio que a Aula 05 ([nota 01](../../aula-08-memoria/notas-de-aula/01-o-agente-e-o-estado.md), §6) enunciou: **restrição por arquitetura vence restrição por prompt**. Conteúdo proveniente de fonte externa não deve alimentar memória procedural sem verificação, e a verificação é código, não instrução.

O tratamento completo — OWASP Agentic Security Initiative, *tool misuse*, isolamento — é objeto da Aula 14. Esta nota nomeia a superfície e para.

---

### 5. O que a fase entrega

A lista da Aula 03 (nota 04, §9), ao fim desta aula:

| Pendência | Situação |
|---|---|
| ~~outros padrões de arquitetura~~ | encerrada — Aula 05 |
| ~~estado, no interior de uma execução~~ | encerrada — Aula 05 |
| ~~confiabilidade~~ | encerrada — Aula 05 |
| ~~context engineering dinâmica~~ | encerrada — Aula 05 |
| ~~memória entre execuções~~ | **encerrada — esta aula** |
| **MCP** | Aula 10 |
| **multiagente** (coordenação) | Aula 12 |

E uma pendência nova, criada por esta aula: **memória torna o agente não reprodutível**. A mesma entrada, no mesmo modelo, com os mesmos parâmetros, produz saída diferente amanhã — porque a memória mudou. É a primeira vez no curso em que isso ocorre, e é um problema de avaliação, não de implementação: a Aula 11 parte dele.

---

## Exemplos

### Exemplo 1 — A contradição, medida

```
  pergunta: "Qual o teto de refeição em viagem nacional?"

  1º  [2026-03-01]  distância 0.2841
      O teto de reembolso para refeição em viagem nacional é de R$ 120,00
      por pessoa.
  2º  [2026-08-01]  distância 0.3122
      O teto de reembolso para refeição em viagem nacional passou a ser de
      R$ 150,00 por pessoa.
```

*(Valores ilustrativos; a execução do `04-esquecer-contradicao.py` produz os do modelo em uso.)*

A diferença de distância é de 0,028 — desprezível como sinal, e suficiente para inverter a ordem. O primeiro colocado está revogado desde agosto.

Caso a execução do laboratório produza a ordem inversa, o resultado permanece informativo: a margem estreita significa que uma reformulação da pergunta inverte o resultado. O que a demonstração estabelece não é que o vetor erre sistematicamente, e sim que **ele não está medindo o que decide o caso**.

Aplicado o desempate:

```
  vigente:     [2026-08-01] ... R$ 150,00 por pessoa
  descartados: ['2026-03-01']
```

### Exemplo 2 — A remoção, verificada

```
  ANTES:  episódica=10 · semântica=2 fatos sobre F-088 · procedural=1 · checkpoints=1
  DEPOIS: episódica=6 · semântica=0 · procedural=1
  removidos: 4 episódios, 2 fatos

  "Apaguei das três memórias." É o ponto em que a maioria para.

  A PROVA — verificar, não afirmar
  FALHOU — 1 vestígio(s) de F-088:
      [checkpoint] exec-0a1f.json

  A CORREÇÃO, E A SEGUNDA PASSADA
  removidos 1 checkpoint(s) que mencionavam o titular.
  OK — nenhum vestígio de F-088 nas quatro estruturas.
```

O `FALHOU` é o exemplo. A remoção estava correta sobre o que fez — as três memórias foram apagadas — e errada sobre o resultado, porque a lista de lugares a varrer veio da taxonomia em vez de vir do sistema.

A regra procedural permanece, e corretamente: ela não menciona o titular. A verificação a inspecionou por conteúdo e não a encontrou — que é o comportamento esperado, e distinto de não tê-la inspecionado.

### Exemplo 3 — A economia, em passos

```
  SEM memória, a segunda execução da mesma tarefa:
      passo 0  consultar_historico(F-88)   -> erro de formato
      passo 1  consultar_historico(F-088)  -> ok
      passo 2  consultar_politica(refeicao)
      passo 3  registrar_parecer
      = 4 passos

  COM memória:
      [2026-08-11] Refeição de R$ 240,00 por pessoa em Lisboa. Art. 4º §2º.
      [regra] Ids de funcionário têm o formato F seguido de três dígitos.
      = 2 passos
```

Dois passos economizados em quatro, na segunda execução da mesma tarefa.

A medida importa mais do que o número. Memória é infraestrutura com custo — armazenamento que cresce sem teto, tokens no contexto de toda execução, e o risco caracterizado na §4. Justificá-la exige a contrapartida em unidade comparável, e passos economizados é a unidade que quem paga a conta reconhece.

É a mesma disciplina que a Aula 05 impôs ao orçamento e a Aula 06 impôs ao `recall@k`: **mediu em vez de opinar**.

---

## Fontes e leituras

PARK, J. S. et al. **Generative agents**: interactive simulacra of human behavior. arXiv:2304.03442, 2023. (A combinação de recência, importância e relevância discutida na §2.)

PACKER, C. et al. **MemGPT**: towards LLMs as operating systems. arXiv:2310.08560, 2023.

OWASP. **Agentic Security Initiative**. (A superfície caracterizada na §4; o tratamento é objeto da Aula 14.)

BRASIL. **Lei nº 13.709, de 14 de agosto de 2018**: Lei Geral de Proteção de Dados Pessoais. (Art. 18, que fundamenta a remoção sob solicitação da §3.)

**Material da disciplina.** Aula 04, [nota 01 §10.2](../../aula-04-casos-de-uso-e-escolha-do-projeto/notas-de-aula/01-o-que-e-um-agente.md) — o caso das instâncias expostas. Aula 05, [nota 01 §6](../../aula-08-memoria/notas-de-aula/01-o-agente-e-o-estado.md) — restrição por arquitetura; [nota 02 §2](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md) — o registro do motivo. Aula 07, [nota 02 §5](../../aula-07-rag-e-documentos/notas-de-aula/02-o-que-o-embedding-nao-ve.md) — a cegueira de tempo. Aula 07, [nota 03 §3](../../aula-07-rag-e-documentos/notas-de-aula/03-medir-o-rag-e-a-recuperacao-como-ferramenta.md) — a condição de aplicação que o carimbo não substitui.
