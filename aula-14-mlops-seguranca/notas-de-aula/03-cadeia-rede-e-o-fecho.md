# IA Aplicada com LLMs — Aula 14: Segurança — Cadeia, imagem, rede, e o fecho

---

## Objetivos de aprendizagem

Ao final desta nota, espera-se que o aluno seja capaz de:

- **Avaliar** a superfície de cadeia de suprimentos, de imagem e de rede de um sistema implantado.
- **Justificar** por que a maioria dos incidentes de escala ocorre nessa camada.
- **Ler** um `Dockerfile` como superfície, identificando imagem base sem versão fixada, processo como `root`, porta exposta além da necessária e segredo em camada.
- **Aplicar** as práticas de curadoria de dependência à imagem base, a servidores, a ferramentas de terceiros e a pacotes sugeridos pelo modelo.
- **Identificar** o deputado confuso num agente com credencial de serviço, e substituí-la por credencial delegada.
- **Construir** uma suíte de casos hostis a partir da máquina de avaliação da aula de *evals*.
- **Declarar** um risco aceito, com a razão.

---

## Desenvolvimento teórico

### 1. O caso de encerramento

A [nota sobre o que é um agente](../../aula-04-casos-de-uso-e-escolha-do-projeto/notas-de-aula/01-o-que-e-um-agente.md) (§10.2) abriu este caso, e a aula de memória o retomou ao tratar da memória como superfície. Aqui ele fecha, com o vocabulário completo.

O **OpenClaw**, *runtime* de agente auto-hospedado e de código aberto descrito na aula de casos de uso, teve adoção explosiva e, no início de 2026, tornou-se o primeiro incidente de escala da área. Os números públicos:

- **dezenas de milhares de instâncias expostas** na internet pública, em dezenas de países — levantamentos independentes divergem entre quarenta e cento e trinta mil, e a divergência decorre do método de varredura;
- **mais de 90% delas** com falha de autenticação, em um dos levantamentos;
- **milhares exploráveis por execução remota de código**;
- **varredura por atacantes iniciada no mesmo dia** do anúncio público do projeto;
- **centenas de extensões maliciosas** publicadas no repositório de extensões — mais de um décimo do catálogo —, distribuindo software de roubo de credenciais.

> As aulas de casos de uso e de memória citam "mais de trinta mil instâncias"; a cifra corrente é de ordem maior. A diferença não é erro de uma das fontes: o número cresceu entre duas aulas do mesmo semestre, e esse crescimento é, ele próprio, um dado sobre a velocidade com que uma superfície exposta é encontrada.

### 2. A leitura, e por que ela determina a ordem do curso

Nada do que está acima é defeito de arquitetura de agente. É:

- **porta aberta** — a interface escutava em todas as interfaces de rede em vez de apenas no laço local;
- **padrão inseguro** — a autenticação vinha desligada por omissão, e a documentação recomendava ligá-la sem exigir;
- **registro sem curadoria** — qualquer pessoa publicava extensão, e o agente as instalava.

Os três são assunto de **deploy** e de **dependência** — a aula de produção. E é isso que justifica segurança vir **depois** de produção:

> Ensinar defesa arquitetural antes de existir uma superfície publicada seria defender o flanco errado. E descrever a superfície antes que ela exista seria descrever o vazio.

Há um segundo efeito, mais incômodo e mais útil: **um sistema pode ter arquitetura impecável e cair por um padrão de configuração**. A nota anterior não é suficiente, e dizê-lo é honesto.

### 3. A imagem, relida como superfície

A aula de produção empacotou o agente numa imagem Docker e escreveu o `Dockerfile` com critérios de reprodutibilidade, de tempo de build e de segredo fora da imagem. O mesmo arquivo, relido, é uma **declaração de superfície**: cada instrução decide o que a imagem herda de terceiros, com que privilégio o processo roda, o que fica acessível pela rede e o que fica gravado nas camadas. Lido por quem ataca, o `Dockerfile` responde quatro perguntas.

| Instrução | Defeito | Consequência | Risco |
|---|---|---|---|
| `FROM` | imagem base sem versão fixada | a base muda sem que o `Dockerfile` mude, e a imagem herda a vulnerabilidade de quem a publicou | ASI04 |
| ausência de `USER` | processo como `root` | qualquer execução de código obtida no processo — a alternativa por execução de código da aula de MCP, ou uma ferramenta induzida por injeção — passa a ter controle do container | ASI05, ASI03 |
| `EXPOSE` e publicação de porta | porta exposta além da necessária | um depurador, um painel ou uma interface administrativa ficam alcançáveis pela mesma rede que o serviço | o caso da §1 |
| `COPY` de arquivo com segredo | segredo copiado para uma camada | a camada é imutável: um `RUN rm` posterior cria outra camada e não apaga a anterior, e `docker history` ou a extração da imagem recuperam o arquivo | ASI03 |

Três observações precisam acompanhar a tabela.

**Versão fixada é decisão de atualização, e não congelamento.** Uma *tag* como `python:3.11-slim` é um ponteiro que o publicador move; a forma estrita de fixar é o *digest* (`python:3.11-slim@sha256:...`). Fixar não torna a base segura — torna a atualização **um ato deliberado**, que passa por reconstrução, varredura de vulnerabilidades e pela suíte, em vez de um efeito colateral do próximo build.

**`0.0.0.0` dentro do container não é o defeito do caso.** O processo no container precisa escutar em todas as interfaces **do container** para que a plataforma o alcance; a fronteira está fora dele, na porta que a plataforma ou o `docker run -p` publicam. O defeito do caso de encerramento é o equivalente no hospedeiro: `-p 8000:8000` publica em todas as interfaces da máquina, e `-p 127.0.0.1:8000:8000` restringe ao laço local.

**A correção do segredo é a mesma da aula de produção: revogar a chave.** Reconstruir a imagem não alcança as cópias já distribuídas, do mesmo modo que reescrever o histórico do git não alcança os clones. A prevenção é o `.dockerignore` e o segredo injetado em tempo de execução, por variável de ambiente da plataforma.

O caso de encerramento é o mesmo erro em escala: **padrão inseguro aceito sem leitura**. A diferença entre a instância exposta do caso e o serviço construído no curso não é de natureza, e sim de quantas linhas do `Dockerfile` foram lidas como superfície.

### 4. MCP, tratado

A aula de MCP nomeou três ataques e parou. Eles compartilham um mecanismo único, e nomeá-lo torna os três compreensíveis de uma vez:

> **A descrição da ferramenta entra no contexto do modelo. Portanto, a descrição é entrada não confiável vinda de terceiro.**

| Ataque | Mecanismo |
|---|---|
| ***tool poisoning*** | instruções embutidas na descrição |
| ***rug pull*** | a descrição muda **depois** da aprovação |
| ***tool shadowing*** | um servidor altera, pela própria descrição, o comportamento do agente em relação a ferramentas de **outro** servidor |

É **ASI04**, e a escala justifica a preocupação: ordem de dez mil servidores públicos, dos quais cerca de um oitavo atinge limiar alto de confiança em documentação, manutenção e confiabilidade (Zhao et al., 2025).

**As práticas**, e são as de gestão de dependência de sempre, com um acréscimo:

- **fixar a versão** da imagem base, do servidor e da ferramenta, como se fixa qualquer dependência;
- **revisar a descrição, e não só o código** — este é o acréscimo, e é o que quase ninguém faz;
- **verificar assinatura** quando existir. É o que propostas de identidade de ferramenta (Bhatt et al., 2025) endereçam no nível de MCP, e o que o *Agent Card* assinado do A2A resolve no nível de agente — controle que a aula de sistemas multiagente trata;
- **tratar toda declaração vinda de fora como entrada**, e não como configuração.

**A mesma cadeia, uma camada acima: *skills*.** As extensões do caso de encerramento (§1) não eram servidores MCP: eram ***skills*** — pacotes de instrução e código que o agente instala e passa a executar como comportamento próprio. A OWASP deu a essa camada um catálogo separado, o **Agentic Skills Top 10** (AST10), com a distinção que o justifica: o MCP é **como** o modelo fala com a ferramenta; a *skill* é **o que** ela faz. Os dez riscos:

| | Risco | Já visto como |
|---|---|---|
| **AST01** | *skill* maliciosa | as extensões do caso da §1 |
| **AST02** | comprometimento da cadeia de suprimentos | ASI04 |
| **AST03** | *skill* com privilégio excessivo | ASI02 e ASI03 |
| **AST04** | metadado inseguro | *tool poisoning* — a descrição é entrada |
| **AST05** | instrução externa não confiável | injeção indireta (nota 01, §3) |
| **AST06** | isolamento fraco | *sandbox* (nota 02) e o processo como `root` (§3) |
| **AST07** | deriva de atualização | *rug pull* — e a versão não fixada |
| **AST08** | varredura insuficiente | revisar a descrição, e não só o código |
| **AST09** | ausência de governança | o registro sem curadoria do caso |
| **AST10** | reuso entre plataformas | a mesma *skill* instalada em agentes diferentes, com permissões diferentes |

A coluna da direita é o ponto: **nenhum risco é novo**. É a lista de dependência de sempre aplicada a uma camada que também costuma ser instalada sem leitura — e as práticas acima valem para ela sem acréscimo.

> O projeto cita, para o primeiro trimestre de 2026, cerca de quatro mil *skills* analisadas com pouco mais de um terço apresentando falha de segurança, e uma única campanha com mais de mil *skills* maliciosas no repositório do caso da §1. São cifras de um levantamento em andamento, revistas a cada versão do catálogo.

**A dependência que o modelo inventa.** Há um caminho para a cadeia de suprimentos que não passa por registro algum: o modelo, ao gerar código, sugere importar um pacote **que não existe**. O nome é plausível, e é recorrente — o mesmo modelo tende a alucinar o mesmo nome para o mesmo pedido. Um atacante que registra esse nome no índice público de pacotes recebe a instalação de quem executou o código sem conferir. A prática tem nome, ***slopsquatting***, e a medição que a sustenta encontrou, em média, ao menos 5,2% de pacotes inexistentes entre os sugeridos por modelos comerciais e 21,7% entre os sugeridos por modelos abertos (Spracklen et al., 2025). O catálogo da OWASP para aplicações de LLM a situa entre cadeia de suprimentos (LLM03) e confiança excessiva na saída do modelo (LLM09). A prática que a neutraliza é a mesma da lista acima: a dependência sugerida pelo modelo é **entrada**, e entra no projeto pelo mesmo crivo de qualquer outra — existência, procedência, versão fixada.

### 5. Com que credencial o agente age

O mapa da nota 01 deixou a pergunta do ASI03 em aberto: **o agente age com a credencial de quem?** A resposta mais comum em sistemas construídos às pressas é a pior: com uma credencial **de serviço**, criada uma vez para o agente e com o alcance somado de tudo o que ele algum dia precisou fazer.

O defeito que isso produz tem nome desde 1988. O **deputado confuso** (*confused deputy*) é o programa que detém uma autoridade e a exerce a pedido de quem não a detém (Hardy, 1988). O agente com credencial de serviço é o caso exato: qualquer usuário que fala com ele — e qualquer texto injetado que o redireciona — passa a dispor do alcance da credencial do agente, e não do próprio.

```
  CREDENCIAL DE SERVIÇO                    CREDENCIAL DELEGADA
  o agente lê todas as despesas            o agente lê as despesas que
  de todos os setores                      ESTE usuário pode ler

  usuário do setor A pede:                 usuário do setor A pede:
  "resuma as despesas do setor B"          "resuma as despesas do setor B"
  -> o agente consegue, e resume           -> a ferramenta nega: o usuário
                                              não tem acesso ao setor B
```

As três práticas decorrem da definição:

- **a autorização é verificada na ferramenta, sobre a identidade do usuário** — não pelo modelo, e não pelo prompt (nota 02, §8). O modelo pode pedir qualquer coisa; a ferramenta responde conforme quem está do outro lado;
- **a credencial é delegada, com escopo e validade curtos** — um *token* emitido para este usuário, esta tarefa e os próximos minutos, em vez de uma chave permanente com o alcance de todos;
- **ação em nome do sistema é separada de ação em nome do usuário** — quando o agente precisa de fato de uma autoridade própria (gravar o *trace*, por exemplo), ela é outra credencial, com o mínimo para aquela função.

A diferença entre as duas colunas do quadro não é de configuração: na da esquerda, a injeção herda o alcance do sistema inteiro; na da direita, herda o de um usuário. O dano que cabe numa injeção bem-sucedida é delimitado aqui, antes de qualquer padrão de projeto.

### 6. *Red team* como suíte

Testar defesa é a mesma máquina da aula de *evals*, com casos hostis em vez de casos representativos. Os scripts `04.1-red-team-sem-defesa.py`, `04.2-red-team-minimizacao.py` e `04.3-red-team-llm-duplo.py` rodam a mesma suíte contra três configurações e imprimem as duas taxas em colunas separadas. A estrutura é idêntica: conjunto versionado, execução repetida, relatório por fatia.

E o mesmo princípio de origem vale, com um ajuste: **cada tentativa bem-sucedida vira caso permanente** — exatamente como cada incidente de produção virava.

A suíte hostil passa a ter uma métrica própria, e ela **não se mistura** com a de utilidade:

| Suíte | Mede | Pergunta |
|---|---|---|
| representativa (*evals*) | utilidade | o sistema resolve? |
| **hostil** | resistência | o sistema obedece a quem não deveria? |

Reportá-las juntas esconde a troca que a nota 02 estabeleceu: uma defesa que sobe a resistência e derruba a utilidade precisa aparecer nas duas colunas.

E o desafio que fecha o exercício desta aula é justamente esse: **medir quanto de utilidade uma defesa custou**. É o dado que falta em quase toda discussão de segurança de agentes.

A suíte hostil não é descartada ao fim desta aula. Ela entra no **carimbo** — o registro, definido na aula de *evals*, dos campos que identificam uma configuração do sistema (prompt, modelo, parâmetros, índice, memória, versão de servidor) e das suítes que a validam — ao lado do *golden dataset*.

### 7. O que registrar, e o que fazer

**Registrar.** O *trace* das aulas de arquitetura de agentes e de produção é o que torna a investigação possível: sem trajetória gravada, sabe-se que houve dano e não se sabe por onde entrou. O que precisa estar lá: a origem de cada conteúdo que entrou no contexto, as ferramentas chamadas com argumentos, e as escritas com identificador.

**Fazer, quando se descobre.** Revogar credencial; e **invalidar a memória contaminada** — e aqui a aula de memória cobra a existência de uma operação de esquecimento seletivo. Se ela não existir, o sistema **não tem como se recuperar** de um envenenamento, e cada execução futura repete o comportamento injetado.

Essa é a razão adicional para que a operação de esquecimento exista, e ela não é de privacidade nem de custo: é de recuperação de incidente.

### 8. Aceitar risco é engenharia

Uma parte do trabalho que raramente é ensinada: **nem todo risco se mitiga**, e a decisão de aceitar um risco é legítima quando é **declarada**.

Um sistema pode manter as três pernas da trifecta porque cortá-las inviabiliza o produto. Pode consumir um servidor de terceiro sem assinatura porque não há alternativa. Pode operar sem sandbox porque o custo não se justifica no volume atual.

> **Aceitar risco declarado é engenharia. Não declarar é omissão.**

O que o registro precisa conter: qual é o risco, por que foi aceito, o que o limita hoje, e que evento faria reavaliar.

### 9. O fecho: a superfície, e o que fica para a aula de sistemas multiagente

O sistema construído em catorze encontros — que **raciocina** (*prompt engineering*, arquitetura de agentes), **recupera** (*embeddings* e RAG, RAG e documentos), **lembra** (memória), **se conecta** (MCP), **é medido** (*evals*) e **está no ar** (produção) — **é uma superfície de ataque**. A diferença em relação ao início desta aula é que agora ele tem uma suíte que diz **o quanto resiste**, ao lado da que diz o quanto acerta.

A afirmação não é uma advertência sobre agentes de IA. É uma descrição do que todo sistema em produção é, tornada concreta, linha por linha, no código escrito ao longo do curso — do documento recuperado ao `Dockerfile`.

**O que existe, e não foi tratado.** Os catálogos de referência desta aula cobrem um território maior que o sistema do curso. O que ficou de fora ficou por um critério: o sistema **consome um modelo por API e não o treina**, e os riscos abaixo pertencem a quem treina, hospeda ou opera o modelo em escala pública.

| Risco | Fonte | Por que fica fora |
|---|---|---|
| ataques ao modelo: exemplo adversarial, roubo do modelo, inferência de pertencimento (descobrir se um registro estava no treino), inversão (reconstruir dado de treino) | MITRE ATLAS | exigem acesso ao modelo ou aos seus pesos; são a superfície do fornecedor |
| envenenamento de dado de treino e de *fine-tuning*, *backdoor* no modelo | LLM04 | o sistema não treina; o envenenamento do corpus de RAG já é injeção (nota 01) |
| fraquezas de vetores e *embeddings*: índice compartilhado entre clientes, inversão do *embedding* | LLM08 | o índice do curso tem um único dono; passa a valer quando o mesmo índice serve a vários clientes |
| consumo sem limite: custo como ataque (*denial of wallet*), extração do modelo por consulta em massa | LLM10 | o teto de gasto da nota 02 contém o dano; limite por usuário e detecção de extração são operação de API pública |

A tabela não declara esses riscos irrelevantes: declara a condição em que cada um passa a ser do sistema. É o mesmo formato do risco aceito da §8, aplicado ao escopo da aula.

A lista de pendências da [nota sobre *tool calling*](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) (§9) tem um item de pé: **multiagente**. A aula de sistemas multiagente o risca, e recebe desta aula os dois riscos que foram nomeados aqui e não tratados:

- **ASI07, comunicação insegura entre agentes.** Hoje o endereço é um só: o resumo que o sub-agente de arquitetura de agentes devolve ao principal. Com vários agentes, cada *handoff* é uma fronteira em que o que atravessa é entrada não confiável para quem a recebe, e o *Agent Card* do A2A é a declaração que precisa ser verificada do lado do protocolo.
- **ASI08, falhas em cascata.** Hoje o endereço é a ferramenta que falha no meio da execução (arquitetura de agentes). Com vários agentes, o erro de um vira entrada do próximo, e a confiabilidade compõe multiplicativamente.

Dividir o agente em vários abre fronteiras novas, e aquela aula as trata como contrato e como superfície, com os padrões de defesa da nota 02 — o mapear-reduzir é a mesma topologia de um sistema multiagente, usada como parede.

---

## Exemplos

### Exemplo 1 — A superfície do caso, decomposta

```
  CAMADA                        DEFEITO                        DE QUEM É
  ─────────────────────────────────────────────────────────────────────
  rede           escutava em 0.0.0.0 em vez de 127.0.0.1       deploy (produção)
  autenticação   desligada por omissão                          padrão do produto
  dependência    registro de extensões sem curadoria            supply chain (MCP)
  execução       extensões com acesso ao sistema de arquivos    sandbox (nota 02, §5)
  ─────────────────────────────────────────────────────────────────────
  arquitetura de agente                                         SEM DEFEITO
```

A última linha é o argumento da §2.

### Exemplo 2 — O `Dockerfile`, lido como superfície

A versão com os quatro defeitos da §3, anotada:

```dockerfile
FROM python:3.11                    # [1] tag movida pelo publicador: a base muda sem que este arquivo mude
WORKDIR /app
COPY . .                            # [4] sem .dockerignore, o .env entra nesta camada
RUN pip install -r requirements.txt
RUN rm -f .env                      # [4] apaga numa camada nova; a do COPY conserva o arquivo
EXPOSE 8000 5678                    # [3] 5678 é a porta do depurador, exposta junto com a do serviço
CMD ["uvicorn", "servico:app", "--host", "0.0.0.0", "--port", "8000"]
                                    # [2] sem USER: o processo roda como root
```

E a versão com a superfície reduzida, no desenho da aula de produção e com o *digest* acrescentado à base:

```dockerfile
FROM python:3.11-slim@sha256:<digest>   # [1] base fixada; atualizar é decisão, não efeito colateral
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .                                # [4] .dockerignore exclui .env; o segredo chega por variável da plataforma
RUN useradd --create-home agente
USER agente                             # [2] execução de código obtida no processo não é root
EXPOSE 8000                             # [3] só a porta do serviço
CMD ["sh", "-c", "uvicorn servico:app --host 0.0.0.0 --port ${PORT:-8000}"]
```

```
O QUE A SEGUNDA VERSÃO DEIXOU DE CONSEGUIR FAZER:
  - receber correção da base sem um build deliberado;
  - instalar pacote ou escrever fora do diretório do usuário em execução;
  - ser depurada remotamente pela porta 5678.
```

A lista é a mesma exigência da nota 02: a redução de superfície também custa capacidade, e o custo é declarado.

### Exemplo 3 — A suíte hostil, ao lado da representativa

Quadro ilustrativo; os números medidos saem dos scripts `04.x`, para o modelo configurado no `.env`.

```
                         utilidade    resistência
  sem defesa                  87%            0%
  + minimização de contexto   87%           20%
  + confirmação inescapável   85%           60%
  + dual LLM no ramo de RAG   71%           90%
                              ^^^
                              o preço, e ele precisa aparecer
```

O último passo custou catorze pontos de utilidade (de 85% para 71%), e as defesas somadas, dezesseis. Pode valer a pena; a decisão exige o número.

### Exemplo 4 — Um risco aceito, declarado

```markdown
## RISCO-03 — trifecta letal presente

O agente lê pareceres (privado), recupera documentos do corpus (não
confiável) e envia e-mail de notificação (saída).

**Por que foi aceito:** a notificação por e-mail é requisito do produto;
sem ela, o fluxo de aprovação não fecha.

**O que o limita hoje:** o destinatário é restrito a uma lista fixa de
endereços internos, validada em código e não configurável por prompt.

**O que faria reavaliar:** qualquer pedido de destinatário dinâmico.
```

---

## Fontes e leituras

VAN HAMME, T. et al. **MATRA**: modeling the attack surface of agentic AI systems — OpenClaw case study. arXiv:2605.10763, 2026.

ZHAO, S. et al. **Parasites in the toolchain**: a large-scale analysis of attacks on the MCP ecosystem. arXiv:2509.06572, 2025.

BHATT, M.; NARAJALA, V. S.; HABLER, I. **ETDI**: mitigating tool squatting and rug pull attacks in Model Context Protocol (MCP). arXiv:2506.01333, 2025.

INVARIANT LABS. **MCP Security Notification**: tool poisoning attacks. 2025.

SPRACKLEN, J. et al. **We have a package for you!** A comprehensive analysis of package hallucinations by code generating LLMs. In: USENIX SECURITY SYMPOSIUM, 34., 2025. Disponível em: https://arxiv.org/abs/2406.10279. Acesso em: 21 out. 2026.

HARDY, N. **The confused deputy**: (or why capabilities might have been invented). ACM SIGOPS Operating Systems Review, v. 22, n. 4, p. 36-38, 1988.

MITRE. **ATLAS**: Adversarial Threat Landscape for AI Systems. Disponível em: https://atlas.mitre.org/. Acesso em: 21 out. 2026.

DOCKER. **Dockerfile reference**. Docker Documentation.

DOCKER. **Building best practices**. Docker Documentation.

OWASP. **Agentic Skills Top 10** (AST10). Versão 1.0, 2026. Disponível em: https://owasp.org/projects/agentic-skills-top-10.

OWASP GENAI SECURITY PROJECT. **OWASP top 10 for agentic applications for 2026**. [S. l.]: OWASP Foundation, 9 dez. 2025. Disponível em: https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications/. Acesso em: 7 out. 2026.

OWASP GENAI SECURITY PROJECT. **OWASP top 10 for LLM applications 2025**. [S. l.]: OWASP Foundation, 2024. Disponível em: https://genai.owasp.org/llm-top-10/. Acesso em: 20 out. 2026.

**Material da disciplina.** [O que é um agente, e onde eles estão](../../aula-04-casos-de-uso-e-escolha-do-projeto/notas-de-aula/01-o-que-e-um-agente.md) — o caso, aberto ali. [Esquecer e proteger](../../aula-08-memoria/notas-de-aula/04-esquecer-e-proteger.md), em memória — o esquecimento seletivo, aqui exigido como recuperação de incidente. [O preço do protocolo](../../aula-10-mcp/notas-de-aula/03-o-preco-do-protocolo.md), em MCP — os três ataques, agora tratados. [De *script* a imagem](../../aula-13-mlops-empacotamento/notas-de-aula/03-de-script-a-imagem.md), em produção — o `Dockerfile` e a superfície criada. [O juiz, e a suíte](../../aula-11-evals/notas-de-aula/03-o-juiz-e-a-suite.md), em *evals* — a máquina de avaliação, aqui com casos hostis.
