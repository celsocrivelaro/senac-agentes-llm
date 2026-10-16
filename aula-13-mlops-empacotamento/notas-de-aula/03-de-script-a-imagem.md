# IA Aplicada com LLMs — Aula 13: MLOps e empacotamento — De script a imagem

---

## Objetivos de aprendizagem

Ao final desta nota, espera-se que o aluno seja capaz de:

- **Explicar** por que o artefato que sai da máquina de quem escreveu o agente é a imagem, e não o código.
- **Distinguir** imagem e container, e **identificar** o que fica fora da imagem — segredo, modelo e estado.
- **Empacotar** o agente em uma imagem Docker reprodutível, com imagem base e dependências fixadas, a ordem das camadas justificada, cópia explícita dos módulos e nenhum segredo em camada.
- **Executar** o agente em container com o segredo entregue em tempo de execução, a imagem identificada por nome e *tag* e o modelo alcançado fora do container — como programa de linha de comando e como API com a porta publicada.
- **Localizar** um segredo no histórico do git e numa camada de imagem, e **justificar** por que a correção é revogar a chave.
- **Decidir** onde mora o estado de um sistema cujo container é efêmero, com o preço de cada saída, e **compor** com Docker Compose a API e um banco Postgres cujo estado vive num volume.
- **Escolher** entre fila, *webhook* e SSE quando a execução se alonga, e **justificar** a execução durável como requisito.

---

## Desenvolvimento teórico

### 1. De script a imagem

Um agente que funciona numa máquina depende do que aquela máquina tem: a versão do interpretador Python, os pacotes instalados e as versões de cada um, as variáveis de ambiente, as bibliotecas do sistema operacional. Nenhuma dessas dependências está escrita no código, e por isso copiar o código para outra máquina não reproduz o comportamento — reproduz só a parte que estava declarada.

Uma **imagem** é um sistema de arquivos imutável, acompanhado dos metadados de execução (diretório de trabalho, variáveis de ambiente, porta, comando), a partir do qual o Docker cria processos isolados. A imagem carrega o código **e** tudo de que ele depende, e é construída a partir de um **`Dockerfile`**: o arquivo versionado que declara, instrução por instrução, como o sistema de arquivos é montado. O artefato que viaja entre máquinas deixa de ser o código e passa a ser a imagem.

```text
  máquina de quem escreveu            qualquer máquina com Docker
  +-------------------------+         +-------------------------+
  | agente.py, cliente.py   |         |  +-------------------+  |
  | Python 3.x (qual?)      |  build  |  | imagem            |  |
  | pacotes (quais versões?)| ------> |  |  código           |  |
  | variáveis de ambiente   |         |  |  Python fixado    |  |
  | sistema operacional     |         |  |  pacotes fixados  |  |
  +-------------------------+         |  |  diretório, cmd.  |  |
     dependências implícitas          |  +-------------------+  |
                                      +-------------------------+
                                         dependências declaradas
```

É a aplicação, ao agente, do tópico de *deploy* e *serving* descrito na [nota sobre MLOps](02-mlops.md) (§10), na parte de conteinerização. Nada aqui é novo em conceito: é o sistema construído ao longo do curso saindo da máquina de quem o escreveu. O motivo de declarar o ambiente é o mesmo que motivou o versionamento do prompt na aula de *prompt engineering*: o que não está declarado não é reproduzível nem comparável.

O agente do código da aula é deliberadamente pequeno — um auditor de despesas com duas ferramentas (`consultar_despesa` e `consultar_politica`), montado com `create_agent` —, porque o assunto não é o agente, e sim a imagem que o carrega. Ele aparece em três formas, cada uma numa pasta autossuficiente, que é também o contexto de build da própria imagem:

| Pasta | Forma | O que muda em relação à anterior |
|---|---|---|
| `01-agente-cli/` | programa de linha de comando: recebe o objetivo como argumento, responde e termina | — |
| `02-agente-web/` | API HTTP: recebe a pergunta por `POST /perguntas` e devolve a resposta (§5.3) | o último bloco do `Dockerfile` e a forma de executar |
| `03-agente-web-banco/` | a mesma API, com a política de reembolso numa tabela do Postgres, banco e API subidos juntos por Docker Compose (§7.1) | nada na imagem da API; muda o ambiente em volta dela |

As seções 4 a 7 usam a primeira forma, salvo indicação; a segunda é tratada em §5.3 e a terceira em §7.1.

### 2. Imagem e container

A distinção foi apresentada na [nota sobre Docker](../../aula-00-sobre-o-curso/notas-de-aula/02-docker.md), na aula sobre o curso; esta seção fixa as duas definições no sentido que o restante da nota exige.

- **Imagem**: o artefato **imutável**, construído uma vez e identificado por um nome com *tag* (`auditor-de-despesas:1.0.0`). Não executa nada; é o molde.
- **Container**: uma **execução** da imagem — um processo isolado do sistema hospedeiro, cujo sistema de arquivos nasce do sistema de arquivos da imagem, acrescido de uma camada gravável própria. O container termina quando o processo principal termina.

Uma imagem dá origem a quantos containers forem executados, e nenhum deles altera a imagem: o que um container grava fica na camada gravável dele, e não na imagem. Duas consequências estruturam esta nota. A primeira é que a imagem pode ser copiada e executada em qualquer máquina com Docker com o mesmo comportamento, que é o objetivo do empacotamento. A segunda é que o container é **descartável**: o que ele grava não faz parte do artefato, e a §7 trata do que isso exige.

### 3. O que fica fora da imagem

Três coisas, por razões diferentes, não entram na imagem do agente. Cada uma tem uma seção desta nota.

| O que | Por que fica fora | Como chega ao container | Seção |
|---|---|---|---|
| **segredo** (a chave de API) | quem tem a imagem teria a chave; e a mesma imagem precisa rodar com chaves diferentes | variável de ambiente, em tempo de execução (`--env-file`) | §5, §6 |
| **modelo** | é um provedor de API ou um servidor (Ollama) que roda fora do container | endereço e chave, por variável de ambiente | §5.2 |
| **estado** (índice, memória, dados de negócio) | é escrito ou alterado durante a operação, e o que o container escreve não volta para a imagem | volume, banco ou serviço, ou reconstrução | §7 |

O critério comum é a separação entre o que é **igual em toda execução** — o código, o interpretador, as dependências —, que vai para a imagem, e o que **varia por ambiente ou por execução**, que entra pelo ambiente do container. A mesma imagem, sem reconstrução, roda com Mistral numa máquina e com Ollama em outra, porque o provedor é escolhido por `LLM_PROVEDOR` no ambiente, e não no código.

O `cliente.py` funciona igual dentro e fora do container por um detalhe de projeto. Ele chama `load_dotenv()`, que lê o arquivo `.env` se ele existir no diretório e **não sobrescreve** variáveis já presentes no ambiente. Fora do container, o `.env` existe e é lido. Dentro do container, o `.env` não existe — está fora do contexto de build (§4.4) —, e as variáveis já chegaram ao processo pelo `--env-file`.

### 4. A imagem

#### 4.1 Camadas e a ordem das instruções

Cada instrução do `Dockerfile` que altera o sistema de arquivos (`RUN`, `COPY`, `ADD`) produz uma **camada** (*layer*): o conjunto das diferenças em relação à camada anterior. Instruções como `EXPOSE`, `CMD` e `ENTRYPOINT` gravam apenas metadados. A imagem é a pilha ordenada dessas camadas, e cada camada é imutável depois de criada (DOCKER, 2026a).

O construtor mantém um **cache de build**: a camada da instrução $i_k$ é reaproveitada se, e somente se, a própria instrução e os arquivos que ela lê não mudaram **e** as instruções $i_1, \dots, i_{k-1}$ também foram reaproveitadas. Uma mudança que invalida $i_k$ obriga a reconstruir todas as camadas de $i_k$ até $i_n$, e o tempo do build passa a ser

$$
T_{\text{build}}(k) = \sum_{j=k}^{n} t_j ,
$$

onde $t_j$ é o tempo de construir a camada $j$. No agente, a camada mais cara é a instalação das dependências — download e instalação de algumas dezenas de pacotes, na ordem de dezenas de segundos a minutos, conforme a rede (estimativa) —, e a que muda com mais frequência é o código, a cada *commit*. A ordem que minimiza o build típico põe **o que muda pouco antes do que muda muito**: o `requirements.txt` é copiado e instalado **antes** do código.

```text
  ORDEM INVERTIDA                       ORDEM DO DOCKERFILE DA AULA
  COPY requirements.txt + código        COPY requirements.txt
  RUN pip install   <- invalidada       RUN pip install    <- cache
                                        COPY agente.py ... <- invalidada

  após mudar 1 linha do agente.py:      após mudar 1 linha do agente.py:
    pip install ....... refeito           pip install ....... cache
    COPY .............. refeito           COPY .............. refeito
```

Com a ordem invertida, alterar uma linha do agente reinstala todas as dependências. A verificação é direta: alterar uma linha do `agente.py` e reconstruir; a saída do `docker build` marca a camada do `pip install` como `CACHED`.

#### 4.2 O `Dockerfile` do agente

O `Dockerfile` de `01-agente-cli/` tem quatro blocos, separados por linhas de comentário numeradas. O trecho abaixo reproduz as instruções e os separadores, sem os comentários explicativos, que estão no arquivo:

```dockerfile
# ---------------------------------------------------------------- 1
FROM python:3.11.13-slim-bookworm
WORKDIR /app

# ---------------------------------------------------------------- 2
COPY requirements.txt .
RUN pip install -r requirements.txt

# ---------------------------------------------------------------- 3
COPY agente.py cliente.py ./

# ---------------------------------------------------------------- 4
ENTRYPOINT ["python", "agente.py"]
```

No `Dockerfile`, comentário só é reconhecido em linha própria, iniciada por `#`; um `#` no fim de uma instrução vira argumento dela. Por isso a numeração ocupa linhas separadas.

| # | Instrução | Por quê |
|---|---|---|
| 1 | imagem base com versão **fixada** até o *patch*, e o diretório de trabalho | `python:3.11-slim` é uma etiqueta móvel: a mesma linha produz imagens diferentes em dias diferentes. É o problema do alias `-latest`, visto na escolha de modelos, aplicado ao ambiente. O `WORKDIR /app` cria o diretório e o torna o diretório corrente das instruções seguintes e do processo |
| 2 | `requirements.txt` **antes** do código | a camada mais cara só é reconstruída quando as dependências mudam (§4.1) |
| 3 | **só** os módulos do agente, por nome | nada além do listado entra na imagem, mesmo que o contexto de build contenha mais; é a segunda barreira contra o `.env` (§4.4) |
| 4 | `ENTRYPOINT` na forma vetorial | fixa o programa; os argumentos passados ao `docker run` depois do nome da imagem são acrescentados ao comando e chegam ao `agente.py` como `sys.argv[1:]`, que é o objetivo |

A forma vetorial do `ENTRYPOINT` (`["python", "agente.py"]`) executa o programa diretamente, sem um *shell* intermediário: o Python é o processo principal do container. A forma de *shell* (`ENTRYPOINT python agente.py`) interpõe um `sh -c`, que não repassa os argumentos do `docker run` ao programa (DOCKER, 2026a).

Os três `Dockerfile` da aula têm os mesmos quatro blocos. Os blocos 1 e 2 são idênticos nas três pastas; o bloco 3 acrescenta `api.py` à lista nas formas de API; o bloco 4 é o que distingue o programa que termina do servidor que fica de pé (§5.3).

O `Dockerfile` não declara usuário, e por isso o processo do container roda como `root`, o padrão do Docker. Os critérios desta nota são reprodutibilidade, tempo de build e segredo fora da imagem; a leitura do mesmo arquivo como superfície de ataque — o processo como `root` entre os defeitos que ela aponta — é feita na aula de segurança, na [nota sobre cadeia, imagem e rede](../../aula-15-seguranca/notas-de-aula/03-cadeia-rede-e-o-fecho.md).

#### 4.3 Só o que o agente importa

O `requirements.txt` de `01-agente-cli/` lista apenas as dependências que o agente importa — o LangChain e o LangGraph, as três integrações de provedor que o `cliente.py` pode construir, com os clientes `ollama` e `groq` de que elas dependem, e o `python-dotenv` —, todas com versão exata (`langchain==1.3.15`). O de `02-agente-web/` acrescenta só o `fastapi` e o `uvicorn`, que a API importa; o de `03-agente-web-banco/` acrescenta ainda o driver do Postgres, `psycopg[binary]==3.2.10`, cuja variante `binary` traz a `libpq` compilada e dispensa compilador na imagem *slim*. A imagem instala esse arquivo, e não o `requirements.txt` da disciplina inteira, que reúne as dependências de todas as aulas.

A escolha tem três efeitos: imagem menor, build mais rápido e **menos código de terceiro dentro do container**. O terceiro efeito é o de segurança: cada pacote instalado é código que roda com as permissões do processo e que pode conter vulnerabilidade, e um pacote que o agente não importa acrescenta essa superfície sem acrescentar função.

A versão exata fixa o segundo nível de reprodutibilidade. A imagem base fixada (bloco 1) garante o mesmo interpretador; o `requirements.txt` com `==` garante os mesmos pacotes diretos. Sem os dois, uma reconstrução em outro dia pode trazer uma versão nova de uma dependência com API diferente — o efeito que a aula de *frameworks* e orquestração mostrou ao tratar da evolução das APIs do LangChain.

#### 4.4 O contexto de build e o `.dockerignore`

O **contexto de build** é o conjunto de arquivos que o construtor pode ler durante o build — o diretório passado ao `docker build`, que no agente é o `.` ao fim do comando, a própria pasta da forma. O que não está no contexto não pode ser copiado para a imagem por nenhuma instrução (DOCKER, 2026b).

O **`.dockerignore`** é a lista de padrões excluídos do contexto. O das três pastas é o mesmo, e é, sem os comentários:

```text
.env
.env.*
!.env.example
*.key
*.pem

.git
.venv
__pycache__
*.pyc
```

A lista é a mesma do `.gitignore`, pela mesma razão: o que não deve entrar no repositório também não deve entrar na imagem. A linha `!.env.example` reinclui o modelo de configuração, que não contém segredo.

O `.dockerignore` e a lista explícita do `COPY` (bloco 3) são **duas barreiras independentes**. Um `COPY . .` copiaria todo o contexto e dependeria só do `.dockerignore` para manter o `.env` fora da imagem; uma linha apagada no `.dockerignore` bastaria para o vazamento. Com o `COPY` listando os arquivos por nome, o `.env` precisaria estar no contexto **e** ser nomeado numa instrução.

### 5. Construir e executar

A partir de `01-agente-cli/`, dois comandos constroem a imagem e executam um container:

```bash
docker build -t auditor-de-despesas:1.0.0 .
docker run --rm --env-file .env auditor-de-despesas:1.0.0 "A despesa D-4613 está dentro da política?"
```

- `-t` dá à imagem o nome e a *tag* (§5.1); o `.` é o contexto de build.
- `--rm` remove o container quando o processo termina; `--env-file .env` lê o arquivo **na máquina hospedeira** e entrega cada linha como variável de ambiente do processo. O arquivo não entra na imagem nem no container — entram apenas os valores, no ambiente do processo.
- O texto depois do nome da imagem é acrescentado ao `ENTRYPOINT` e vira o objetivo do agente.

A primeira linha da saída identifica o provedor e o modelo em uso, pelo `print` do `agente.py`; a segunda repete o objetivo, e em seguida vem a resposta do agente:

```python
print(f"auditor-de-despesas · {PROVEDOR}/{MODELO}")
print(f"objetivo: {objetivo}\n")
```

O build não chama o modelo; cada `docker run` faz uma execução do agente, com o custo de uma execução.

A VM da disciplina não tem Docker e não é alterada para isso. Na VM, o agente roda diretamente com `python agente.py`, e o `Dockerfile` é escrito e versionado mesmo sem ser construído ali; o build e a execução em container ficam para a máquina que tiver Docker instalado.

#### 5.1 A *tag* da imagem

Uma imagem é referida por um **nome com *tag***, na forma `nome:tag` — `auditor-de-despesas:1.0.0`. O nome identifica o produto; a *tag* distingue as construções dele. As três formas da aula usam nomes distintos, porque são produtos distintos: `auditor-de-despesas:1.0.0`, `auditor-de-despesas-web:1.0.0` e `auditor-de-despesas-banco:1.0.0` (este último atribuído pela chave `image` do `docker-compose.yml`, ao lado de `build`). Sem `:tag`, o Docker assume `latest`, que é apenas uma *tag* como outra qualquer, e não "a mais recente" em sentido algum (DOCKER, 2026a).

A *tag* é a **identidade do artefato imutável** do ponto de vista de quem opera: o comando que implanta, o registro que armazena e o histórico de implantações se referem à imagem por ela. Dois limites definem o que ela garante:

- **A *tag* é um rótulo, e pode ser reatribuída.** Um novo `docker build -t auditor-de-despesas:1.0.0` move o rótulo para a imagem nova, e a anterior perde o nome. O identificador que não muda é o *digest* do conteúdo (`sha256:...`). Por isso, uma *tag* de versão é tratada como gravada uma vez: uma alteração gera uma *tag* nova (`1.0.1`), e nunca reconstrói a existente.
- **O programa não lê a *tag*.** O nome da imagem existe nos metadados do Docker, fora do container; o `agente.py` não o conhece e não o imprime. Saber qual imagem está em execução é pergunta feita ao Docker (`docker ps` lista a imagem de cada container), e não ao processo.

Na terminologia da [nota sobre MLOps](02-mlops.md), a *tag* é o elo de linhagem entre o artefato implantado e o comportamento observado: uma regressão medida em produção aponta para uma imagem específica, e a imagem aponta para um *commit*. A coincidência entre *tag* e *commit* é disciplina do processo de build, e não do Docker: na prática, o comando de build é gerado por um *script* ou por um *pipeline* de integração contínua, e não digitado.

#### 5.2 Onde o container encontra o modelo

O modelo fica fora da imagem (§3), e o container precisa alcançá-lo pela rede. Com provedor de API (Mistral, Groq), o endereço é público e basta a chave, entregue pelo `--env-file`.

Com o Ollama, o servidor do modelo roda na máquina hospedeira, e o endereço padrão do `cliente.py` é `http://localhost:11434`. Dentro do container, **`localhost` é o próprio container**, e não a máquina hospedeira: o container tem interface de rede própria, e nenhum processo escuta a porta 11434 dentro dele. A chamada falha com conexão recusada.

```text
  máquina hospedeira
  +-----------------------------------------------------------+
  |  Ollama :11434                                            |
  |     ^                                                     |
  |     |  host.docker.internal:11434                         |
  |  +--|------------------------------------+                |
  |  |  container                            |                |
  |  |  agente.py --> localhost:11434 --X    |  (o próprio    |
  |  |                                       |   container)   |
  |  +---------------------------------------+                |
  +-----------------------------------------------------------+
```

O nome **`host.docker.internal`** resolve, de dentro do container, para a máquina hospedeira. O `.env` não precisa mudar: o endereço é sobrescrito no `docker run`, e a variável passada por `-e` prevalece sobre a do `--env-file`:

```bash
docker run --rm --env-file .env -e OLLAMA_BASE_URL=http://host.docker.internal:11434 auditor-de-despesas:1.0.0
```

Dois limites. No Docker Desktop (macOS e Windows), o nome existe por padrão; no Docker Engine em Linux, é preciso declará-lo com `--add-host=host.docker.internal:host-gateway`. E o Ollama escuta, por padrão, apenas em `127.0.0.1`; quando a conexão chega pela interface da ponte do Docker, e não pelo *loopback*, o servidor precisa ser configurado para escutar em outras interfaces (`OLLAMA_HOST=0.0.0.0`), com a consequência de ficar acessível a quem alcançar a máquina pela rede (OLLAMA, 2026).

#### 5.3 O agente como API

A pasta `02-agente-web/` empacota o mesmo agente atrás de uma API HTTP. O `agente.py` expõe a função `rodar(pergunta)`, e o `api.py`, escrito com FastAPI, define dois *endpoints*:

| *Endpoint* | O que faz |
|---|---|
| `POST /perguntas` | recebe `{"pergunta": "..."}`, executa o agente e devolve a pergunta e a resposta |
| `GET /saude` | responde sem chamar o modelo, com `ok` e o provedor e modelo em uso: indica que o processo está de pé |

Os blocos 1 a 3 do `Dockerfile` são os mesmos da forma de linha de comando, com `api.py` acrescentado à lista do `COPY`. O bloco 4 muda, porque o processo deixa de ser um programa que executa e termina e passa a ser um servidor que fica de pé:

```dockerfile
EXPOSE 8000
CMD ["uvicorn", "api:app", "--host", "0.0.0.0", "--port", "8000"]
```

E a execução passa a publicar uma porta:

```bash
docker build -t auditor-de-despesas-web:1.0.0 .
docker run --rm -p 8000:8000 --env-file .env auditor-de-despesas-web:1.0.0
curl -X POST localhost:8000/perguntas -H 'content-type: application/json' -d '{"pergunta": "A despesa D-4613 está dentro da política?"}'
```

Três pontos decorrem da mudança:

- **`--host 0.0.0.0` é obrigatório.** O container tem a própria interface de rede, e o `127.0.0.1` dele é o *loopback* do container. Com o padrão do uvicorn, o servidor escuta só ali, e o tráfego que o Docker encaminha pela porta publicada chega a uma interface em que nada escuta. É o mesmo fato de §5.2, no sentido inverso: lá, `localhost` dentro do container não alcança a máquina; aqui, a máquina não alcança o `localhost` do container.
- **`EXPOSE` documenta, `-p` publica.** A instrução `EXPOSE` registra na imagem a porta em que o processo escuta, mas não abre nada; é o `-p 8000:8000` do `docker run` que liga a porta 8000 da máquina hospedeira à do container (DOCKER, 2026a).
- **O nome do módulo tem de ser importável.** O uvicorn recebe `api:app` e **importa** o módulo `api`; um arquivo chamado `02-api.py`, na convenção numerada das outras pastas, não poderia ser importado, porque um nome que começa por dígito não é identificador Python válido.

A API é **síncrona**: a conexão fica aberta enquanto o agente trabalha, e a resposta só chega quando a execução termina. Serve a um agente de poucos passos, como este, que responde em segundos. O limite dessa forma, para execuções longas, é o assunto de §8.

### 6. O segredo, e o que o histórico guarda — duas vezes

A regra cobrada desde a aula de fundamentos de LLMs e agentes vale aqui sem alteração: **a chave de API nunca entra no repositório**. O `.env` está no `.gitignore` e no `.dockerignore`, e a chave chega ao container em tempo de execução, pelo `--env-file`. O segredo vaza de duas formas, e as duas têm a mesma estrutura: **apagar o arquivo não apaga o segredo**.

**No git.** Um *commit* é imutável e guarda o conteúdo integral de cada arquivo versionado. Se o `.env` foi commitado uma vez, apagá-lo num *commit* seguinte só acrescenta um *commit* em que o arquivo não existe; a chave continua no anterior, em todo clone e em todo *fork*. Duas verificações respondem se isso já ocorreu:

```bash
git log --all -- .env
git log --all -S "_API_KEY=" --oneline
```

A primeira lista os *commits* que tocaram o arquivo `.env`; a segunda, os *commits* em que o número de ocorrências da cadeia `_API_KEY=` mudou, em qualquer arquivo — o que encontra a chave mesmo quando ela foi colada num arquivo com outro nome. Saída vazia nas duas é o resultado esperado.

**Na imagem.** Uma camada é imutável: uma instrução posterior que apaga um arquivo apenas acrescenta uma camada nova, que o esconde, e a camada anterior continua na imagem. O `Dockerfile` abaixo parece seguro e não é:

```dockerfile
# a chave entra nesta camada
COPY .env .
RUN pip install -r requirements.txt
# esta camada esconde o arquivo; a do COPY ainda o contém
RUN rm .env
```

O `docker history` lista as camadas da imagem, com a instrução que gerou cada uma e o tamanho, e a do `COPY .env` aparece com os poucos bytes do arquivo:

```bash
docker history auditor-de-despesas:1.0.0
```

Exportar a imagem com `docker save` produz um arquivo com as camadas, e abrir a camada do `COPY` recupera o `.env` íntegro. Quem tem a imagem — um registro, uma cópia em outra máquina, um cache de build — tem a chave. O mesmo vale para um segredo passado por `--build-arg`: os valores de `ARG` ficam registrados nos metadados da imagem e aparecem no `docker history`, e por isso o argumento de build nunca transporta a chave (DOCKER, 2026a).

**A correção é a mesma nas duas formas**, e não é reescrever o histórico nem reconstruir a imagem: nenhuma das duas operações alcança os clones, os *forks*, as cópias da imagem e os caches que já existem. A correção é **revogar a chave** no provedor e emitir outra; a limpeza do histórico, quando feita, é complementar e vem depois (GITHUB, 2026). A prevenção também é comum às duas: o segredo entra em tempo de execução, nunca em tempo de build, e nenhum `COPY` o alcança.

É o mesmo formato de lição que o versionamento de prompt da aula de *prompt engineering* estabeleceu — **o histórico guarda** —, agora com consequência de segurança em vez de comparabilidade.

### 7. O container é efêmero: onde mora o estado?

O sistema de arquivos de um container **nasce da imagem a cada execução**. O que o processo grava durante a execução vai para a camada gravável daquele container, e não para a imagem; com `--rm`, a camada gravável é removida assim que o processo termina. A execução seguinte, da mesma imagem, começa de um sistema de arquivos idêntico ao da anterior no início, e não ao da anterior no fim.

A demonstração usa a imagem do agente com o `ENTRYPOINT` substituído por um *shell*, gravando um arquivo em `/app`, gravável porque o processo roda como `root` (§4.2):

```bash
docker run --rm --entrypoint sh auditor-de-despesas:1.0.0 -c "date > /app/registro.txt; cat /app/registro.txt"
docker run --rm --entrypoint sh auditor-de-despesas:1.0.0 -c "cat /app/registro.txt"
```

A primeira execução imprime a data; a segunda falha com *No such file or directory*. As duas nasceram da mesma imagem, e a imagem não continha nada do que a primeira gravou.

O agente das duas primeiras formas não tem estado entre execuções — as despesas e a política são constantes no código —, e por isso o empacotamento não perde nada. O sistema construído ao longo do curso tem: o **índice** da aula de RAG e documentos, gravado em disco pelo banco vetorial, e a **memória** da aula de memória, escrita pelo agente a cada interação. A pergunta de arquitetura que o container efêmero impõe é **onde mora o estado**, e há três saídas, cada uma com preço:

| Saída | Como | Preço |
|---|---|---|
| **reconstruir** | o processo refaz o estado a partir da fonte, na partida; ou o estado é gerado no build e entra numa camada da imagem | na partida: cada execução paga o custo de embutir o corpus, em tempo e em tokens. No build: imagem maior, e uma imagem nova a cada atualização do corpus |
| **externalizar** | volume montado com `-v`, banco de dados ou serviço separado, que existem fora do ciclo de vida do container | um componente a mais, com disponibilidade, cópia de segurança e, no caso de banco ou serviço, conta própria |
| **aceitar a perda, e declarar isso** | nada é guardado entre execuções, e a documentação do sistema diz que nada é guardado | legítimo para o que é derivado; inaceitável para o que não é |

O **volume** é a forma mais direta de externalizar em Docker: um diretório gerido pelo Docker fora da camada gravável, montado num caminho do container e preservado entre execuções (DOCKER, 2026c). Com a mesma demonstração, o volume nomeado `estado-agente` montado em `/dados` faz o arquivo sobreviver:

```bash
docker run --rm -v estado-agente:/dados --entrypoint sh auditor-de-despesas:1.0.0 -c "date > /dados/registro.txt"
docker run --rm -v estado-agente:/dados --entrypoint sh auditor-de-despesas:1.0.0 -c "cat /dados/registro.txt"
```

O caminho `/dados` não existe na imagem, e o Docker o cria como ponto de montagem. O volume não é montado sobre `/app` por uma razão estrutural: um volume novo montado sobre um diretório da imagem recebe a cópia do conteúdo desse diretório, e nas execuções seguintes essa cópia — o código da imagem que o criou — esconderia o código de uma imagem nova (DOCKER, 2026c). O estado ocupa um caminho próprio, separado do que vem da imagem.

A terceira linha da tabela separa os dois estados do sistema do curso. O **índice** é derivado: existe um corpus, em outro lugar, a partir do qual ele é reconstruído, e perdê-lo custa tempo e tokens, mas não informação. A **memória não é derivada de nada**: ela registra o que ocorreu nas interações, e perdê-la é perder o que o sistema aprendeu, sem fonte de onde recuperar. Para o índice, as três saídas são defensáveis, e a reconstrução no build serve bem a um corpus estático. Para a memória, só a externalização serve, e a reconstrução no build não existe como opção, porque a memória é escrita durante a operação, depois que a imagem foi construída.

A razão de fundo é a imutabilidade. A imagem é imutável e o container é descartável por desenho, e é essa propriedade que permite trocar uma versão por outra sem cerimônia: parar o container da `1.0.0` e iniciar um da `1.1.0` não exige migrar nada que esteja dentro dele, porque nada que importe está dentro dele. A mesma propriedade obriga o estado a ter **endereço declarado** — um volume, um banco, um serviço, ou a declaração explícita de que se aceita perdê-lo. Um sistema cujo estado mora na camada gravável do container perde esse estado na primeira troca de versão.

#### 7.1 O banco como estado externalizado

A terceira forma, `03-agente-web-banco/`, aplica a segunda saída da tabela a um dado de negócio: a **política de reembolso** deixa de ser um dicionário no código e passa a ser a tabela `politica` de um banco **Postgres**. A política é um dado que muda por decisão de negócio, e não por *commit*; no código, cada mudança de teto exigiria uma imagem nova. No banco, ela muda sem reconstrução, e a imagem da API continua a mesma da segunda forma — o `Dockerfile` de `03-agente-web-banco/` repete os quatro blocos do de `02-agente-web/`, e a imagem não sabe que existe um banco. O endereço dele chega em tempo de execução, pela variável `DATABASE_URL`.

O **Docker Compose** é a ferramenta que declara, num arquivo YAML, um conjunto de containers que operam juntos — os **serviços** —, com as redes e os volumes de que dependem, e os sobe e derruba como uma unidade (DOCKER, 2026d). O `docker-compose.yml` da pasta declara dois serviços e um volume:

```text
  máquina hospedeira
  +-------------------------------------------------------------+
  |  :8000 (ports: "8000:8000")                                 |
  |    |                                                        |
  |  +-|----------------- rede do compose --------------------+ |
  |  | v                                                      | |
  |  | agente   auditor-de-despesas-banco:1.0.0 (build: .)    | |
  |  |          uvicorn :8000                                 | |
  |  |          DATABASE_URL=postgresql://...@banco:5432/...  | |
  |  |    |                                                   | |
  |  |    |  pelo nome do serviço                             | |
  |  |    v                                                   | |
  |  | banco    postgres:16.10, :5432, sem porta publicada    | |
  |  |          /var/lib/postgresql/data    <- volume dados   | |
  |  |          /docker-entrypoint-initdb.d <- ./banco (ro)   | |
  |  +--------------------------------------------------------+ |
  +-------------------------------------------------------------+
```

Um único comando, `docker compose up --build`, constrói a imagem da API e sobe os dois serviços. Cada elemento do arquivo responde a uma parte da pergunta sobre o estado:

- **O estado mora no volume nomeado `dados`**, montado em `/var/lib/postgresql/data`, o diretório de dados do Postgres. O container `banco` é tão descartável quanto o `agente`: recriá-lo não apaga a política, porque ela não está na camada gravável dele. `docker compose down` derruba os dois containers e preserva o volume; `docker compose down -v` derruba e **apaga** o volume.
- **O banco nasce de um script de inicialização.** A imagem oficial do Postgres executa, em ordem alfabética, os arquivos `.sql` e `.sh` de `/docker-entrypoint-initdb.d/` — e o compose monta ali a pasta `./banco`, somente leitura. O `01-politica.sql` cria a tabela `politica` (`categoria` como chave primária, `teto` em `NUMERIC(10, 2)` com `CHECK (teto > 0)`) e insere as quatro categorias. A execução ocorre **uma vez**, quando o diretório de dados está vazio (DOCKER HUB, 2026). Alterar o script depois não altera um banco que já existe; para que ele rode de novo, `docker compose down -v` apaga o volume. É a distinção de §7 entre o derivado e o não derivado, aplicada ao banco: o script é a fonte, e o banco só é derivado dela até a primeira alteração feita em operação.
- **A imagem do banco é fixada até o *patch*** (`postgres:16.10`), pelo mesmo motivo da imagem base da API (§4.2).
- **A API alcança o banco pelo nome do serviço.** O compose cria uma rede própria para os serviços do arquivo, e cada serviço é alcançável pelos demais pelo próprio nome (DOCKER, 2026e). Por isso o `DATABASE_URL` aponta para `banco:5432`, e não para `localhost` — dentro do container `agente`, `localhost` é ele mesmo, o fato de §5.2. O valor é montado no próprio compose a partir de `POSTGRES_USER`, `POSTGRES_PASSWORD` e `POSTGRES_DB`, definidos no `.env` da pasta, que também alimentam a criação do banco.
- **O banco não publica porta.** O serviço `banco` não tem `ports`: o Postgres é alcançável apenas de dentro da rede do compose. A API precisa dele; a máquina hospedeira, não. Só o `agente` publica a porta 8000.
- **"De pé" não é "pronto".** O processo do Postgres inicia antes de aceitar conexões, e na primeira subida ainda executa o script de inicialização. O `healthcheck` do serviço `banco` roda `pg_isready` a cada 2 s, até 15 tentativas, e o `depends_on` do `agente` com `condition: service_healthy` só inicia a API quando o banco é declarado saudável (DOCKER, 2026d). No teste, `$${POSTGRES_USER}` leva `$$` para que o compose não substitua a variável: ela é resolvida pelo *shell* dentro do container.

Do lado do código, a ferramenta `consultar_politica` do `agente.py` faz uma consulta **parametrizada**:

```python
linha = conexao.execute("SELECT teto FROM politica WHERE categoria = %s", (categoria,)).fetchone()
```

A categoria é um argumento que o **modelo** escolhe ao chamar a ferramenta, e o modelo não é fonte confiável: a concatenação do texto na SQL abriria injeção de SQL. Com o parâmetro, o driver envia o valor separado do comando, e nenhum texto vindo do modelo é interpretado como SQL. Categoria inexistente devolve o erro com a lista das categorias válidas, lida da própria tabela.

O `GET /saude` da terceira forma acrescenta o campo `banco`, que diz se o banco responde a um `SELECT 1`, ainda sem chamar o modelo. A resposta tem a forma abaixo — saída **ilustrativa**, derivada do código, e não de uma execução do compose:

```json
{"ok": true, "banco": true, "modelo": "mistral/mistral-small-latest"}
```

O preço da externalização, previsto na tabela de §7, aparece por inteiro: um componente a mais, com versão própria, credenciais próprias, um volume do qual se faz cópia de segurança e uma ordem de inicialização a respeitar. Em troca, o estado tem endereço declarado, e a imagem da API pode ser trocada por outra sem que a política se perca.

### 8. Quando a execução é longa

A API de §5.3 já atende por rede, na forma mais simples: requisição e resposta síncronas. Ela funciona porque o agente da aula responde em segundos. Quando a execução se alonga, a forma de entrega do resultado passa a ser uma decisão de arquitetura, e a forma mais simples deixa de servir.

**Requisição-resposta síncrona não serve a um agente de execução longa.** Uma execução de doze passos, com chamadas ao modelo e a ferramentas, leva minutos, e balanceadores, *proxies* reversos e *gateways* encerram conexões ociosas em dezenas de segundos. O cliente recebe um erro de tempo esgotado enquanto o agente continua a executar do outro lado — e a consumir tokens por um resultado que ninguém recebe. É o **tempo de parede**, tratado na aula de arquitetura de agentes como uma das moedas do orçamento, reaparecendo como limite de infraestrutura.

Os três padrões que resolvem o problema têm o mesmo princípio: **desacoplar a aceitação do pedido da entrega do resultado**.

- **Fila com trabalhador** (*queue + worker*): o pedido entra numa fila e é respondido de imediato com um identificador; um processo separado consome a fila e executa; o cliente consulta o estado pelo identificador (*polling*).
- ***Webhook***: o cliente registra um endereço ao fazer o pedido, e o serviço faz uma requisição a esse endereço quando a execução termina.
- **SSE** (*Server-Sent Events*): o serviço mantém a conexão aberta e envia eventos de progresso por ela, num fluxo unidirecional do servidor para o cliente, o que mantém a conexão ativa enquanto a execução avança.

O critério de escolha decorre de duas variáveis: a **duração** da execução e a **presença de alguém esperando** a resposta.

| Padrão | Quando |
|---|---|
| **fila com trabalhador** | execução longa, cliente que pode voltar depois |
| ***webhook*** | o cliente é outro sistema, com endereço próprio para receber o resultado |
| **SSE** | uma pessoa diante da interface, acompanhando o progresso |

Os padrões não se excluem: um mesmo serviço pode aceitar pedidos numa fila e oferecer o progresso de cada um por SSE. São os mesmos três modos de entrega que o protocolo A2A padroniza — *polling*, *streaming* e notificação *push* (A2A PROJECT, 2026) —, retomados na aula de sistemas multiagente, na [nota sobre o contrato e o protocolo](../../aula-16-sistemas-multiagente/notas-de-aula/02-do-contrato-ao-protocolo.md).

O que viabiliza os três padrões é a **execução durável**: a propriedade de uma execução cujo estado é gravado a cada passo, de modo que ela pode ser interrompida e retomada do último ponto gravado, em outro processo (LANGCHAIN, 2026). O *checkpoint*, introduzido na aula de memória e materializado no *checkpointer* do LangGraph na aula de *frameworks* e orquestração, deixa de ser conveniência e passa a ser requisito, por três razões independentes: o container que executa pode ser substituído no meio da execução (§7), por troca de versão ou por falha; toda troca de versão encontra execuções em andamento; e uma aprovação humana — o término por humano modelado na aula de arquitetura de agentes — pode chegar horas depois, quando o processo que suspendeu a execução já não existe. Um *checkpoint* gravado na camada gravável do container não cumpre o requisito: ele é estado, e a §7 se aplica a ele — o lugar dele é um armazenamento externo, como o banco de §7.1.

---

## Fontes e leituras

A2A PROJECT. **Agent2Agent (A2A) Protocol Specification**. Linux Foundation, 2026. Disponível em: https://a2a-protocol.org/latest/specification/. — os modos de entrega *polling*, *streaming* e *push*.

DOCKER. **Dockerfile reference**. 2026a. Disponível em: https://docs.docker.com/reference/dockerfile/. — instruções, camadas, forma vetorial e forma de *shell* do `ENTRYPOINT`, `EXPOSE`, `ARG` e a *tag* `latest`.

DOCKER. **Build context**. 2026b. Disponível em: https://docs.docker.com/build/concepts/context/. — contexto de build e `.dockerignore`.

DOCKER. **Volumes**. 2026c. Disponível em: https://docs.docker.com/engine/storage/volumes/. — volumes nomeados e preenchimento de um volume novo com o conteúdo da imagem.

DOCKER. **Compose file reference**. 2026d. Disponível em: https://docs.docker.com/reference/compose-file/. — serviços, volumes, `healthcheck` e `depends_on` com `condition: service_healthy`.

DOCKER. **Networking in Compose**. 2026e. Disponível em: https://docs.docker.com/compose/how-tos/networking/. — a rede padrão do projeto e a resolução do nome do serviço.

DOCKER. **Building best practices**. Disponível em: https://docs.docker.com/build/building/best-practices/. — ordem das instruções e cache de build, imagem base fixada, `.dockerignore`.

DOCKER. **docker image history**. Disponível em: https://docs.docker.com/reference/cli/docker/image/history/. — listagem das camadas de uma imagem.

DOCKER HUB. **postgres: Docker Official Image**. 2026. Disponível em: https://hub.docker.com/_/postgres. — variáveis `POSTGRES_*`, diretório de dados e scripts de inicialização em `/docker-entrypoint-initdb.d/`.

GITHUB. **Removing sensitive data from a repository**. GitHub Docs, 2026. Disponível em: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository. — por que a revogação da credencial precede a limpeza do histórico.

LANGCHAIN. **Durable execution**. LangGraph Docs, 2026. Disponível em: https://docs.langchain.com/oss/python/langgraph/durable-execution. — persistência e retomada de execuções.

OLLAMA. **FAQ**. 2026. Disponível em: https://docs.ollama.com/faq. — `OLLAMA_HOST` e a exposição do servidor na rede.

**Material da disciplina.** [Introdução ao Docker](../../aula-00-sobre-o-curso/notas-de-aula/02-docker.md), na aula sobre o curso — imagem, container e `Dockerfile`. [MLOps](02-mlops.md), nesta aula — o tópico de *deploy* e *serving* (§10) e a linhagem. [O prompt como código](../../aula-03-prompt-engineering/notas-de-aula/03-prompt-como-codigo.md), em *prompt engineering* — o versionamento e o histórico que guarda. [Confiabilidade](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md), em arquitetura de agentes — o orçamento em quatro moedas e as formas de terminação. [Bancos vetoriais](../../aula-06-embeddings-e-rag/notas-de-aula/04-bancos-vetoriais.md), em *embeddings* e RAG — o índice persistido em disco. [O agente e o estado](../../aula-08-memoria/notas-de-aula/01-o-agente-e-o-estado.md), em memória — o *checkpoint* (§8). [Esquecer e proteger](../../aula-08-memoria/notas-de-aula/04-esquecer-e-proteger.md), em memória — a memória que não é derivada de nada. [*Frameworks* e orquestração](../../aula-09-frameworks-e-orquestracao/notas-de-aula/01-frameworks-e-orquestracao.md) — *checkpointer*, *store* e o humano no meio do grafo. [Observabilidade](../../aula-14-mlops-observabilidade/notas-de-aula/02-observabilidade.md) — a decomposição por versão. [Cadeia, imagem, rede, e o fecho](../../aula-15-seguranca/notas-de-aula/03-cadeia-rede-e-o-fecho.md), em segurança — o `Dockerfile` lido como superfície, inclusive o processo como `root`. [Do contrato ao protocolo](../../aula-16-sistemas-multiagente/notas-de-aula/02-do-contrato-ao-protocolo.md), em sistemas multiagente — o A2A e os modos de entrega.
