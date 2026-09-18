# IA Aplicada com LLMs — Aula 06: Embeddings e RAG — O router por embedding

## Introdução

A Aula 05 apresentou o *Router* e enumerou três formas de implementar o classificador que decide a rota. Sobre a segunda, aquela nota registrou:

> | **Embedding / similaridade** | converte a entrada em vetor e compara com exemplos de cada categoria | uma chamada barata, sem geração de texto — **é a Aula 06** |
>
> A forma do meio ainda não está disponível ao aluno — ela depende de embeddings, que é a aula seguinte, e é por isso que aparece aqui **nomeada e não implementada**.

O laboratório daquela aula levou o registro ao código, com uma linha que é uma dívida escrita por extenso:

```python
router_embedding = None   # Aula 06
```

Esta nota paga a dívida. E o faz sem introduzir mecanismo algum: o router por embedding é a função `ranquear` da [nota 01](01-o-vetor-e-a-similaridade.md), com o corpus trocado por exemplares de rota.

> **Pré-requisitos:** notas [01](01-o-vetor-e-a-similaridade.md) e [02](02-chunking-e-a-medida-da-busca.md) desta aula · Aula 05, [nota 01-2](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-2-router.md) — o *Router* e as três formas de rotear.
>
> **Código:** [`04-router.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula06-embeddings-e-rag/04-router.py). Como os demais scripts, importa o `gerar_matrix_embbeddings` do `embedding.py` e o `cosseno` do `similaridade.py`.

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Implementar** um classificador por similaridade a partir de exemplares de rota, sem treino e sem geração de texto.
- **Reconhecer** que buscar e classificar são a mesma operação, e que o que as distingue é apenas o que foi indexado.
- **Comparar** as três formas de rotear pelo custo e pela condição de não uso de cada uma.
- **Diagnosticar** uma decisão de roteamento frágil pela margem entre a primeira e a segunda rota, em vez do score absoluto.
- **Justificar** a ordem da cascata — regra, embedding, modelo — pelo limite conhecido de cada degrau.

---

## Desenvolvimento teórico

### 1. Classificar é buscar, com outro índice

Um buscador recebe uma pergunta, compara com chunks de um documento e devolve os mais próximos. Um router recebe uma mensagem, compara com **exemplares de rota** e devolve a rota do mais próximo.

A operação é idêntica. O que muda é o que está no índice:

| | O que se indexa | O que se devolve |
|---|---|---|
| Buscador (nota 02) | chunks de um documento | os `k` trechos mais próximos |
| Router (esta nota) | frases-exemplo de cada rota | a rota do exemplar mais próximo |

```python
EXEMPLARES = {
    "consulta_status": ["qual o status do meu pedido",
                        "onde está minha encomenda", ...],
    "reclamacao":      ["meu pedido está atrasado e ninguém resolve",
                        "a encomenda chegou danificada", ...],
    "fora_de_escopo":  ["vocês têm vaga de emprego", ...],
}
```

Não há treino, não há *prompt*, não há geração. O router "sabe" o que essas frases dizem, e nada além disso. Acrescentar uma rota é acrescentar uma chave no dicionário — o que torna esta forma barata de manter, e é parte do argumento a favor dela.

A assimetria do `03-buscador.py` vale aqui igual: os exemplares são embutidos **uma vez**, na largada; cada mensagem classificada custa uma chamada — a dela.

### 2. A margem importa mais que o score

O router devolve a rota vencedora e o score dela. O score isolado engana, pelo motivo que a [nota 01 §5](01-o-vetor-e-a-similaridade.md) estabeleceu: textos em português compartilham estrutura, e o piso real do cosseno fica bem acima de zero.

O número que informa é a **margem** — a distância entre o exemplar vencedor e o exemplar mais próximo de **outra** rota:

```
  consulta_status  score 0.8482  margem +0.1320     <- decisão folgada
  consulta_status  score 0.7761  margem +0.0108     <- empate disfarçado
```

As duas caíram na **mesma rota**, com scores da mesma ordem — e só a segunda está errada: é uma reclamação sobre caixa amassada. O score não denuncia; a margem sim. Com 0,0108, o segundo colocado ficou a um centésimo, e qualquer variação de redação trocaria a rota sem que o sistema soubesse que esteve perto de decidir o contrário. **Margem estreita é candidata a revisão humana, não a decisão automática.**

### 3. Onde ele quebra

O par abaixo difere em uma palavra e exige tratamentos opostos — um é agradecimento, o outro é entrega perdida:

```
  "o pedido 48219 chegou certinho, obrigado"
  "o pedido 48219 não chegou"
```

O router manda os dois para o mesmo lugar:

```
  consulta_status  score 0.8313  margem +0.0638  'o pedido 48219 chegou certinho, obrigado'
  consulta_status  score 0.7963  margem +0.0041  'o pedido 48219 não chegou'

  cosseno entre as duas .... 0.8908
```

A reclamação vai para a fila de status, e a margem da decisão errada é de **0,0041** — o empate disfarçado da §2, agora com consequência. E não adianta mexer nos exemplares: o problema não está neles, está em como o vetor representa as duas frases. A Aula 07 volta a este caso, dá nome a ele e mostra que não é isolado.

O que importa registrar agora é o **contexto** em que o erro ocorre. Não há corpus, não há trechos recuperados, não há modelo escrevendo:

> **O erro acontece sem nenhuma geração de texto envolvida.** Não é um problema de RAG — é um problema de **representação**, e aparece em qualquer sistema que decida por proximidade, inclusive num classificador de três rotas.

### 4. A cascata, completa pela primeira vez

A saída é a mesma que a Aula 05 estabeleceu para o *Router*, e agora os três degraus existem:

| Degrau | Custo | Resolve | Não resolve |
|---|---|---|---|
| **Regra** | zero chamadas | polaridade, número, identificador, data — tudo que é determinístico | variação de redação que ninguém enumerou |
| **Embedding** | uma chamada barata, sem geração | mensagens que dizem a mesma coisa com outras palavras | o que o vetor não representa |
| **Semântico (LLM)** | uma chamada de geração | o caso que as duas anteriores não previram | o custo, quando o volume cresce |

```python
def regra(mensagem: str) -> str | None:
    """Polaridade se resolve com palavra-chave, não com vetor."""
    baixo = mensagem.lower()
    if any(t in baixo for t in ("não chegou", "nao chegou", "não recebi")):
        return "reclamacao"
    return None
```

Três termos em código resolvem a negação que nenhum ajuste de exemplar resolve. É a mesma conclusão da nota 02 — **regra determinística onde ela existe** —, agora com a ordem de aplicação explícita.

Cada degrau existe porque o anterior tem um limite **conhecido**. Um router que começa no degrau mais caro não economiza nada; um que nunca sobe erra nos casos que não previu.

### 5. O limiar, e a rota que diz "não sei"

Sem piso, o classificador **sempre** devolve uma rota — a menos ruim entre as que existem. É o mesmo defeito que a Aula 05 apontou no router semântico: sem escape, o classificador é obrigado a escolher, e escolhe.

```
  piso 0.00 -> consulta_status   score 0.7718
  piso 0.82 -> nenhuma           nada acima do piso
```

A rota `nenhuma` não é um detalhe de robustez: é o que permite ao sistema declarar ignorância e mandar o caso para uma fila humana. Ela reaparece na Aula 07 com outro nome — o campo `suficiente: false` da resposta com contrato.

E o piso não se chuta. Calibra-se contra um conjunto de mensagens com rota conhecida, exatamente como o `k` se escolhe na [nota 02](02-chunking-e-a-medida-da-busca.md) — que é, de novo, o conjunto de avaliação desta aula em outro uso.

---

## Exemplos

### Exemplo 1 — O lote roteado, com a margem visível

```
  consulta_status  score 0.8482  margem +0.1320
                   'Qual o status do pedido 48219?'

  consulta_status  score 0.8160  margem +0.0404
                   'onde está meu pedido 31002'

  reclamacao       score 0.8665  margem +0.0704
                   'O pedido 48219 está atrasado há duas semanas...'

  consulta_status  score 0.7761  margem +0.0108     <- ERRADO
                   'a caixa chegou toda amassada, quero reembolso'

  fora_de_escopo   score 0.8085  margem +0.0751
                   'vocês estão contratando?'
```

*(Valores medidos com `mistral-embed`.)*

Nenhuma chamada de geração no lote inteiro. O router semântico da Aula 05 gastaria uma por mensagem.

**E o lote já contém um erro.** A quarta mensagem é uma reclamação evidente — caixa amassada, pedido de reembolso — e foi para `consulta_status`. Repare em qual sinal denuncia: não o score, que é 0,7761 e não destoa; a **margem, de 0,0108**. O segundo lugar ficou a um centésimo do primeiro.

É o argumento da §2 com dado real: uma margem dessa ordem não é uma decisão, é um empate. Um sistema que encaminhasse tudo automaticamente mandaria essa mensagem para a fila errada sem registrar que quase tinha escolhido a outra.

### Exemplo 2 — A negação, com o preço na tela

```
  consulta_status  score 0.8313  margem +0.0638   'o pedido 48219 chegou certinho, obrigado'
  consulta_status  score 0.7963  margem +0.0041   'o pedido 48219 não chegou'

  cosseno entre as duas .... 0.8907
```

A mesma rota para um agradecimento e para uma entrega perdida. O `0,89` entre as duas mensagens é o mesmo fenômeno de `"despesa aprovada"` × `"despesa reprovada"`, que a Aula 07 mede em `0,94` — aqui num domínio diferente e com consequência operacional imediata.

A margem da segunda linha — **0,0041** — é ainda mais estreita que a do Exemplo 1. O router não está decidindo; está desempatando ruído.

### Exemplo 3 — A cascata desempatando

```
  consulta_status  [EMBEDDING]  1 chamada      'o pedido 48219 chegou certinho, obrigado'
  reclamacao       [REGRA]      zero chamadas  'o pedido 48219 não chegou'
```

A regra captura o caso que sabe reconhecer e **declina** do resto — devolvendo `None`, não um palpite. É o mesmo desenho do `01-router.py` da Aula 05, e a diferença é que agora o degrau seguinte não é o modelo de geração: é o embedding.

### Exemplo 4 — O piso, e o que ele barra

```
  mensagem: 'gostaria de saber sobre o índice de reajuste da tabela de fretes'

  piso 0.00 -> consulta_status   score 0.7718
  piso 0.75 -> consulta_status   score 0.7713
  piso 0.82 -> nenhuma           (nada acima do piso)
```

A mensagem não pertence a nenhuma das três rotas. Sem piso, ela vira consulta de status. Com piso em 0,75 — que parece alto — ela **ainda** passa, porque o piso de textos sem relação nenhuma, medido na [nota 01](01-o-vetor-e-a-similaridade.md) §5, é justamente dessa ordem.

Só em 0,82 o router declina. E esse número não veio de intuição: veio de olhar onde ficam os acertos folgados do Exemplo 1 — entre 0,81 e 0,87 — e cortar abaixo deles.

---

## Fontes e leituras

**Material da disciplina.** Aula 05, [nota 01-2](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-2-router.md) — o *Router* e as três formas de rotear, e a linha `router_embedding = None` que esta nota preenche. Aula 06, [nota 01](01-o-vetor-e-a-similaridade.md) §4 e §5 — `ranquear` e a faixa de valores reais; Aula 07, nota 04 §2 — a cegueira de negação, que é o nome do defeito visto aqui.

ANTHROPIC. **Building effective agents**, 2024. — o padrão *Routing*, e a observação de que a classificação pode ser feita "por um modelo, por um classificador ou por regras tradicionais".
