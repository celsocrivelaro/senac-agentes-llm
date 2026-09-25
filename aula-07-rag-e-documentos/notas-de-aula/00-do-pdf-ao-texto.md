# IA Aplicada com LLMs — Aula 07: RAG e Documentos — Do PDF ao texto: a entrada de dados

## Introdução

Em seis aulas o corpus foi uma variável Python. O regulamento do `dados.py` é uma string, e ninguém precisou perguntar de onde ela veio. Na base de conhecimento de um case real ele chega em PDF, e a conversão de documento em caracteres tem modos de falha próprios — silenciosos, e anteriores a tudo o que se faz depois.

A ordem importa. Não há o que indexar antes de haver texto, não há *chunk* melhor que a extração que o produziu, e **nenhuma métrica de recuperação detecta o que nunca entrou no índice**. Por isso esta é a primeira nota da aula: ela vem antes da escolha da forma de recuperar (a [nota 01](01-as-formas-de-recuperar.md)) e antes de tudo o que se monta sobre o índice, e não depende de nada disso.

> **Pré-requisitos:** Aula 06, [nota 02](../../aula-06-embeddings-e-rag/notas-de-aula/02-chunking-e-a-medida-da-busca.md) — *chunking* por estrutura e `recall@k`.
>
> **Código:** [`00-extrair-pdf.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula07-rag-e-documentos/00-extrair-pdf.py).

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Explicar** por que um PDF não contém texto, e **distinguir** as três classes de PDF pelo tratamento que cada uma exige.
- **Adotar a licença da dependência como critério de escolha**, ao lado de desempenho.
- **Extrair** texto e tabela de um PDF e **verificar** que a extração está completa antes de indexar.
- **Antecipar** o que a extração decide para o resto do pipeline, e por que nada disso é corrigível adiante.

---

## Desenvolvimento teórico

### 1. O PDF não contém texto

A frase é literal. Um arquivo PDF guarda **instruções de desenho**: *"coloque o glifo A na coordenada (72, 640), com a fonte X em 11pt"*. Não existe parágrafo, não existe tabela, não existe ordem de leitura. O que a biblioteca devolve não é *o texto do documento* — é uma **reconstrução** feita a partir de coordenadas, e ela pode estar errada sem avisar.

```
  formato        o que está guardado        o que custa extrair
  ──────────────────────────────────────────────────────────────
  .txt / .md     o texto, e a estrutura     nada
  .docx          o texto, e a estrutura     pouco (é XML zipado)
  .html          o texto, e a estrutura     pouco
  .pdf           glifos e coordenadas       RECONSTRUÇÃO
  .pdf escaneado imagens de página          OCR, e ele erra
```

São três classes de PDF, e elas pedem ferramentas diferentes:

| Classe | Como reconhecer | O que usar |
|---|---|---|
| **digital** | o texto é selecionável no leitor | extração direta |
| **escaneado** | é uma foto da página | OCR, que não é assunto desta aula |
| **híbrido** | digital, com tabelas ou anexos como imagem | extração **mais** detecção do que faltou |

> **O híbrido é a classe perigosa**, porque falha em silêncio: a extração devolve as páginas de texto, some com as de imagem, e ninguém nota até a busca não achar uma cláusula que "está no PDF".

### 2. A biblioteca, e a licença como critério

| Biblioteca | Licença | O que faz | Quando |
|---|---|---|---|
| **pypdf** | BSD-3 | texto corrido, puro Python, sem dependência binária | o caminho mais curto, e suficiente para documento simples |
| **pdfplumber** | MIT | posição de cada caractere, *layout* e **extração de tabela** | quando o documento tem tabela — e documento normativo tem |
| **PyMuPDF** | **AGPL-3, ou licença comercial** | a mais rápida, e a melhor em *layout* | exige leitura da licença antes: a AGPL obriga a abrir o código de um serviço que a use |

A terceira linha é a lição, e ela não é sobre PDF. A biblioteca mais recomendada em *blog* e em fórum é a que tem a licença mais restritiva, e quase nenhum tutorial menciona isso. **Licença é critério de escolha de dependência**, ao lado de desempenho — e num trabalho que vira produto, é ela que decide.

O laboratório usa `pdfplumber`, e só ele: é permissivo e é o mais completo dos permissivos — o mesmo objeto devolve o texto corrido e a tabela, que é o que este anexo exige.

### 3. O experimento: a tabela que se desfaz

O regulamento em PDF tem um anexo com o quadro de limites por categoria. Extraído como **texto** — que é o modo padrão, e o que quase todo tutorial mostra —, ele sai assim:

```
Categoria Limite Artigo
Alimentação R$ 120,00 por refeição Art. 4º §1º
Alimentação internacional R$ 260,00 por refeição Art. 4º §2º
Hospedagem internacional R$ 640,00 por diária Art. 6º §2º
```

**Parece ter funcionado, e não funcionou.** A tabela virou linhas de texto e as **fronteiras de coluna desapareceram**. Onde termina a categoria e começa o limite? Para quem lê, o espaço basta. Para o corte em *chunks* e para o cosseno, `"Hospedagem internacional R$ 640,00 por diária Art. 6º §2º"` é uma frase só — e o valor, que decide a resposta, é indistinguível do resto dela.

A mesma biblioteca, na mesma página, chamada com `extract_tables()` em vez de `extract_text()`, devolve **estrutura**:

```
['Categoria', 'Limite', 'Artigo']
['Alimentação', 'R$ 120,00 por refeição', 'Art. 4º §1º']
['Alimentação internacional', 'R$ 260,00 por refeição', 'Art. 4º §2º']
['Hospedagem internacional', 'R$ 640,00 por diária', 'Art. 6º §2º']
```

Agora é dado. Com as colunas separadas, cada linha vira um *chunk* com o cabeçalho herdado — o corte por estrutura da Aula 06 — ou, melhor, a tabela vira **metadado** e a faixa de valor se resolve com `<=`, que é a consulta estruturada da [nota 01](01-as-formas-de-recuperar.md), §2.

> **Dois avisos de honestidade.**
>
> **A detecção de tabela é heurística, e depende das réguas.** O `extract_tables()` procura as linhas desenhadas na página; o anexo do laboratório tem réguas, e por isso sai limpo. Numa tabela alinhada só por espaçamento — que é comum — a detecção não acha nada, e é preciso passar uma estratégia por posição de texto. Em documento real ela também devolve, às vezes, uma "tabela" espúria com a página inteira dentro de uma célula: filtrar o resultado é parte do trabalho.
>
> **A biblioteca reclama e continua.** O `pdfminer`, que o `pdfplumber` usa por baixo, imprime avisos de fonte em PDFs malformados e devolve o resultado certo assim mesmo. PDF é um formato sujo, e ruído no `stderr` não é sinal de falha — o que é sinal de falha é não conferir a saída.

### 4. As outras fontes

O PDF é o caso dominante em acervo normativo, e não é o único:

| Fonte | Como se extrai | O que costuma quebrar |
|---|---|---|
| **HTML** (*scraping*) | requisição, mais extração do corpo | menu, rodapé e aviso de *cookie* viram *chunk*; a página muda e o índice envelhece |
| **Office** (`.docx`, `.xlsx`) | biblioteca do formato | planilha não é prosa: célula fora de contexto não significa nada |
| **E-mail e tickets** | API do sistema de origem | a citação da mensagem anterior se repete em todas as respostas do encadeamento |
| **Banco de dados e APIs** | consulta direta | o dado já é estruturado, e frequentemente **não precisa de RAG** — ver a [nota 01](01-as-formas-de-recuperar.md), §2 |
| **Áudio e vídeo** | transcrição automática | sem pontuação confiável não há fronteira de *chunk* |

O *scraping* acrescenta três problemas que o arquivo local não tem. O **conteúdo acessório** — menu, rodapé, barra lateral — é texto legítimo do ponto de vista do extrator, e separar o corpo do acessório é a parte não trivial da tarefa. A **volatilidade**: a página muda sem aviso, e o trecho recuperado passa a estar correto em relação ao índice e incorreto em relação ao mundo. E a questão **contratual**: termos de uso, `robots.txt` e limites de requisição são decisão anterior à técnica.

### 5. O que fazer antes de cortar em *chunks*

Uma lista curta, e é o que separa uma base de conhecimento de uma pasta de PDFs:

1. **Contar os caracteres por página.** Página com zero ou quase zero é imagem: o PDF é híbrido, e a extração devolve vazio para ela.
2. **Procurar o que sumiu.** Abrir o PDF no leitor, escolher três trechos difíceis — uma tabela, uma nota de rodapé, um anexo — e conferir se estão na saída.
3. **Decidir o que é tabela**, e tratar tabela como tabela, não como parágrafo.
4. **Guardar a procedência.** Arquivo, página e posição viram metadado, e é o que permite citar a fonte e conferir a citação em código.
5. **Normalizar o óbvio.** Hifenização de fim de linha, cabeçalho e rodapé repetidos em toda página, número de página no meio do texto.

O item 4 é o mais esquecido e o mais caro. Sem número de página no metadado, a resposta cita *"o regulamento"*, e não há como conferir.

### 6. O que a extração decide para o resto do pipeline

Três consequências, e nenhuma delas é corrigível adiante:

**O *chunk* não pode ser melhor que a extração.** A estratégia por estrutura da Aula 06 corta nos marcadores de artigo. Se a extração os embaralhou ou perdeu, o corte degrada para contagem de caracteres sem que nada no código mude — a mesma função, sobre um texto que não tem mais as fronteiras que ela procura.

**A perda é silenciosa.** As métricas da [nota 03](03-medir-o-rag-e-a-recuperacao-como-ferramenta.md) medem o pipeline **sobre o índice existente**. Um `recall@k` de 1,0 sobre um índice ao qual faltam quatro páginas continua sendo 1,0. Nenhuma métrica de recuperação detecta o que nunca entrou.

**A extração entra no carimbo.** A Aula 06 estabeleceu que trocar a estratégia de *chunking* invalida a comparação. A biblioteca de extração e a versão dela pertencem à mesma lista: dois extratores produzem textos diferentes do mesmo PDF, e dois índices construídos assim não são comparáveis.

---

## Exemplos

### Exemplo 1 — A extração que perdeu uma página

Saída de `00-extrair-pdf.py` sobre o regulamento em PDF:

```
  página 1:  2211 caracteres
  página 2:  1116 caracteres
  página 3:   458 caracteres
  página 4:     0 caracteres   <-- SEM CAMADA DE TEXTO

  1 de 4 páginas sem texto extraível: [4]
```

O pipeline construído sobre essa extração funciona. Indexa três páginas, responde com fluência, cita corretamente o que indexou — e é incapaz de responder qualquer pergunta cuja resposta estivesse na quarta. A diferença entre "extraí o documento" e "extraí parte do documento" custa três linhas de verificação.

### Exemplo 2 — A estrutura que sobreviveu

O mesmo script compara os marcadores de artigo do texto extraído com os do original, na ordem:

```
  marcadores `Art. N` no original: 12
  marcadores `Art. N` no extraído: 19  (inclui os do anexo, que não está no original)

  Os marcadores do corpo vieram na MESMA ORDEM.
```

A igualdade de ordem é a condição para que o *chunking* por estrutura da Aula 06 continue aplicável. Quando ela se rompe, o sintoma é leitura em múltiplas colunas — e a correção é trocar o extrator, não ajustar o corte.

---

## Fontes e leituras

ADOBE. **PDF reference**, sixth edition, version 1.7. 2006. (O modelo de conteúdo como instruções de desenho, §1.)

**Material da disciplina.** Aula 06, [nota 02](../../aula-06-embeddings-e-rag/notas-de-aula/02-chunking-e-a-medida-da-busca.md) — o corte por estrutura, que a §6 mostra depender da extração, e o `recall@k`, que a §6 mostra não detectar o que não entrou. Desta aula, [nota 01](01-as-formas-de-recuperar.md) §2 — a consulta estruturada, que é o destino da tabela extraída como dado; [nota 03](03-medir-o-rag-e-a-recuperacao-como-ferramenta.md) — as métricas que medem o pipeline sobre o índice existente.
