# Exercício 6 — A base de conhecimento do seu trabalho

## Contexto

A Aula 05 fez você desenhar a arquitetura do seu agente. Faltou nela uma decisão que aquele desenho não cobre: **de onde o agente tira o que ele sabe.**

Um modelo de linguagem sabe muita coisa, e nada do que é seu. Ele não conhece a política interna da sua empresa, nem o catálogo do seu cliente, nem os processos que ninguém publicou. Essa informação existe em documentos, e é ela que esta aula ensina a recuperar.

O exercício é decidir **qual informação, de onde, e cortada como** — no papel, antes da primeira linha de código.

> **Não se escreve código aqui.** A entrega é um texto.
>
> Se você quiser sentir no código o que cada decisão cobra antes de decidir, o [exercicio_06.md](exercicio_06.md) é o par prático: lá você indexa um regulamento dado e mede o `recall@k` das três estratégias de corte.

---

## O que responder

Quatro perguntas. Responda na ordem — cada uma depende da anterior.

### 1. Qual informação especializada o agente precisa, e por que ela não está no modelo

Liste o conhecimento que o seu agente precisa ter e que **nenhum modelo de linguagem tem**.

Para cada item, a segunda metade da pergunta é a que vale: **por que este dado precisa vir de fora?** Três respostas são aceitáveis, e vale saber qual é a sua:

| Razão | Exemplo |
|---|---|
| **é privado** | política interna, contrato, histórico de cliente |
| **é recente demais** | mudou depois do treinamento do modelo |
| **é específico demais** | existe, mas o modelo erra os detalhes — número de artigo, código de peça, exceção rara |

> **Cuidado com o terceiro.** Um modelo frequentemente *parece* saber coisas específicas e erra nos detalhes, com a mesma confiança de quando acerta. Se você não tem certeza de que ele sabe, teste antes de decidir: pergunte a ele e confira a resposta contra a fonte.

E o inverso também conta: **o que o seu agente precisa saber que o modelo já sabe?** Indexar o que ele já domina é pagar por contexto que não acrescenta nada.

### 2. Onde esses dados estão, e em que estado

Para cada fonte identificada em (1):

- **Onde ela vive**: arquivo em pasta compartilhada, página de intranet, sistema com API, banco de dados, e-mail, cabeça de alguém?
- **Em que formato**: PDF, Word, HTML, planilha, registro de banco?
- **Quem é o dono**, e com que frequência muda;
- **Se você tem acesso** — e, se não tem, como pretende conseguir.

> **Esta pergunta costuma ser a que mata o case**, e é melhor que ela mate agora. Um agente que depende de um dado ao qual você não tem acesso não vai existir, por melhor que seja a arquitetura.

Se alguma fonte for **PDF escaneado**, marque: extrair texto dali é um problema próprio, e é assunto da Aula 07.

### 3. O que vai para o índice — e o que não vai

Nem tudo que existe precisa ser indexado, e esta é uma decisão de projeto, não uma consequência.

- **O que entra**: quais documentos, e qual o volume aproximado — número de arquivos, de páginas, de registros;
- **Quantos chunks isso deve dar**, na ordem de grandeza. Dezenas? Milhares? Milhões?
- **O que fica de fora**, e por quê;
- **O que se resolve por consulta estruturada** em vez de por busca semântica.

> **O último item é o mais importante da pergunta.** A [nota 04](../notas-de-aula/04-bancos-vetoriais.md) mostrou que faixa de valor, data e identificador se resolvem com filtro de metadado — `==` e `<=` —, e não com cosseno. Se o seu agente precisa responder *"qual o saldo do cliente X"*, isso é consulta a banco, não recuperação por similaridade. Separe as duas agora.

E a estimativa de volume tem uma consequência direta: com dezenas ou centenas de chunks, **você não precisa de banco vetorial nenhum**. A nota 04 mediu isso — em 28 vetores o banco perde para uma matriz de numpy. Diga o que a sua escala pede.

### 4. A estratégia de chunking

Para cada tipo de documento, **como ele será cortado e por quê**.

A [nota 02](../notas-de-aula/02-chunking-e-a-medida-da-busca.md) mediu três estratégias sobre um documento normativo e a estrutura venceu — mas aquele documento **tinha** estrutura. Para cada uma das suas fontes:

- **A unidade natural existe?** Artigo, seção, item de FAQ, linha de planilha, mensagem?
- **Se existe**, corte por ela. Se não existe, por contagem de caracteres — e diga qual tamanho e por quê;
- **O chunk faz sentido sozinho?** Este é o critério único. Um parágrafo que diga *"o limite de que trata o item anterior"* é inútil isolado, e a correção é herdar o cabeçalho;
- **Que metadado cada chunk carrega**, além do texto — e aqui reencontre a pergunta (3): é o metadado que permite o filtro.

> **Documentos diferentes pedem cortes diferentes.** Uma base com FAQ e contratos não tem uma estratégia; tem duas. Dizer isso explicitamente vale mais que escolher uma e forçá-la sobre tudo.

---

## Entrega

Um documento em `exercicios/aula-06-base-de-conhecimento.md`, no repositório do trabalho.

Texto corrido, com as quatro perguntas como seções. Tabelas onde couberem. **Duas a quatro páginas** — se passar muito disso, provavelmente há descrição de fonte onde deveria haver decisão.

Estas escolhas vão mudar quando encontrarem o corpus real — é esperado, e é o mesmo que aconteceu com a arquitetura da Aula 05. Vale versionar este documento em vez de sobrescrevê-lo.

> **Onde isto reaparece:** a **Parte 2 do trabalho** pede `docs/rag.md`, com a versão que sobreviveu e o que mudou. O que você escreve aqui é o rascunho dele.

---

## Critérios

| | |
|---|---|
| **Justificou a necessidade** | cada fonte tem uma razão explícita para não vir do modelo |
| **Verificou o acesso** | não há fonte listada que você não saiba como obter |
| **Separou busca de consulta** | o que é filtro está marcado como filtro, e não como similaridade |
| **Estimou a escala** | há uma ordem de grandeza, e a escolha de ferramenta é coerente com ela |
| **Corte justificado por documento** | e não uma estratégia única aplicada a tudo |

---

## Dicas

- **Comece pela pergunta que o usuário faz.** Do enunciado da pergunta sai qual documento a responde — e, daí, o que precisa estar no índice.
- **Se você não consegue nomear o documento que responde**, o problema não é de chunking. É que a informação não está escrita em lugar nenhum, e nenhuma ferramenta desta aula resolve isso.
- **Teste o que o modelo já sabe antes de indexar.** Cinco perguntas ao modelo, conferidas contra a fonte, economizam um índice inteiro.
- **Volume pequeno é uma resposta legítima.** "São 40 páginas, cabe em memória, não vou usar banco" é uma decisão defensável — e defendê-la com número é exatamente o que a aula pede.
- **Não tente acertar.** É a `v1`, e ela existe para estar errada de um jeito que você consiga enxergar depois.
