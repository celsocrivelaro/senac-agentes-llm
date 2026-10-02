# Exercício 10 — Um servidor MCP de CNPJ e CEP, e o cliente que o consome

## Contexto

A aula mostrou os dois lados do protocolo: o **servidor**, que publica
ferramentas para quem quiser consumi-las, e o **cliente**, que abre a sessão,
descobre o que o servidor oferece e faz as chamadas. Neste exercício você
escreve os dois, sobre uma API pública e real: a
[BrasilAPI](https://brasilapi.com.br/docs).

O servidor embrulha duas consultas da BrasilAPI — **CNPJ** e **CEP** — como
ferramentas MCP. O cliente recebe um CNPJ e **encadeia** as duas: consulta a
empresa, extrai o CEP dela da resposta e, com ele, consulta o endereço e as
coordenadas.

```
                       ┌──────────────────────────┐        ┌───────────────┐
   CNPJ ──► cliente ──►│ servidor MCP             │──────► │  BrasilAPI    │
            MCP        │  ├ consultar_cnpj(cnpj)  │  HTTP  │  /cnpj/v1     │
              ▲        │  └ consultar_cep(cep)    │        │  /cep/v2      │
              │        └──────────────────────────┘        └───────────────┘
              │
              └── 1. consultar_cnpj  →  razão social, CEP
                  2. consultar_cep   →  endereço, latitude, longitude
```

> **Questão a ser respondida ao final:** o cliente que você escreveu conhece a
> BrasilAPI? O que precisaria mudar nele se o servidor trocasse a BrasilAPI por
> outra fonte de dados?

## Objetivo

Implementar:

1. um **servidor MCP** com as ferramentas `consultar_cnpj` e `consultar_cep`;
2. um **cliente MCP** que receba um CNPJ, chame `consultar_cnpj`, use o CEP da
   empresa para chamar `consultar_cep` e apresente o resultado.

## As duas consultas da BrasilAPI

| Ferramenta | Endpoint | Campos que interessam |
|---|---|---|
| `consultar_cnpj` | `GET https://brasilapi.com.br/api/cnpj/v1/{cnpj}` | `cnpj`, `razao_social`, `cep`, `logradouro`, `numero`, `complemento`, `bairro`, `municipio`, `uf` |
| `consultar_cep` | `GET https://brasilapi.com.br/api/cep/v2/{cep}` | `cep`, `street`, `neighborhood`, `city`, `state`, `location.coordinates.latitude`, `location.coordinates.longitude` |

Documentação: [CNPJ](https://brasilapi.com.br/docs#tag/CNPJ) e
[CEP](https://brasilapi.com.br/docs#tag/CEP). Os dois endpoints recebem **só
dígitos**: 14 para o CNPJ e 8 para o CEP. Não exigem chave.

Duas observações sobre o que volta:

- a resposta do CNPJ traz o CEP **sem hífen** (`"01311902"`), pronto para a
  segunda consulta;
- as coordenadas do CEP **nem sempre existem**: para alguns CEPs,
  `location.coordinates` vem vazio. O cliente precisa lidar com isso.

## Requisitos

### 1. O servidor

Escreva o servidor com o FastMCP, como o `00-servidor.py` da aula, rodando
**sozinho**, num terminal próprio, por Streamable HTTP.

- **Duas ferramentas**, `consultar_cnpj(cnpj)` e `consultar_cep(cep)`, cada uma
  fazendo a chamada HTTP correspondente à BrasilAPI.
- **Normalize a entrada**: aceite `19.131.243/0001-97` e `01311-902`, e mande à
  API só os dígitos.
- **Devolva só o que interessa**, e não o JSON inteiro da BrasilAPI. A resposta
  do CNPJ tem dezenas de campos (sócios, CNAEs, regime tributário…), e tudo o
  que a ferramenta devolve é texto que o cliente — ou um modelo — vai ter de
  carregar. Justifique, em comentário, os campos que você manteve.

### 2. A descrição, tratada como prompt público

Cada ferramenta precisa de uma descrição que responda três coisas:

- o que a ferramenta faz;
- o **formato exato** do argumento — quantos dígitos, se aceita pontuação;
- **o que ela não faz** — por exemplo, `consultar_cnpj` não devolve
  coordenadas, e `consultar_cep` não busca por nome de rua.

O seu cliente não lê a descrição; um agente leria. Escreva-a para ele.

### 3. O erro, classificado

Toda ferramenta do servidor deve distinguir:

| Natureza | Exemplo | Como volta |
|---|---|---|
| erro de **domínio** | CNPJ com 13 dígitos; CNPJ ou CEP inexistente (a BrasilAPI responde 404) | como **conteúdo**, dizendo o que errou, qual era o formato certo e o que fazer agora |
| erro de **infraestrutura** | BrasilAPI fora do ar, *timeout*, resposta 5xx | como **falha**, tratada pelo código do cliente |

Defina um *timeout* para a chamada HTTP. Sem ele, uma BrasilAPI lenta prende o
servidor, e o cliente junto.

### 4. O cliente

Escreva o cliente com o `ClientSession` do SDK, como o `02-agente-com-mcp.py`.
Ele deve:

1. receber o CNPJ pela linha de comando:
   `python cliente_cnpj.py 19.131.243/0001-97`;
2. abrir a sessão e **listar as ferramentas** do servidor, falhando com mensagem
   clara se `consultar_cnpj` ou `consultar_cep` não estiverem lá;
3. chamar `consultar_cnpj` e extrair da resposta a razão social e o CEP;
4. chamar `consultar_cep` **com o CEP da empresa**;
5. apresentar o resultado no formato da seção seguinte. O **endereço** sai da
   resposta do CEP — rua, bairro, cidade e UF —, e não da do CNPJ: é a segunda
   consulta que justifica existir.

O encadeamento é do **cliente**: o servidor não sabe que as duas ferramentas
são usadas juntas, e não deve saber. `consultar_cep` precisa continuar servindo
a quem só tem um CEP na mão.

Quando a primeira consulta devolver erro de domínio, o cliente **não** faz a
segunda: mostra o erro e termina.

### 5. O carimbo

Registre, no cabeçalho do servidor, a **revisão da especificação do protocolo**
e a **versão do SDK** (`mcp`) usadas. Num servidor de terceiro, quem muda esses
números é outra pessoa — sem aviso, e sem que a resposta deixe de ser
sintaticamente válida.

## O que deve sair na tela

```
$ python cliente_cnpj.py 19.131.243/0001-97

SERVIDOR: <nome>   (ferramentas: consultar_cnpj, consultar_cep)

CNPJ ............ 19.131.243/0001-97
Razão social .... OPEN KNOWLEDGE BRASIL
CEP ............. 01311-902
Endereço ........ Avenida Paulista 37 — Bela Vista, São Paulo/SP
Lat / Long ...... -23.5475 / -46.63611
```

Quando o CEP não tiver coordenadas:

```
Lat / Long ...... não disponível para este CEP
```

Quando o CNPJ for inválido ou não existir, a mensagem de erro de domínio que o
**servidor** devolveu, e nenhuma linha de endereço.

Teste com pelo menos três CNPJs: um válido com coordenadas, um com formato
inválido e um com formato válido que não exista.

## Desafios opcionais

**A. O mesmo servidor, consumido por um agente.** Conecte um agente — o laço do
`02-agente-com-mcp.py`, trocando o servidor da aula pelo seu — ao seu
servidor e pergunte *"onde fica a empresa do CNPJ 19.131.243/0001-97?"*. Agora
quem decide encadear as duas ferramentas é o **modelo**, guiado só pelas
descrições. Ele fez as duas chamadas? Na ordem certa? Registre quantas chamadas
foram feitas e compare com o seu cliente, que faz sempre exatamente duas.

**B. Um recurso.** Exponha como **recurso** algo que a aplicação sempre
precisa e que não depende de pergunta nenhuma — por exemplo, a lista de UFs com
o nome por extenso, para o cliente escrever *São Paulo (SP)*. Justifique por que
aquilo é recurso, e não ferramenta.

## Entrega

- o servidor e o cliente, com **parâmetros e justificativas no código** — sem
  relatório à parte;
- a saída do cliente para os três CNPJs de teste, colada no fim do arquivo do
  cliente, em comentário;
- a resposta à questão do início, em comentário no cabeçalho do cliente.

## Dicas

- `requests` e `httpx` já estão no `requirements.txt` da disciplina. A
  BrasilAPI responde em JSON; `resposta.json()` resolve.
- A BrasilAPI tem limite de requisições. Durante o desenvolvimento, teste com
  poucos CNPJs, e não num laço.
- Formate o CNPJ e o CEP com pontuação **só na hora de imprimir**. Entre o
  cliente e o servidor, e entre o servidor e a API, circulam só dígitos.
- Se o cliente tiver uma linha com `brasilapi.com.br`, releia o Requisito 4.
