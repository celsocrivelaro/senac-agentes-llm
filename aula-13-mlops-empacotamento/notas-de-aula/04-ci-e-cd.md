# IA Aplicada com LLMs — Aula 13: MLOps e empacotamento — CI e CD

---

## Objetivos de aprendizagem

Ao final desta nota, espera-se que o aluno seja capaz de:

- **Explicar** a integração contínua como prática de equipe — integrar no ramo principal várias vezes ao dia, com *build* e testes automáticos, e consertar primeiro o *build* quebrado — e **estimar** por que o custo da integração cresce mais que linearmente com o tempo de divergência.
- **Distinguir** entrega contínua de implantação contínua pelo ponto em que a decisão humana fica, e **distinguir** implantar de liberar.
- **Descrever** a anatomia de um *pipeline* — gatilho, *job*, etapa, executor, dependência, artefato, *cache*, *secret* e ambiente — e **justificar** a regra do artefato construído uma vez e promovido.
- **Ordenar** os portões de um *pipeline* pelo custo e pela probabilidade de rejeição, e **relacionar** essa ordem ao *shift left*.
- **Adaptar** o *pipeline* a um agente com LLM: separar testes determinísticos do *eval* com taxa mínima, tratar a chave do modelo como *secret* e **quantificar** o poder de discriminação de um *eval* pequeno.
- **Ler** o workflow de `05-agent-ci/` — os três *jobs*, as dependências, as *tags* da imagem — e **identificar** o que ele verifica e o que deixa de verificar.
- **Situar** Jenkins, GitHub Actions, GitLab CI/CD, CircleCI e Argo CD pelo modelo de execução, e **justificar** a fixação de uma *action* por SHA como decisão de cadeia de suprimentos.

---

## Desenvolvimento teórico

### 1. Integração contínua: a prática antes da ferramenta

A definição curta de **integração contínua** (*continuous integration*, CI) e o seu lugar entre as práticas de DevOps estão na [nota sobre DevOps](01-devops.md) (§8.2): integrar as mudanças ao ramo principal com frequência, verificando cada integração com *build* e testes automáticos, de modo que o ramo principal permaneça implantável (FOWLER, 2006). Esta seção trata do que a definição exige na prática e do custo que ela evita.

**A prática.** Fowler (2006) e Duvall, Matyas e Glover (2007) descrevem a integração contínua como um conjunto de hábitos da equipe, dos quais o servidor de CI é apenas o executor:

| Hábito | O que exige |
|---|---|
| repositório único | todo o necessário para construir o sistema está versionado, e o *build* parte de um *checkout* limpo |
| *build* automatizado | um único comando produz o artefato, sem passo manual |
| *build* que se testa | o mesmo comando executa a suíte automática, e uma falha de teste falha o *build* |
| integrar todo dia | cada desenvolvedor integra ao ramo principal ao menos uma vez por dia, em geral várias |
| verificar cada integração | cada *commit* no ramo principal dispara o *build* numa máquina neutra, e não só na de quem escreveu |
| consertar primeiro | um *build* quebrado no ramo principal é a prioridade: corrige-se em minutos ou reverte-se a mudança |
| *build* rápido | a verificação termina em poucos minutos, porque o retorno lento deixa de ser esperado e passa a ser ignorado |
| resultado visível | o estado do ramo principal é público, e todos sabem quando ele está quebrado |

A forma de ramificação compatível com esses hábitos é o **desenvolvimento baseado no tronco** (*trunk-based development*): ramos de vida curta, de horas ou de poucos dias, integrados ao tronco por mudanças pequenas. Ramos de semanas, com um servidor de CI configurado em cada um, têm a ferramenta e não têm a prática: cada ramo é verificado isoladamente, e a integração — o evento que a prática pretende tornar trivial — continua acontecendo raramente e em lote.

**A regra do *build* quebrado.** A segunda maneira do DevOps, na nota sobre DevOps (§6), afirma que um defeito detectado **para a linha**. Na integração contínua, a linha é o tronco: enquanto ele está vermelho, toda integração seguinte é feita sobre uma base defeituosa, e uma segunda falha se mistura à primeira. Por isso a correção do tronco precede qualquer trabalho novo, e a reversão do *commit* responsável é uma correção legítima — o objetivo é devolver o tronco ao estado implantável, e não depurar a mudança sob pressão.

**O custo da integração tardia.** O argumento da nota sobre DevOps (§3) sobre o tamanho do lote de implantação vale para o lote de integração, com um agravante. Sejam dois ramos que divergem do tronco por um intervalo $T$, cada um acumulando mudanças à taxa $\lambda$. Ao integrar, cada mudança de um lado pode conflitar com cada mudança do outro — textualmente, quando tocam as mesmas linhas, ou semanticamente, quando uma altera uma premissa de que a outra depende. O número de pares candidatos a conflito é

$$C(T) = (\lambda T)^2$$

quadrático em $T$, contra o crescimento linear do lote de cada lado. A estimativa é de ordem de grandeza: a maioria dos pares não conflita, e a fração que conflita depende do acoplamento do código. A conclusão, porém, não depende da constante: dobrar o intervalo entre integrações multiplica por quatro os pares a examinar, e é isso que transforma a integração rara num evento de dias, que a literatura de CI chama de "inferno da integração" (*integration hell*). Integrar várias vezes ao dia mantém $\lambda T$ próximo de 1, e o conflito, quando ocorre, envolve uma única mudança recente, ainda presente na memória de quem a escreveu.

**Limite.** A integração contínua garante o que a suíte verifica, e nada além. Um tronco verde com uma suíte fraca é um tronco implantável apenas no sentido formal.

### 2. Entrega contínua e implantação contínua

A distinção entre as duas está na nota sobre DevOps (§8.3): na **entrega contínua** (*continuous delivery*), toda mudança que passa pelo *pipeline* resulta num artefato que **pode** ir para produção; na **implantação contínua** (*continuous deployment*), toda mudança que passa **vai**, sem intervenção humana (HUMBLE; FARLEY, 2010). Esta seção aprofunda três pontos.

**Onde fica a decisão humana.** As duas práticas automatizam o mesmo caminho até o último portão e diferem só no que acontece depois dele. Na entrega contínua, a implantação é um comando executado por uma pessoa, que decide **quando** — por razão de negócio, de calendário ou de risco —, mas não **como**: o procedimento é o mesmo automatizado que já levou o artefato à homologação. O que a entrega contínua elimina não é a decisão, é o trabalho manual e o risco de executá-la. A implantação contínua retira também a decisão, e por isso exige que os portões automáticos sejam a única barreira considerada suficiente.

**Implantar não é liberar.** **Implantar** (*deploy*) é colocar uma versão do artefato em execução num ambiente; **liberar** (*release*) é expor um comportamento novo a usuários. As duas operações podem ser separadas: o código novo é implantado desligado, atrás de uma **chave de funcionalidade** (*feature flag*), e liberado depois, para uma fração dos usuários ou para todos, sem nova implantação. As estratégias de liberação da [nota sobre MLOps](02-mlops.md) (§10) — *canary*, *shadow*, teste A/B — são formas de liberar gradualmente o que já está implantado. A separação permite implantação contínua com liberação controlada: o tronco vai para produção a cada *commit*, e a exposição ao usuário segue outro ritmo.

**O artefato construído uma vez e promovido.** O *pipeline* constrói o artefato no primeiro estágio, e os estágios seguintes **promovem** o mesmo artefato, identificado pelo mesmo *digest*, entre ambientes (HUMBLE; FARLEY, 2010). O que varia entre ambientes é a configuração injetada na execução — endereço do banco, chave do modelo, nível de *log* —, e não o conteúdo do artefato.

```text
              build (uma vez)
                    |
                    v
        imagem  sha256:9f3c...   <- identidade do artefato
                    |
     +--------------+--------------+------------------+
     |              |              |                  |
     v              v              v                  v
  testes      homologação      produção          produção
  (CI)        cfg: homolog     cfg: prod         (reversão:
                                                  sha256:41ab...)
     o mesmo digest em todos os ambientes; muda só a configuração
```

Reconstruir o artefato em cada ambiente reabre a diferença que o *pipeline* existe para fechar: uma dependência resolvida de outra forma, uma imagem base atualizada entre um *build* e outro, e o que foi testado deixa de ser o que é implantado. A *tag* e o *digest* da imagem, tratados na [nota sobre de *script* a imagem](03-de-script-a-imagem.md) (§5.1), são o mecanismo concreto dessa identidade. A reversão, nesse esquema, é a promoção de um *digest* anterior, já testado, e não uma reconstrução.

**Limite.** A entrega contínua só é contínua se o artefato no fim do *pipeline* estiver de fato pronto para produção. Um *pipeline* que termina num artefato que ainda depende de um passo manual — uma migração aplicada à mão, uma configuração ajustada no servidor — automatiza a parte fácil e deixa a arriscada fora dele.

### 3. Anatomia de um *pipeline*

Um ***pipeline*** de CI/CD é um grafo acíclico dirigido de unidades de trabalho, declarado num arquivo versionado com o projeto e executado por um serviço a cada evento que o dispara. Os elementos são comuns às ferramentas da §7, com nomes que variam:

| Elemento | Definição |
|---|---|
| **gatilho** (*trigger*) | o evento que inicia uma execução: *push* para um ramo, abertura ou atualização de um *pull request*, agendamento periódico (*cron*), disparo manual, ou a conclusão de outro *pipeline* |
| ***job*** | a unidade de execução: um conjunto de etapas executado num mesmo executor, com sistema de arquivos próprio |
| **etapa** (*step*) | um comando ou uma ação reutilizável dentro do *job*, executados em sequência; a falha de uma etapa falha o *job* |
| **executor** (*runner*, *agent*) | a máquina, real ou virtual, ou o *container* que executa o *job*; hospedado pelo serviço ou mantido pela própria organização |
| **dependência** | a aresta do grafo: um *job* só começa quando os *jobs* de que depende terminam com sucesso; *jobs* sem dependência entre si executam em paralelo |
| **artefato** | um arquivo produzido por um *job* e guardado para outro *job* ou para download: o binário, a imagem, o relatório de testes |
| ***cache*** | um diretório preservado entre execuções e indexado por uma chave — em geral o *hash* do arquivo de dependências —, para não baixar de novo o que não mudou |
| ***secret*** | um valor sensível guardado pelo serviço, fora do repositório, entregue ao *job* como variável de ambiente e mascarado nos *logs* |
| **ambiente** (*environment*) | um destino de implantação nomeado — homologação, produção — com *secrets* e regras de proteção próprios, como aprovação obrigatória antes da execução |

Artefato e *cache* têm propósitos opostos. O artefato **transporta** o resultado de uma execução para adiante e precisa ser exato; o *cache* **acelera** a próxima execução e pode ser descartado sem mudar o resultado. Usar o *cache* para transportar o resultado entre *jobs* reintroduz a dependência de estado que o *pipeline* existe para eliminar.

```text
 gatilho: push / pull request / cron / manual
    |
    v
 +--------------------- execução do pipeline ---------------------+
 |                                                                |
 |   +-----------+      +-----------+                             |
 |   |  job A    |      |  job B    |   (A e B em paralelo)       |
 |   | runner 1  |      | runner 2  |                             |
 |   | etapas... |      | etapas... | <-- cache (dependências)    |
 |   +-----------+      +-----------+ <-- secrets (do serviço)    |
 |         |                  |                                  |
 |         +--------+---------+                                   |
 |                  | needs: A, B                                 |
 |                  v                                             |
 |            +-----------+                                       |
 |            |  job C    | --> artefato --> ambiente (produção)  |
 |            +-----------+                                       |
 +----------------------------------------------------------------+
```

O **executor** é um ponto de confiança. Ele recebe o código do repositório e os *secrets* do serviço, e executa o que o arquivo do *pipeline* manda. Num executor mantido pela própria organização e compartilhado entre repositórios, um *job* malicioso pode deixar algo para o *job* seguinte; os executores hospedados pelos serviços são, em geral, máquinas descartadas ao fim de cada *job*.

### 4. Os portões e a ordem

Cada *job* que pode reprovar a mudança é um **portão** (*gate*): a revisão só avança se ele passar. A ordem usual é

1. **análise estática e de segurança** — *lint* do código e do `Dockerfile`, varredura de configuração e de dependências vulneráveis: segundos, sem executar nada;
2. **testes unitários** — funções e módulos isolados, sem rede nem serviço externo: segundos;
3. **testes de integração** — o sistema com banco, fila ou API reais ou em *container*: minutos;
4. **testes de aceitação e *eval*** — o comportamento de ponta a ponta contra critérios de negócio: minutos a horas, e no caso de um agente, custo em *tokens*;
5. **publicação ou implantação** — o artefato vai para o registro ou para o ambiente.

**Formalização.** Sejam $n$ portões independentes, cada um com custo $c_i$ (tempo, dinheiro) e probabilidade $p_i$ de rejeitar uma mudança defeituosa. Executados na ordem $1, \dots, n$, com parada no primeiro que rejeita, o custo esperado por mudança é

$$E[C] = c_1 + (1 - p_1)\,c_2 + (1 - p_1)(1 - p_2)\,c_3 + \dots = \sum_{i=1}^{n} c_i \prod_{j<i} (1 - p_j)$$

Trocar dois portões adjacentes $i$ e $i+1$ só altera os dois termos correspondentes, e a troca reduz o custo se e somente se $c_{i+1}\,p_i < c_i\,p_{i+1}$. Logo, a ordem que minimiza $E[C]$ é a ordem crescente da razão $c_i / p_i$: primeiro o portão que rejeita mais por unidade de custo. A hipótese de independência é uma simplificação — os portões detectam defeitos sobrepostos —, mas a regra prática que ela produz é a que a literatura recomenda: o rápido e barato primeiro (HUMBLE; FARLEY, 2010). Um *lint* que leva dois segundos e pega um quinto das mudanças defeituosas vem antes de um *eval* que leva cinco minutos e pega metade.

A mesma ordem minimiza o **tempo de retorno**: quem integrou recebe a reprovação no menor tempo possível, ainda com o contexto da mudança. Na formalização, basta tomar $c_i$ como a duração do portão.

**Paralelismo.** Portões sem dependência entre si executam em paralelo, e o tempo de retorno passa a ser o do mais lento do grupo, e não a soma. A análise estática e os testes unitários costumam formar esse primeiro grupo; a publicação depende de todos.

***Shift left*.** Na representação do ciclo de desenvolvimento da esquerda (escrita) para a direita (produção), ***shift left*** é o deslocamento de uma verificação para mais cedo no ciclo: para o *pipeline* em vez da homologação, para o *commit* em vez do *pipeline*, para o editor em vez do *commit*. O argumento é o mesmo da formalização: o custo de corrigir um defeito cresce com a distância entre o momento em que ele foi introduzido e o momento em que é detectado. A análise de segurança do `Dockerfile` no primeiro portão é *shift left* de uma verificação que, sem o *pipeline*, só aconteceria numa auditoria da imagem em produção.

**Limite.** A ordem otimiza o custo de rejeitar; não substitui portões ausentes. Um defeito que nenhum portão detecta passa pelo *pipeline* mais barato possível e chega à produção do mesmo jeito.

### 5. CI/CD para um agente com LLM

A [nota sobre MLOps](02-mlops.md) (§5 e §9) estabeleceu que um sistema cujo comportamento depende de dados e de modelo precisa de portões que um *pipeline* de software convencional não tem, e que, num sistema com LLM, o portão de integração contínua é a suíte de regressão da aula de *evals*. Num agente, os portões se dividem em duas classes de natureza diferente.

**Testes determinísticos.** Verificam o código em volta do modelo: as ferramentas, a extração da resposta, a validação da entrada, a API. Não chamam o modelo — quando o fluxo passa por ele, o modelo é substituído por um **dublê** (*test double*) que devolve uma resposta fixa. São rápidos, gratuitos e repetíveis, e o critério é a igualdade: a ferramenta devolve exatamente o esperado ou o teste falha. Rodam em todo *push*.

**O *eval*.** Verifica o comportamento do agente com o modelo real contra um gabarito. É lento, custa *tokens* e não é determinístico: a mesma entrada pode produzir respostas diferentes em execuções diferentes, como tratado na [nota sobre o juiz e a suíte](../../aula-11-evals/notas-de-aula/03-o-juiz-e-a-suite.md) da aula de *evals*. Por isso o critério não é a igualdade caso a caso, e sim uma **taxa mínima de acerto** sobre o conjunto: o portão passa se a fração de casos corretos é maior ou igual a um limiar declarado antes da execução.

**O poder de um *eval* pequeno.** Seja $n$ o número de casos, $q$ a probabilidade de acerto de cada caso — tomada como igual e independente entre casos — e $k$ o mínimo de acertos exigido. O número de acertos é binomial, e a probabilidade de o portão passar é

$$P(\text{passa}) = \sum_{i=k}^{n} \binom{n}{i} q^i (1 - q)^{n - i}$$

Com $n = 4$ e limiar de 75%, ou seja $k = 3$:

| $q$ (acerto real por caso) | $P(\text{passa})$ |
|---|---|
| 0,9 | $0{,}9^4 + 4 \cdot 0{,}9^3 \cdot 0{,}1 \approx 0{,}95$ |
| 0,7 | $0{,}7^4 + 4 \cdot 0{,}7^3 \cdot 0{,}3 \approx 0{,}65$ |

Um agente que acerta 90% dos casos é reprovado em cerca de uma execução a cada vinte, sem que nada tenha mudado; um que acerta 70% passa em cerca de duas a cada três. Com quatro casos, o portão detecta regressões grosseiras — o agente que deixou de chamar a ferramenta, o prompt que quebrou a extração do veredito — e não distingue versões próximas. A cobertura e o tamanho do conjunto, a repetição de cada caso e o limiar de promoção são decisões da suíte de regressão, tratadas na aula de *evals* (§5 e §6 da nota citada); no *pipeline*, o *eval* é a execução automática dessa suíte, e herda o poder estatístico que ela tiver.

**O *secret* do modelo.** A chave do provedor é um *secret* do serviço de CI, entregue ao *job* do *eval* como variável de ambiente, e nunca um arquivo do repositório — a regra da [nota sobre de *script* a imagem](03-de-script-a-imagem.md) (§6), aplicada ao *pipeline*. Duas consequências decorrem disso:

- **o *eval* é opcional quando a chave não existe**: um *fork* do repositório, ou uma execução sem o *secret* configurado, roda os testes determinísticos e pula o *eval*, em vez de falhar por falta de credencial. No GitHub Actions, os *secrets* não são entregues a workflows disparados por *pull request* vindo de *fork* (GITHUB, 2026), e o mesmo *pipeline* precisa funcionar nos dois casos;
- **o custo é controlado pelo gatilho**: cada execução do *eval* consome *tokens*. Executar o *eval* só no *push* para o tronco, ou só em mudanças que tocam o prompt ou a configuração do modelo, ou por agendamento, é uma decisão de custo a declarar, e não um detalhe do arquivo.

**Limite.** O *eval* no *pipeline* mede o modelo no momento da execução. Um provedor que atualiza o modelo por trás de um nome sem versão muda o comportamento sem *commit*, e nenhum gatilho de *push* dispara; o *eval* agendado é a forma de detectar esse *drift* entre mudanças de código.

### 6. O caso: `05-agent-ci`

A pasta `codigo/05-agent-ci/` contém o agente como API da [nota sobre de *script* a imagem](03-de-script-a-imagem.md), com testes e um workflow do GitHub Actions em `.github/workflows/ci.yml`. O workflow tem três *jobs*:

```text
  gatilhos: push na main | pull request | disparo manual

  +---------------------------+     +-------------------------------+
  | seguranca                 |     | testes                        |
  |  hadolint  (Dockerfile)   |     |  pytest       (14 unitários)  |
  |  Trivy config (relatório) |     |  pytest -m eval  (se houver   |
  +---------------------------+     |    GROQ_API_KEY; taxa >= 75%) |
               |                    +-------------------------------+
               |                                   |
               +-----------------+-----------------+
                                 | needs: [seguranca, testes]
                                 v      só em push na main
               +-----------------------------------------+
               | publicar                                |
               |  docker build + push -> Docker Hub      |
               |  tags: latest  e  <SHA do commit>       |
               +-----------------------------------------+
```

| *Job* | O que verifica ou faz | Critério de falha |
|---|---|---|
| `seguranca` | **hadolint** lê o `Dockerfile` contra regras de boas práticas; **Trivy**, em modo de configuração, procura má configuração de segurança | hadolint falha só em achado de severidade *error*; o Trivy é informativo (`exit-code: "0"`) |
| `testes` | instala `requirements-dev.txt` com *cache* do pip; roda os testes unitários de ferramentas, do veredito e da API; roda o *eval* se o *secret* da chave existir | qualquer teste unitário falho; taxa de acerto do *eval* abaixo de 75% |
| `publicar` | constrói a imagem e a envia ao Docker Hub com duas *tags* | só executa em *push* para a `main` e depois que os dois *jobs* anteriores passam |

Os testes seguem a divisão da §5. Os unitários (`tests/test_ferramentas.py`, `tests/test_api.py`) não chamam o modelo: o `conftest.py` define uma chave falsa para que o cliente do modelo possa ser construído, e o teste da API substitui a função que roda o agente por um dublê. O *eval* (`tests/test_eval.py`) tem quatro casos com veredito conhecido pela política e o limiar de 75%; o marcador `eval`, excluído por padrão no `pytest.ini`, impede que um `pytest` sem argumento gaste *tokens*. O `requirements-dev.txt` acrescenta o pytest e o httpx às dependências do agente e não entra na imagem, que instala só o `requirements.txt`.

**Os resultados.** Os comandos equivalentes aos três *jobs*, listados em `comandos.md`, foram executados localmente; o workflow não foi executado no GitHub. O hadolint passou, com um único aviso — DL3042, `pip install` sem `--no-cache-dir`, que deixa o *cache* do pip dentro da camada —, abaixo do limiar de falha. O Trivy apontou o achado AVD-DS-0002, de severidade alta: o `Dockerfile` não declara `USER`, e o processo do *container* roda como `root`. Os 14 testes unitários passaram sem chamar o modelo, e o *eval* passou com o modelo real.

**O achado de `root` como portão opcional.** O Trivy está configurado para relatar e não reprovar, e o achado não impede a publicação. A escolha é deliberada e reversível por uma linha: com `exit-code: "1"`, o mesmo achado passa a barrar o *job* `publicar`. A decisão de quando um achado de segurança se torna portão é a mesma da formalização da §4, com um termo a mais: um portão que reprova sempre, por um achado conhecido e ainda não corrigido, é contornado ou desligado, e deixa de detectar o achado novo. A correção do `Dockerfile` — um usuário sem privilégio declarado na imagem — e a leitura do arquivo como superfície de ataque são tratadas na aula de segurança, na [nota sobre cadeia, imagem e rede](../../aula-15-seguranca/notas-de-aula/03-cadeia-rede-e-o-fecho.md) (§3).

**As duas *tags*.** A imagem publicada recebe `latest` e o SHA do *commit* (`github.sha`). A *tag* `latest` é movida a cada publicação e responde "qual é a imagem mais recente do tronco"; a *tag* do SHA é escrita uma vez e liga a imagem ao *commit* exato que a produziu — a linhagem da nota sobre MLOps, mecanizada pelo *pipeline* em vez de digitada (nota sobre de *script* a imagem, §5.1). Por isso a implantação referencia a *tag* do SHA: a referência a `latest` não identifica qual imagem foi implantada, e a mesma referência aponta para outra imagem a cada publicação.

**Outras decisões do arquivo.** `permissions: contents: read` restringe o *token* que o GitHub entrega a cada *job* à leitura do repositório: um passo comprometido não consegue escrever no repositório com ele. O *pull request* executa `seguranca` e `testes`, mas não `publicar`: a mudança é verificada antes de chegar ao tronco, e só o tronco produz imagem publicada. O token do Docker Hub, e não a senha da conta, é o *secret* de publicação, porque um token pode ser revogado sozinho e ter escopo restrito.

**O que o *pipeline* não verifica.** Duas lacunas em relação ao modelo da §2:

- **os testes não rodam na imagem publicada.** O *job* `testes` executa o código-fonte no Python do executor; a imagem só é construída no *job* `publicar`, depois dos testes. Uma diferença entre o ambiente do executor e o da imagem — uma dependência de sistema ausente, um arquivo fora do `COPY` — passa pelos testes e chega ao registro. A forma estrita constrói a imagem primeiro, executa os testes dentro dela e publica o mesmo *digest*;
- **não há implantação.** O *pipeline* termina no registro: é integração contínua com entrega do artefato, e não implantação contínua. Levar a imagem a um ambiente é um passo posterior, manual ou de outra ferramenta (§7).

### 7. As ferramentas

As ferramentas de CI/CD diferem em quem hospeda o executor, onde o *pipeline* é declarado e em que direção o artefato chega ao ambiente. As cinco abaixo são citadas como exemplos da categoria, sem tutorial.

| | Ferramenta | Modelo | *Pipeline* declarado em |
|---|---|---|---|
| <img src="../imagens/logo-jenkins.svg" alt="Jenkins" width="32"> | **Jenkins** | servidor de automação de código aberto, auto-hospedado; executa *jobs* em agentes ligados ao servidor e estende-se por *plugins* | `Jenkinsfile` na raiz do repositório, em sintaxe declarativa ou em *script* Groovy (JENKINS, 2026) |
| <img src="../imagens/logo-githubactions.svg" alt="GitHub Actions" width="32"> | **GitHub Actions** | serviço integrado ao GitHub; executores hospedados pelo GitHub ou pela organização; etapas reutilizáveis publicadas como *actions* | arquivos YAML em `.github/workflows/` (GITHUB, 2026) |
| <img src="../imagens/logo-gitlab.svg" alt="GitLab" width="32"> | **GitLab CI/CD** | integrado ao GitLab, hospedado ou auto-hospedado; *jobs* executados pelo GitLab Runner e agrupados em estágios | `.gitlab-ci.yml` na raiz do repositório (GITLAB, 2026) |
| <img src="../imagens/logo-circleci.svg" alt="CircleCI" width="32"> | **CircleCI** | serviço hospedado, independente da plataforma do repositório; configuração reutilizável empacotada em *orbs* | `.circleci/config.yml` (CIRCLECI, 2026) |
| <img src="../imagens/logo-argo.svg" alt="Argo CD" width="32"> | **Argo CD** | ferramenta de implantação contínua para Kubernetes por GitOps: um agente no *cluster* compara o estado declarado num repositório git com o estado real e sincroniza a diferença | manifestos Kubernetes versionados num repositório git (ARGO PROJECT, 2026) |

**O que as ferramentas têm em comum.** Nas cinco, o *pipeline* é **código**: um arquivo de texto versionado no repositório, revisado como o resto da mudança, com histórico e reversão. A prática de controle de versão de tudo, da nota sobre DevOps (§8.1), estendida ao próprio processo de entrega: o *pipeline* deixa de ser configuração feita num painel e conhecida por uma pessoa. A anatomia da §3 vale para todas; a migração de uma ferramenta para outra é tradução de sintaxe, e não de conceito.

***Push* e *pull*.** As quatro primeiras implantam por ***push***: o *pipeline*, ao terminar, executa o comando que leva o artefato ao ambiente, e por isso precisa de credencial de escrita no ambiente. O Argo CD implanta por ***pull***: o *pipeline* só atualiza o repositório que declara o estado desejado — por exemplo, a *tag* da imagem num manifesto —, e o agente dentro do *cluster* detecta a diferença e a aplica. Esse modelo é chamado **GitOps**: o repositório git é a fonte de verdade do estado do ambiente, e a reconciliação entre declarado e real é contínua — a infraestrutura como código da nota sobre DevOps (§8.4), com o ciclo de aplicação automatizado. A credencial de escrita no ambiente fica dentro dele, e não no serviço de CI.

**Fixar uma *action*: por *tag* ou por SHA.** O workflow de `05-agent-ci` referencia cada *action* por *tag* de versão (`actions/checkout@v4`). A *tag* é um ponteiro que quem controla o repositório da *action* pode mover para outro *commit*; o SHA completo do *commit* é a única referência imutável, e é a forma recomendada pela documentação de segurança do GitHub Actions (GITHUB, 2026). O risco tem precedente: em março de 2025, as *tags* da *action* `tj-actions/changed-files` foram movidas para um *commit* malicioso que imprimia os *secrets* dos workflows nos *logs* (CVE-2025-30066), e todo workflow que a referenciava por *tag* executou o código novo sem nenhuma mudança no próprio repositório. O argumento é o mesmo da imagem base fixada até o *patch*, na nota sobre de *script* a imagem (§4.2): uma referência móvel transfere a quem a move a decisão sobre o que é executado. A fixação por SHA tem o custo de tornar a atualização manual, e ferramentas de atualização de dependências abrem a mudança como *pull request* para revisão. O tema — a cadeia de suprimentos de software e as referências móveis — é desenvolvido na aula de segurança, na nota sobre cadeia, imagem e rede.

---

## Fontes e leituras

ARGO PROJECT. **Argo CD**: declarative GitOps CD for Kubernetes. Documentação oficial, 2026. Disponível em: https://argo-cd.readthedocs.io/.

CIRCLECI. **CircleCI documentation**. 2026. Disponível em: https://circleci.com/docs/.

DUVALL, P. M.; MATYAS, S.; GLOVER, A. **Continuous integration**: improving software quality and reducing risk. Upper Saddle River: Addison-Wesley, 2007.

FORSGREN, N.; HUMBLE, J.; KIM, G. **Accelerate**: the science of lean software and DevOps. Portland: IT Revolution, 2018.

FOWLER, M. **Continuous integration**. 2006. Disponível em: https://martinfowler.com/articles/continuousIntegration.html.

GITHUB. **GitHub Actions documentation**. GitHub Docs, 2026. Disponível em: https://docs.github.com/en/actions. — sintaxe de workflow, *secrets* (inclusive em *pull requests* vindos de *fork*), permissões do `GITHUB_TOKEN` e o guia de uso seguro, com a fixação de *actions* pelo SHA completo do *commit*.

GITLAB. **GitLab CI/CD**. GitLab Docs, 2026. Disponível em: https://docs.gitlab.com/ci/.

HADOLINT. **Hadolint**: Haskell Dockerfile linter. 2026. Disponível em: https://github.com/hadolint/hadolint. — regras como a DL3042.

HUMBLE, J.; FARLEY, D. **Continuous delivery**: reliable software releases through build, test, and deployment automation. Boston: Addison-Wesley, 2010. Cap. 3: Continuous integration; cap. 5: Anatomy of the deployment pipeline.

JENKINS. **Pipeline**. Jenkins User Handbook, 2026. Disponível em: https://www.jenkins.io/doc/book/pipeline/. — o `Jenkinsfile` e as sintaxes declarativa e de *script*.

KIM, G.; HUMBLE, J.; DEBOIS, P.; WILLIS, J. **The DevOps handbook**: how to create world-class agility, reliability, and security in technology organizations. Portland: IT Revolution, 2016.

TRIVY. **Trivy documentation**. Aqua Security, 2026. Disponível em: https://trivy.dev/. — varredura de configuração (*misconfiguration*) e o achado AVD-DS-0002.

**Material da disciplina.** [DevOps](01-devops.md), nesta aula — a definição de integração contínua (§8.2), de entrega e implantação contínuas (§8.3) e o argumento do tamanho do lote (§3). [MLOps](02-mlops.md), nesta aula — os portões novos (§5) e CI/CD/CT (§9). [De *script* a imagem](03-de-script-a-imagem.md), nesta aula — o `Dockerfile` (§4.2), a *tag* da imagem (§5.1) e o segredo (§6). [O juiz e a suíte](../../aula-11-evals/notas-de-aula/03-o-juiz-e-a-suite.md), em *evals* — a suíte de regressão e a pergunta estatística. [Cadeia, imagem, rede, e o fecho](../../aula-15-seguranca/notas-de-aula/03-cadeia-rede-e-o-fecho.md), em segurança — a imagem relida como superfície e as referências móveis na cadeia de suprimentos.

**Logos.** Os logos das ferramentas vêm do projeto Simple Icons (https://simpleicons.org/), distribuídos sob a licença CC0 1.0; as marcas pertencem aos respectivos titulares.
