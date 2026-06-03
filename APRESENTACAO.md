# Sistema de Recuperação de Informação — Busca em Projetos de Lei

### Apresentação do Trabalho — Foco no `buscador_web.py`

---

## 1. O que é o projeto

Um **buscador (motor de busca)** sobre ~50 mil projetos de lei da Câmara dos
Deputados.

O usuário digita uma consulta (ex.: *"proteção de dados pessoais"*) e o sistema
devolve, em **milissegundos**, a lista de projetos de lei **mais relevantes**,
ordenados por um score de relevância (**TF-IDF**), com link direto para a ficha
oficial no site da Câmara.

> O `buscador_web.py` é a **etapa final** do projeto: a interface de consulta.
> Ele não acessa os HTMLs originais — trabalha apenas sobre o **índice
> invertido** (`indice_projetos_lei.json`) gerado pelo `indexador.py`.

---

## 2. Conceito central: o Índice Invertido

Em vez de varrer os 50 mil documentos a cada busca (lento), o sistema usa um
**índice invertido**: uma estrutura que mapeia **cada termo → em quais
documentos ele aparece e quantas vezes**.

```json
{
  "proteç":  { "projetoDeLei_2345643.html": 4, "projetoDeLei_2350112.html": 2 },
  "dad":     { "projetoDeLei_2345643.html": 7, "projetoDeLei_2360087.html": 1 },
  "pessoal": { "projetoDeLei_2345643.html": 3 }
}
```

- A **chave** é o termo já processado (radical/stem).
- O **valor** é a lista de *postings*: `documento → frequência do termo (TF)`.

É exatamente como o índice remissivo no final de um livro: você não lê o livro
inteiro, vai direto na palavra e ela te diz as páginas.

---

## 3. Arquitetura do `buscador_web.py`

O código é dividido em 5 blocos bem definidos:

| Bloco | Classe / Função | Responsabilidade |
|---|---|---|
| 1 | `ProcessadorConsulta` | Pré-processa o texto da consulta |
| 2 | `MotorBusca` | Carrega o índice e calcula o ranking TF-IDF |
| 3 | `HTML` (template) | Interface visual no navegador |
| 4 | Rotas Flask (`/`, `/status`, `/buscar`) | Comunicação navegador ↔ servidor |
| 5 | Ponto de entrada | Carrega o índice e sobe o servidor |

---

## 4. Bloco 1 — Processamento da consulta

A consulta do usuário passa pelo **mesmo pipeline** usado na indexação. Isso é
fundamental: só assim os termos da busca "casam" com os radicais guardados no
índice.

```python
def processar(self, texto):
    texto = re.sub(r"<.*?>", " ", texto)          # 1. remove HTML
    tokens = re.split(r"[\s\.\,\;...]+", texto.lower())  # 2. lowercase + tokeniza
    termos = []
    for tok in tokens:
        if len(tok) < 3 or tok in self.stopwords:  # 3. remove stopwords / curtos
            continue
        termos.append(self.stemmer.stem(tok))       # 4. stemming (RSLP)
    return termos
```

**Exemplo:** `"Proteção dos Dados"` →
1. lowercase/tokeniza → `["proteção", "dos", "dados"]`
2. remove *stopword* `"dos"` → `["proteção", "dados"]`
3. stemming → `["proteç", "dad"]`

- **Stopwords**: palavras muito comuns sem valor de busca (*de, os, para, com…*) — via NLTK.
- **Stemming (RSLP)**: reduz a palavra ao radical (*proteção, proteger, protegido → proteç*), para que variações da mesma palavra sejam tratadas como iguais.

---

## 5. Bloco 2 — Motor de Busca e o ranking TF-IDF

### Por que ranquear?
Não basta achar os documentos que contêm os termos — é preciso ordená-los do
**mais relevante** para o menos relevante. Para isso usamos **TF-IDF**.

### As fórmulas

```
TF(t, d)    = frequência do termo t no documento d        (já está no índice)
IDF(t)      = log10( N / df(t) )                            N = total de documentos
score(d, q) = Σ  TF(t,d) × IDF(t)    para cada termo t da consulta presente em d
```

- **TF (Term Frequency)**: quanto mais vezes o termo aparece no documento, mais
  relevante ele tende a ser.
- **IDF (Inverse Document Frequency)**: termos **raros** valem mais. Uma palavra
  que aparece em quase todos os documentos quase não distingue resultados (IDF
  baixo); uma palavra rara é muito discriminativa (IDF alto).
- **Score**: soma de `TF × IDF` de cada termo da consulta. Quem soma mais, sobe
  no ranking.

### Otimização de desempenho
O **IDF de cada termo é pré-calculado uma única vez**, no carregamento do índice
(`idf_cache`). Assim, cada busca só soma valores já prontos → resposta em
**tempo sub-segundo** mesmo com dezenas de milhares de documentos.

```python
def buscar(self, consulta, top_k=20):
    stems = self.processador.processar(consulta)     # processa a consulta
    scores = {}
    for stem in stems:
        idf = self.idf_cache[stem]                   # IDF já pronto
        for doc_id, tf in self.indice[stem].items(): # percorre só os docs do termo
            scores[doc_id] = scores.get(doc_id, 0.0) + tf * idf   # acumula score
    ordenados = sorted(scores.items(), key=lambda x: x[1], reverse=True)[:top_k]
    ...
```

Note a eficiência: o laço percorre **apenas os documentos que contêm os termos
buscados** (via índice invertido), nunca os 50 mil.

---

## 6. Do resultado ao link da Câmara

Cada documento é identificado pelo nome do arquivo (ex.:
`projetoDeLei_2345643.html`). O motor extrai o **número da proposição** e monta
a URL oficial:

```python
match = re.search(r"(\d+)", doc_id)        # extrai "2345643"
prop_id = match.group(1)
url = f"https://www.camara.leg.br/proposicoesWeb/fichadetramitacao?idProposicao={prop_id}"
```

Por isso o buscador **não precisa dos HTMLs originais** para funcionar: o link
aponta direto para a página viva da Câmara.

---

## 7. Blocos 3 e 4 — Interface Web (Flask)

A aplicação web é servida com **Flask** e tem 3 rotas:

| Rota | Método | O que faz |
|---|---|---|
| `/` | GET | Devolve a página HTML da busca |
| `/status` | GET | Informa se o índice já terminou de carregar |
| `/buscar` | POST | Recebe a consulta (JSON), executa a busca e devolve os resultados (JSON) |

O front-end (HTML + JavaScript embutido) envia a consulta para `/buscar` de
forma assíncrona (*fetch*) e renderiza os resultados na tela, mostrando também o
**tempo da busca** e os **stems** efetivamente pesquisados.

---

## 8. Bloco 5 — Inicialização

Ao rodar `python buscador_web.py`:

1. Procura o `indice_projetos_lei.json` na **mesma pasta** do script.
2. Carrega o índice e **pré-calcula todos os IDFs**.
3. Sobe o servidor em `http://localhost:5000`.

```
=======================================================
  Sistema de Recuperação de Proposições Legislativas
=======================================================
  Carregando índice: .../indice_projetos_lei.json
  ✓ 50.282 documentos | XXX.XXX termos | X.Xs
  Acesse no navegador: http://localhost:5000
=======================================================
```

---

## 9. Fluxo completo (resumo visual)

```
   Usuário digita:  "proteção de dados"
            │
            ▼
   [ProcessadorConsulta]  →  remove HTML, stopwords, aplica stemming  →  ["proteç", "dad"]
            │
            ▼
   [MotorBusca]  →  consulta o índice invertido
            │         para cada termo: score += TF × IDF
            ▼
   Ranking (top 20)  →  ordenado por score
            │
            ▼
   [Flask /buscar]  →  devolve JSON  →  navegador exibe os resultados
                                          com link para camara.leg.br
```

---

## 10. Pontos fortes para destacar na apresentação

- **Índice invertido**: busca não varre os 50 mil documentos — vai direto aos termos.
- **TF-IDF**: ranqueamento clássico de RI, equilibrando frequência e raridade dos termos.
- **Consistência de pipeline**: consulta e indexação usam exatamente o mesmo
  pré-processamento (stopwords + stemming RSLP em português).
- **Desempenho**: IDF pré-computado → respostas em tempo sub-segundo.
- **Interface web** simples e funcional (Flask), com links para a fonte oficial.
- **Separação de responsabilidades**: indexação (offline) × busca (online) são
  independentes.

---

## 11. Tecnologias utilizadas

| Tecnologia | Uso |
|---|---|
| **Python** | Linguagem base |
| **NLTK** | Stopwords em português + Stemmer RSLP |
| **Flask** | Servidor web e API de busca |
| **JSON** | Formato de persistência do índice invertido |
| **TF-IDF** | Modelo de ranqueamento por relevância |

---

### Possíveis perguntas da banca (e respostas curtas)

- **Por que stemming?** Para que "proteção", "proteger" e "protegido" sejam
  tratados como o mesmo conceito, aumentando o recall.
- **Por que IDF com `log`?** Para suavizar o peso: sem o log, termos raros
  dominariam de forma desproporcional o score.
- **Por que o índice é grande (~164 MB)?** Ele guarda, para cada termo, todos os
  documentos onde aparece e a frequência — é o custo de ter busca instantânea.
- **E se um termo da consulta não estiver no índice?** Ele é simplesmente
  ignorado no cálculo do score (`if stem not in self.indice: continue`).
