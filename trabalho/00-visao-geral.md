# Trabalho da disciplina — Visão geral

## O que é este trabalho

Durante o semestre vocês vão construir **um sistema de agentes de IA para um problema real, escolhido por vocês**, e entregá-lo em **três partes**.

Não são três trabalhos. É **um sistema**, construído em camadas: cada parte acrescenta o que a disciplina acabou de ensinar, sobre o mesmo tema, no mesmo repositório. O que vocês escolherem na Parte 1 é o que vocês vão entregar na Parte 3 — muito maior, e muito mais bem feito.

A entrega final é condicionada a uma **apresentação em aula**, em que o grupo demonstra cada agente do sistema funcionando.

> **Importante:** este trabalho **é** o projeto final da disciplina. Não existe outro. A escolha do tema acontece na aula sobre casos de uso de agentes, e a ficha de case preenchida naquela aula é o núcleo da Parte 1.

---

## O que existe no final

Ao fim das três partes, o grupo tem:

- um sistema **multiagente** resolvendo um problema real, com mais de um agente e uma arquitetura de coordenação declarada;
- os agentes **conversando com software tradicional** — API, banco de dados, serviço interno —, via ferramentas próprias e via **MCP**;
- uma **base de memória consultável** pelos agentes (RAG);
- **memória entre execuções** — o que o sistema lembra de ontem, e o que ele deliberadamente não guarda;
- um **workflow orquestrado com LangGraph**;
- um **plano de prompt engineering** documentado e versionado;
- e as três preocupações que separam protótipo de sistema: **segurança**, **MLOps/observabilidade** e **gestão de custos**.

---

## As três entregas

| Parte | Tema central | O que entra |
|---|---|---|
| **1** | **Escolher e provar o terreno** | tema e contexto · **a justificativa de negócio** (por que agente, e que ganho) · análise de modelos · **um agente simples** com prompt engineering e arquitetura básica · integração simples com software tradicional (pode ser mock) |
| **2** | **O agente de verdade** | plano de prompt engineering · arquitetura do agente documentada · **RAG** como memória consultável · **memória** entre execuções · **MCP** no lugar da integração manual · **workflow com LangGraph** |
| **3** | **O sistema** | **multiagente** com arquitetura de coordenação · **segurança** · **MLOps** e observabilidade · **gestão de custos** · **apresentação em aula** |

Cada parte é avaliada **na entrega dela**. Uma Parte 1 fraca não é compensada por uma Parte 3 boa — mas uma Parte 1 bem escolhida torna as outras duas muito mais fáceis.

---

## Parte 1 — Escolher e provar o terreno

O enunciado completo está em **[01-primeira-entrega.md](01-primeira-entrega.md)**.

Em resumo: grupo formado, tema escolhido e detalhado, **a justificativa de negócio do sistema**, uma análise de modelos que justifique a escolha, e **um agente simples rodando** — com poucas ferramentas, prompt versionado e uma arquitetura que vocês saibam defender.

A justificativa de negócio é o campo novo, e é o que mais aproxima o trabalho da vida real: vocês vão ter que **vender o sistema** a quem não vai ler o código de vocês. Por que agente e não software comum, que ganho se espera — com linha de base medida, conta à vista e denominador —, o que muda para o usuário e o que a coisa custa para rodar.

O objetivo desta parte não é impressionar. É **descobrir cedo** se o tema escolhido é viável, enquanto ainda dá tempo de ajustar.

---

## Parte 2 — O agente de verdade

Aqui o agente simples da Parte 1 se transforma em um agente completo. Seis frentes, e **cada uma acrescenta uma peça ao sistema e cobra um número por ela**:

**1. O plano de prompt engineering.** Um documento que declara, para cada etapa do sistema: qual técnica de prompting é usada, **por quê**, qual o contrato de saída, e como aquela etapa é testada. Os prompts vivem em arquivo, versionados, com a combinação `prompt × modelo × parâmetros` registrada — como a disciplina cobra desde a aula de prompt engineering.

**2. A arquitetura do agente, documentada.** Diagrama e texto explicando: quais são as ferramentas, qual é o laço, onde está o estado, qual é o orçamento, quais são as condições de término e onde entra o humano. Vocês precisam conseguir defender **por que este nível de autonomia e não o de baixo**.

**3. RAG como memória consultável.** Uma base de conhecimento que os agentes consultam — não um chatbot sobre PDFs, mas a **memória do sistema**: o conhecimento de domínio que os agentes precisam para decidir. Vocês definem o que entra, como é indexado e como a resposta cita a fonte.

**4. Memória entre execuções.** O RAG consulta documentos que alguém escreveu; a memória guarda o que **o próprio sistema** viveu. As duas se confundem porque ambas recuperam por relevância, e distingui-las no seu case é parte da entrega — junto com a decisão que mais vale nota: **o que o sistema deliberadamente não guarda**.

**5. MCP no lugar da integração manual.** A integração simples da Parte 1 é reescrita como **servidor MCP**. O ganho a demonstrar: a ferramenta deixa de ser código acoplado ao seu agente e passa a ser um serviço que qualquer agente consome.

**6. Workflow com LangGraph.** A orquestração passa a usar **LangGraph**, do ecossistema LangChain — o modelo mental dele é grafo de estado, que é literalmente o que vocês construíram à mão na aula de arquitetura, e é o que torna a comparação possível. E, junto com ela, a pergunta que a disciplina insiste desde aquela aula: **o que o framework te deu, e o que ele te tirou?** Comparar a versão na mão com a versão em framework é parte da entrega — e concluir que não compensa portar, **com a contagem à vista**, é resultado válido.

---

## Parte 3 — O sistema

**1. Multiagente.** Mais de um agente, com arquitetura de coordenação **explícita e justificada**: quem chama quem, quem decide, como o resultado de um chega ao outro, e o que acontece quando um deles falha. Multiagente não é "vários prompts" — é uma decisão de arquitetura que precisa ser defendida contra a alternativa de um agente só.

**2. Segurança.** Prompt injection direto e indireto, tool misuse, dado sensível no contexto, permissão por ferramenta. O que o seu sistema faria se a entrada fosse hostil.

**3. MLOps e observabilidade.** O trace de cada execução, as métricas que vocês acompanham, o conjunto de avaliação, e como vocês sabem que uma mudança melhorou em vez de piorar.

**4. Gestão de custos — e a conferência da promessa.** Quanto custa uma execução, onde o custo mora, e as alavancas que vocês usaram para reduzi-lo. Com números medidos, não estimados.

E o fecho do arco do trabalho: **o ganho prometido na Parte 1, conferido.** Vocês voltam à venda que fizeram, colocam ao lado o número que o sistema realmente entregou e explicam a diferença. Prometer 80% e entregar 30%, com a conta à vista, é um resultado aceitável e honesto — e é o que acontece na maioria dos projetos reais. Prometer "mais eficiência" e não ter como conferir, não é.

**5. A apresentação.** Demonstração ao vivo, **agente por agente**. Cada integrante do grupo deve conseguir explicar qualquer parte do sistema — não apenas a que escreveu.

---

## Recomendações

**Podem usar o tema do TCC.** É recomendado, inclusive: o trabalho vira um capítulo de implementação, e vocês estudam o domínio uma vez em vez de duas. Se o TCC não tiver nada a ver com agentes, não force — um tema forçado atrapalha os dois.

**Escolham um caso com complexidade real de interação com o usuário.** Um sistema que recebe um formulário preenchido e devolve uma resposta não exercita quase nada. Os casos bons são aqueles em que o sistema precisa **descobrir** o que o usuário quer: perguntar o que falta, lidar com informação contraditória, decidir quando já sabe o suficiente.

**Podem usar um modelo diferente do Mistral.** A disciplina usa a API da Mistral nos laboratórios, mas o trabalho é de vocês. Se outro modelo servir melhor ao caso — por contexto maior, por suporte a visão, por custo, por rodar local —, usem. **Mas justifiquem na análise de modelos da Parte 1**, e considerem o custo: o grupo é responsável pelo que gastar.

**Podem ter multimídia.** Imagem, áudio, documento digitalizado. Se o caso pede, use — e escolha um modelo que suporte. Multimídia não vale ponto por si; vale quando o problema realmente precisa dela.

---

## Regras gerais

### Técnicas

- **Python**, e **a biblioteca `openai`** para falar com o modelo. É a mesma pilha dos laboratórios da disciplina: a biblioteca é da OpenAI, mas a `base_url` aponta para o provedor que vocês escolherem — Mistral, outro serviço compatível, ou um modelo local. Trocar de modelo é trocar duas variáveis de ambiente, e é por isso que a exigência não conflita com a recomendação de usar o modelo que servir melhor ao caso.
- **Chave de API nunca no repositório.** `.env` fora do git, `.env.example` dentro, com os nomes das variáveis e nenhum valor.
- **`requirements.txt`** com as dependências fixadas, e o projeto rodando do zero em menos de cinco minutos.
- **Instruções de uso no `README.md`** — não só de instalação. Ver abaixo.
- **Toda a pesquisa e documentação em `docs/`.** Ver abaixo.

### O `README.md` precisa responder duas coisas diferentes

Os grupos costumam escrever só a primeira, e a segunda é a que falta:

- **Como rodar** — clonar, instalar, configurar o `.env`, o comando que sobe o sistema. Testado do zero, por alguém que nunca viu o projeto.
- **Como usar** — o que a pessoa digita, o que o sistema faz com aquilo, que saída ela recebe e como interpretá-la. Inclui: um **exemplo completo de uso** (entrada e saída reais, copiadas de uma execução), o que o sistema **não** faz, e o que acontece quando ele não sabe responder.

Um sistema que roda e ninguém sabe usar não está entregue.

### `docs/` — a pesquisa do case fica versionada

**Todo documento gerado sobre o case vive em `docs/`**, no repositório, em Markdown. Isso inclui a caracterização do problema, a análise de modelos, a pesquisa de casos parecidos na indústria, as fontes consultadas, os registros de decisão e o que for produzido nas Partes 2 e 3.

Duas razões, e a segunda é a que importa: o material fica **junto do código que ele justifica**, e a pesquisa passa a ter **histórico** — dá para ver quando o grupo mudou de ideia sobre o próprio case, e por quê. É o mesmo princípio do versionamento de prompt cobrado desde a Aula 03, aplicado ao raciocínio.

### De processo

- **Grupos de até 4 alunos.** Grupos menores são permitidos; a exigência de profundidade é a mesma.
- **Um repositório Git por grupo**, com histórico de commits que mostre o trabalho de todos os integrantes.
- **Prompts em arquivo e versionados.** Prompt embutido no meio do código não é aceito a partir da Parte 2.
- **Nada roda em produção de terceiros.** Se o caso envolve um sistema real, o grupo trabalha contra **mock** ou contra ambiente próprio. Nenhum trabalho desta disciplina escreve em base de dados de empresa.
- **Dados sensíveis não entram no repositório**, nem no contexto dos modelos. Se o caso envolve dado pessoal, ele é simulado.
- **O tema pode ser refinado, não trocado.** Ajustes de escopo, ferramenta ou arquitetura são esperados e bem-vistos até a Parte 2. Começar de novo depois disso não dá tempo.

---

## O que não é aceito como tema

Quatro escolhas que reprovam antes de começar:

| Anti-padrão | Sintoma | Por quê |
|---|---|---|
| **Sem verificador** | vocês não conseguem dizer como saber que a saída está certa | sem verificação não há como avaliar, nem como o agente se corrigir |
| **Depende de dado que vocês não têm** | o plano contém "vou pedir acesso" | o acesso não chega, ou chega tarde demais |
| **Grande demais** | o tema não cabe em uma frase | vira um pouco de tudo e nada inteiro |
| **Clone de produto** | o esforço é interface e infraestrutura | vocês passam o semestre longe do assunto da disciplina |

A lista acima é curta de propósito, e é ela que vale como critério. O que está **por trás** dela são os casos da Aula 04: a [nota 01, §11](../aula-04-casos-de-uso-e-escolha-do-projeto/notas-de-aula/01-o-que-e-um-agente.md) mostra o que os casos que funcionam têm em comum — feedback verificável, escopo estreito, verificador construído —, e a [nota 02](../aula-04-casos-de-uso-e-escolha-do-projeto/notas-de-aula/02-os-casos-que-falharam.md) traz as autópsias de quem errou.

O lugar de aplicar esta lista ao seu tema é o [exercício 04](../aula-04-casos-de-uso-e-escolha-do-projeto/exercicios/exercicio_04.md), e é lá que ele é validado com o professor. Façam isso **antes** de fechar o tema.

---

## Como as aulas alimentam o trabalho

A partir da escolha do tema, cada aula nova termina com a mesma pergunta: **o que isto muda no seu trabalho?**

| Aula | O que ela entrega para o trabalho |
|---|---|
| Escolha e configuração de modelos | a análise de modelos da Parte 1 |
| Prompt engineering | o plano de prompt engineering e o versionamento |
| Arquitetura de agentes | o agente, o estado, o orçamento, as salvaguardas |
| Casos de uso de agentes | **a escolha do tema** e a ficha de case |
| Embeddings e RAG | a base de conhecimento consultável da Parte 2 |
| Memória | o que o sistema lembra entre execuções, na Parte 2 |
| MCP | a reescrita da integração na Parte 2 |
| Frameworks e orquestração | o workflow em LangGraph da Parte 2 |
| Multiagente | a arquitetura de coordenação da Parte 3 |
| Observabilidade e evals | o MLOps da Parte 3 |
| Segurança e custos | as duas últimas frentes da Parte 3 |

Ou seja: **vocês não precisam saber tudo hoje para escolher hoje.** Precisam escolher um tema que tenha espaço para tudo isso.
