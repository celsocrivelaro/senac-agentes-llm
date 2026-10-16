# Exercício 12 — O case em imagem

## Contexto

A Parte 3 do trabalho cobra **MLOps e observabilidade**. Este exercício
entrega a parte de MLOps que cabe nesta aula: o sistema do grupo empacotado
numa imagem Docker, construída e executada localmente, que qualquer máquina
com Docker executa igual. Não há plataforma de nuvem nem serviço por rede: o
container recebe o objetivo pela linha de comando, executa o agente e termina.

Até aqui, o sistema roda na máquina de quem o escreveu, com o Python, os
pacotes e as variáveis daquela máquina. A imagem carrega o código e tudo de
que ele depende; ficam fora dela o **segredo**, entregue em tempo de
execução, o **modelo**, que é um provedor de API ou um servidor fora do
container, e o **estado**, que não sobrevive ao container e, quando precisa
sobreviver, passa a morar num volume ou num banco ao lado dele. Cada um tem
um requisito abaixo.

A imagem construída aqui é a mesma que a aula de observabilidade instrumenta.

> **Questão a ser respondida ao final:** o que o sistema do grupo grava
> durante uma execução, e onde cada uma dessas coisas passa a morar quando o
> container é descartado ao terminar?

## Objetivo

Construir, no repositório do trabalho:

1. a **imagem** do sistema do grupo, com dependências fixadas e a ordem das
   camadas justificada;
2. a **construção e a execução** do container, reproduzíveis por quem nunca
   viu o projeto, com o segredo entregue em tempo de execução;
3. a **verificação do segredo**, no histórico do git e nas camadas da imagem;
4. a **decisão sobre onde mora o estado**, demonstrada.

## Ambiente

A VM da disciplina não tem Docker. A construção e a execução da imagem exigem
uma máquina com Docker instalado (ver a nota sobre Docker, na aula sobre o
curso): a de um integrante do grupo, a de um colega ou a do laboratório. O
`Dockerfile`, o `.dockerignore` e o `.gitignore` são escritos e versionados no
repositório do trabalho independentemente de onde a imagem for construída.

## Requisitos

### 1. A imagem do sistema do grupo

Escreva o `Dockerfile` do sistema, com a pasta do sistema como contexto de
build, organizado nos **quatro blocos** dos `Dockerfile` do código da aula,
cada um com um comentário em linha própria que o justifica:

1. **imagem base com versão fixada até o *patch***, seguida do `WORKDIR`.
   Uma etiqueta móvel como `python:3.11-slim` produz imagens diferentes em
   dias diferentes a partir da mesma linha, e não serve;
2. **dependências instaladas antes do código**: o arquivo de dependências
   copiado sozinho, seguido da instalação. A camada de instalação é a mais
   cara, e o código é o que muda a cada *commit*. O arquivo de dependências
   é **próprio da imagem**, com versão fixada e apenas o que o sistema
   importa — e não o `requirements.txt` da disciplina inteira;
3. **`COPY` por nome** dos arquivos e pastas que o sistema usa — e não
   `COPY . .`, que deixaria só o `.dockerignore` entre o `.env` e a imagem;
4. **o comando**: `ENTRYPOINT` que fixa o programa e recebe o objetivo como
   argumento do `docker run`.

Justifique por escrito a **ordem das instruções**: para cada camada, com que
frequência ela muda e o que é reconstruído quando muda. Demonstre a
justificativa: altere uma linha do código, reconstrua, e copie a saída do
build em que a camada de instalação aparece como reaproveitada do cache.

### 2. Construir e executar

Construa e execute o sistema em container:

```bash
docker build -t <nome-do-sistema>:<tag> .
docker run --rm --env-file .env <nome-do-sistema>:<tag> "<objetivo>"
```

O critério é a **reprodutibilidade por quem nunca viu o projeto**: a partir de
um clone do repositório e de um `.env` preenchido segundo o `.env.example`,
os dois comandos acima, sem nenhum passo manual entre eles, produzem a
execução. Peça a alguém de fora do grupo que siga o `README.md` numa máquina
com Docker e registre o resultado, ou o passo em que travou.

O container precisa **alcançar o modelo**:

- com **provedor de API**, basta a chave entregue pelo `--env-file`;
- com o **Ollama** rodando na máquina hospedeira, `localhost` dentro do
  container é o próprio container. Sobrescreva o endereço no `docker run`
  com `host.docker.internal` e documente o comando no `README.md`.

Declare qual dos dois o sistema usa e copie a saída completa de uma
execução — entrada e saída reais.

### 3. O segredo

O `.env` fica fora do repositório e fora da imagem:

- no **`.gitignore`**: `.env`, com exceção explícita para o `.env.example`;
- no **`.dockerignore`**: `.env`, `.git`, `.venv`, `__pycache__` e todo
  estado local do sistema (índice, memória, histórico). O que não entra no
  contexto de build não pode ser copiado para a imagem por nenhuma
  instrução. Justifique cada linha.

Apagar o arquivo não apaga o segredo, nem no git nem na imagem. Verifique as
duas formas de vazamento e reporte as saídas.

No git, no repositório do trabalho:

```bash
git log --all -- .env
git log --all -S "_API_KEY=" --oneline
```

Na imagem:

```bash
docker history --no-trunc <nome-do-sistema>:<tag>
```

e a confirmação de que nenhuma camada vem de um `COPY` ou `ADD` que alcance o
`.env`.

Se alguma das verificações não vier vazia, a correção **não é** reescrever o
histórico nem reconstruir a imagem — nenhuma das duas alcança os clones e as
cópias que já existem — e sim **revogar a chave** no provedor e gerar outra.
Relate o que foi encontrado e o que foi feito.

### 4. Onde mora o estado

O sistema de arquivos do container nasce da imagem a cada execução, e o que o
processo grava dentro dele some quando ele termina — com `--rm`,
imediatamente.

Liste **tudo o que o sistema grava** durante uma execução — índice ou base de
conhecimento, memória, histórico de conversas, *checkpoints*, arquivos de
saída — e, para cada item, registre a decisão:

| Componente | Onde é gravado hoje | Derivado de quê | Decisão |
|---|---|---|---|
| índice / base de conhecimento | | | reconstruir · externalizar · aceitar a perda |
| memória | | | |
| histórico | | | |

- **reconstruir**: na partida ou no build. Diga em qual dos dois, e quanto
  tempo leva;
- **externalizar**: volume montado com `-v`, ou banco de dados em container
  próprio, subido ao lado do sistema por `docker compose`. Diga qual, e o
  comando que o monta ou sobe;
- **aceitar a perda**: legítimo para o que é derivado de outra fonte; para o
  que não é derivado de nada, como a memória, exige justificativa explícita
  e a declaração da perda no `README.md`.

A externalização num banco segue a forma `03-agente-web-banco/` do código da
aula, em que a política de reembolso mora numa tabela do Postgres. O
`docker-compose.yml` dela reúne o que essa saída exige, e cada item é
cobrado se o grupo a escolher:

- o banco em **imagem oficial com versão fixada até o *patch***;
- o **esquema e os dados iniciais num script de inicialização**, montado em
  `/docker-entrypoint-initdb.d/` — executado uma vez, com o volume vazio;
- os dados num **volume nomeado**, fora do container;
- o sistema alcançando o banco pelo **nome do serviço**, e não por
  `localhost`, com o endereço entregue em tempo de execução por variável de
  ambiente;
- um **`healthcheck`** no banco e `depends_on` com
  `condition: service_healthy` no sistema, para que ele só parta com o banco
  aceitando conexões.

**Demonstre** cada item externalizado: duas execuções seguidas, em que a
segunda encontra o que a primeira gravou — com `docker run --rm`, duas
execuções do container; com `docker compose`, um `down` sem `-v` entre as
duas subidas. Copie as duas saídas. Se o sistema não grava nada, declare
isso e mostre o trecho do código que o confirma.

## O que deve sair na tela

```
$ docker build -t <sistema>:<tag> .
...
 => CACHED [n/m] RUN pip install -r requirements.txt       (no rebuild)
...

$ docker run --rm --env-file .env <sistema>:<tag> "<objetivo>"
<sistema> · <provedor>/<modelo>
objetivo: <objetivo>

<resposta do sistema>

IMAGEM
  base ............... <imagem:tag fixada até o patch>
  nome ............... <sistema>:<tag>
  tamanho ............ <n> MB
  modelo alcançado ... <provedor de API | Ollama via host.docker.internal>

SEGREDO
  git log --all -- .env ............ <vazio | n commits>
  git log --all -S "_API_KEY=" ..... <vazio | n commits>
  .env em camada da imagem ......... <não | sim>
  ação tomada ...................... <nenhuma | chave revogada>

ESTADO
  <componente> ... <onde é gravado> ... <decisão>
  endereço ........ <volume -v | serviço do compose e volume nomeado | nenhum>
  execução 1 grava <...>   execução 2 encontra <...>
```

## Desafios opcionais

**A. Uma imagem menor.** Reduza o tamanho da imagem — retirando do arquivo de
dependências o que o sistema não importa, ou com *build* em múltiplos
estágios, em que a instalação acontece num estágio e só o resultado é copiado
para o estágio final. Meça antes e depois: tamanho em MB (`docker images`) e
tempo do build sem cache (`docker build --no-cache`). Reporte o que saiu da
imagem e se alguma funcionalidade dependia disso.

**B. Uma decisão de estado alternativa.** Escolha um componente do requisito 4
e implemente a decisão que **não** foi a escolhida — por exemplo, reconstruir
no build o índice que hoje é externalizado em volume, ou externalizar num
banco a memória que hoje vai para um arquivo. Compare as duas por tempo de
partida, tamanho da imagem e o que acontece quando o corpus ou a memória
mudam, e justifique qual fica.

**C. O sistema como API.** Empacote o sistema também como API, nas formas
`02-agente-web/` e `03-agente-web-banco/` do código da aula: uma API que
recebe a pergunta por HTTP e devolve a resposta, numa imagem própria, com
`uvicorn` escutando em `--host 0.0.0.0` e a porta publicada com `-p` no
`docker run` — ou em `ports` no `docker-compose.yml`, quando o estado mora
num banco ao lado. Registre a
saída de `GET /saude` e de uma chamada real ao *endpoint* de pergunta, meça o
tempo da resposta e diga se a forma síncrona serve ao sistema do grupo — ou se
a duração das execuções pediria fila, *webhook* ou SSE.

## Entrega

No repositório do trabalho:

- o `Dockerfile`, o arquivo de dependências da imagem, o `.dockerignore`, o
  `.gitignore` e o `.env.example`, com **parâmetros e justificativas em
  comentário** no próprio arquivo;
- no `README.md`: **como construir**, **como executar** — inclusive com o
  Ollama, se for o caso — e **onde mora o estado**, com um exemplo completo
  de entrada e saída copiado de uma execução real;
- em `aula12-respostas.md`:
  1. os comandos `docker build` e `docker run` e o exemplo completo de uso,
     com entrada e saída reais, e o registro da reprodução por alguém de fora
     do grupo;
  2. o `Dockerfile`, com a justificativa da ordem das instruções e a saída do
     rebuild com a camada de dependências no cache; o arquivo de dependências
     da imagem; e a lista do `.dockerignore`, justificada linha a linha;
  3. a verificação do segredo: as saídas do `git log` e do `docker history`,
     e o que foi feito se alguma não veio vazia;
  4. a tabela de onde mora o estado, com as duas execuções que demonstram o
     que foi externalizado e, se o estado foi para um banco, o
     `docker-compose.yml` e o script de inicialização.

**Critérios**: a imagem executa em outra máquina com Docker sem passo manual ·
a imagem não carrega segredo · o `Dockerfile` tem os quatro blocos e a ordem
das camadas está justificada · as dependências estão fixadas · o estado tem
endereço declarado.

> **Onde isto reaparece:** a **Parte 3 do trabalho** pede `docs/producao.md`.
> O `aula12-respostas.md` é o rascunho da parte de empacotamento dele — na
> entrega, consolidado em `docs/`, junto com o que a aula de observabilidade
> acrescenta.

## Dicas

- O código da aula tem o mesmo agente em três formas, cada uma com o seu
  `Dockerfile` de quatro blocos: `01-agente-cli/` (linha de comando, a forma
  pedida nos requisitos), `02-agente-web/` (API, a do desafio C) e
  `03-agente-web-banco/` (a API com a política num Postgres subido por
  `docker compose`, referência para externalizar o estado num banco e para
  o desafio C). As três servem de referência.
- O script de inicialização do Postgres roda **uma vez**, com o volume
  vazio. Mudar o script depois não altera o banco existente:
  `docker compose down -v` apaga o volume e faz o script rodar de novo. O
  mesmo `-v` usado por engano entre as duas execuções do requisito 4 apaga
  justamente o que se queria demonstrar.
- Dentro da rede do compose, `localhost` é o próprio container; o banco é
  alcançado pelo nome do serviço (`banco:5432` na referência). Sem
  `healthcheck` e `service_healthy`, o sistema pode partir antes de o banco
  aceitar conexões e falhar na primeira consulta.
- O primeiro build baixa a imagem base e instala todas as dependências, e
  depende da rede. Faça-o antes do encontro, na máquina em que a imagem será
  construída.
- Comentário no `Dockerfile` só vale em linha própria: um `#` no fim de uma
  instrução vira argumento dela.
- Se o `docker run` falha com erro de importação, o `COPY` deixou de fora um
  módulo, ou o arquivo de dependências da imagem deixou de fora um pacote. A
  lista explícita cobra esse cuidado, e é por isso que ela protege o `.env`.
- O requisito 4 começa pela execução, e não pela decisão: rode o sistema no
  container, rode de novo, e observe o que faltou na segunda vez.
- Um `RUN rm .env` depois de um `COPY .env` não remove a chave da imagem:
  acrescenta uma camada que a esconde. O `docker history` mostra as duas.
