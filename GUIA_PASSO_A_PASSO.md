# Webscraping da Jurisprudência do TST — Guia Passo-a-Passo

Este guia explica como rodar o script `tst_scraper.py` para coletar decisões
monocráticas (e outros tipos) do portal de Pesquisa de Jurisprudência do TST e
salvar tudo em uma planilha Excel.

---

## 1. Como o portal funciona por trás dos panos

A página `https://jurisprudencia.tst.jus.br` é uma SPA (Single Page Application)
em React. Em vez de o HTML já vir pronto do servidor, o JavaScript do navegador
faz uma chamada de API (fetch) para um backend e monta a tela com o resultado.

Fizemos a engenharia reversa do código JS minificado e descobrimos:

- **Endpoint de configuração:** `/config.json` informa o `base_url` da API:
  `https://jurisprudencia-backend2.tst.jus.br`.
- **Endpoint de pesquisa:**
  `POST /rest/pesquisa-textual/{inicio}/{tamanho}?a={random}`
  Aceita um JSON com filtros (datas, ementa, ministros, tipos, etc.) e devolve
  um lote paginado de resultados.
- **Cada registro já contém o inteiro teor** no campo `txtConteudoDecisao`
  (HTML, eventualmente RTF). **Não é necessário** uma segunda requisição em
  `/rest/documentos/{id}` para cada decisão — isso reduz drasticamente o número
  de chamadas e o tempo total.
- **O hash da URL** (ex.: `#53eabf25b2d3c5691d592f906405be74`) **não influencia
  a busca**. Ele é apenas um marcador local da SPA. Quem dirige a pesquisa é o
  body do POST. Por isso o script não precisa do hash; em vez disso, você
  configura os filtros no próprio Python.

> **Boa prática (compliance):** o `/robots.txt` da página não bloqueia, mas o
> meta-tag `noindex, nofollow, noarchive` indica que o site prefere não ser
> indexado. Mantenha um intervalo entre requisições (o script já faz isso) e
> evite execuções paralelas agressivas.

---

## 2. Instalação do Python (se ainda não tiver)

### Windows
1. Baixe o instalador em <https://www.python.org/downloads/>.
2. **Marque a opção “Add Python to PATH”** antes de clicar em *Install Now*.
3. No Prompt de Comando, confirme: `python --version`.

### macOS
- Recomendado via Homebrew: `brew install python@3.12`.
- Ou direto do instalador em <https://www.python.org/downloads/>.

### Linux
Já vem instalado na maioria das distros. Se não:
```
sudo apt install python3 python3-venv python3-pip
```

---

## 3. Preparando o ambiente do projeto

Abra o terminal na pasta onde estão `tst_scraper.py` e este guia:

```bash
# 1) Cria um ambiente virtual isolado (boa prática para não mexer no Python global)
python -m venv .venv

# 2) Ativa o ambiente
source .venv/bin/activate          # macOS/Linux
.\.venv\Scripts\activate           # Windows (PowerShell)

# 3) Instala as 3 bibliotecas usadas pelo script
pip install requests openpyxl beautifulsoup4
```

| Biblioteca       | Para quê serve                                        |
|------------------|--------------------------------------------------------|
| `requests`       | Fazer chamadas HTTP (GET/POST) ao servidor do TST.    |
| `openpyxl`       | Criar e escrever o arquivo `.xlsx`.                   |
| `beautifulsoup4` | Limpar HTML para extrair o texto puro do inteiro teor.|

---

## 4. Ajustando a sua pesquisa

Abra o `tst_scraper.py` em um editor de texto e localize o dicionário
**`SEARCH_FILTERS`** (linha ~50). Os campos espelham exatamente o formulário
da Pesquisa de Jurisprudência:

```python
SEARCH_FILTERS = {
    "ou": "",                # operador "OU" (qualquer um dos termos)
    "e": "",                 # operador "E" (todos os termos)
    "termoExato": "",        # expressão exata
    "naoContem": "",         # exclusão
    "ementa": "",             # busca apenas na ementa
    "dispositivo": "",        # busca no dispositivo
    "tipos": ["DESPACHO"],    # decisões monocráticas
    "orgao": "TST",           # ou "CSJT"
    "publicacaoInicial": "2024-01-01",   # YYYY-MM-DD ou None
    "publicacaoFinal":   "2024-12-31",
    "julgamentoInicial": None,
    "julgamentoFinal":   None,
    "ordenacao": "data",      # "data" | "numero" | "relevancia"
    # ... (listas vazias para os demais filtros)
}
```

### Códigos dos tipos de jurisprudência

| Código     | Significado                       |
|------------|------------------------------------|
| `DESPACHO` | Decisões Monocráticas (alvo aqui)  |
| `ACORDAO`  | Acórdãos                           |
| `SUM`      | Súmulas                            |
| `PN`       | Precedentes Normativos             |
| `OJ`       | Orientações Jurisprudenciais       |
| `DESPGP`   | Decisões da Presidência            |
| `DESPGVP`  | Decisões da Vice-Presidência       |
| `DESPGCG`  | Decisões da Corregedoria-Geral     |

> Para reproduzir uma pesquisa específica que você fez no portal, abra o
> DevTools do navegador (F12 → aba **Rede / Network**), faça a busca, encontre
> a requisição `pesquisa-textual` e copie o **Request Payload** — todos os
> filtros estão lá em JSON. Cole esses valores no `SEARCH_FILTERS`.

---

## 5. Executando

Com o ambiente virtual ativado:

```bash
# Teste com poucos registros antes da rodada cheia:
python tst_scraper.py -n 10 -p 5 -o teste.xlsx

# Rodada completa, gravando em decisoes_tst.xlsx:
python tst_scraper.py
```

Opções disponíveis (`python tst_scraper.py --help`):

| Flag              | Função                                                     |
|-------------------|------------------------------------------------------------|
| `-o ARQ.xlsx`     | Caminho do arquivo de saída                                |
| `-n N`            | Limite máximo de registros a coletar (útil para testes)    |
| `-p N`            | Tamanho da página (registros por requisição). Padrão 100.  |
| `-v`              | Logs detalhados (DEBUG)                                    |

Durante a execução o script imprime progressos como:

```
00:14:02 [INFO] Total de registros encontrados: 1137
00:14:09 [INFO] Coletados 50 registros...
00:14:17 [INFO] Coletados 100 registros...
...
00:18:31 [INFO] ✓ 1137 registros gravados em /Users/.../decisoes_tst.xlsx
```

> Tempo estimado: ~1–2 minutos para cada 100 registros (depende do tamanho dos
> inteiros teores e da latência da rede).

---

## 6. Estrutura do Excel gerado

| Coluna                  | Conteúdo                                                  |
|-------------------------|------------------------------------------------------------|
| ID                      | Identificador interno (32 caracteres)                     |
| Tipo                    | "Despacho", "Acórdão" etc.                                |
| Processo                | Ex.: `AIRR - 0010983-03.2020.5.15.0006`                   |
| Nº Único                | Numeração estruturada (CNJ)                                |
| Classe                  | Sigla (AIRR, RR, RO etc.)                                 |
| Órgão Julgador          | Ex.: "5ª Turma"                                            |
| Relator                 | Nome do relator (com capitalização normalizada)            |
| Publicação              | Data ISO `YYYY-MM-DD`                                      |
| Data (ordenação)        | Data usada pela ordenação                                  |
| Atualização             | Última atualização do registro                             |
| Tema                    | Códigos de temas (separados por vírgula)                   |
| URL Inteiro Teor        | Link direto para o HTML no backend                         |
| Inteiro Teor (texto)    | Texto puro (HTML/RTF stripped). Truncado em 32.000 chars.  |
| Inteiro Teor (HTML)     | HTML/RTF original. Truncado em 32.000 chars.               |

> **Por que 32.000 caracteres?** É o limite físico do Excel para uma célula
> (32.767). Cerca de 3 % das decisões muito longas excedem esse limite; o
> script trunca e adiciona `[... TRUNCADO POR LIMITE DO EXCEL ...]` no final.
> Se você precisar do texto íntegro desses casos, basta abrir a `URL Inteiro
> Teor` no navegador, ou eu posso adaptar o script para gravar `.html` à parte
> dos longos.

---

## 7. Resolução de problemas

| Sintoma                                          | Causa provável                                        | O que fazer                              |
|--------------------------------------------------|--------------------------------------------------------|-------------------------------------------|
| `HTTP 400` ao iniciar                            | Filtro com tipo errado (string em vez de lista, etc.) | Verifique se `tipos` é lista de strings.  |
| `HTTP 429` ou `503`                              | Rate limit do servidor                                | Aumente `SLEEP_BETWEEN_PAGES`.            |
| 0 registros coletados                            | Filtros muito restritivos                             | Confirme datas e tipos.                   |
| "Falha definitiva ao buscar página inicio=…"     | Conexão instável                                      | Rode novamente; o script já faz retries.  |
| Texto com `\rtf` solto                           | Backend devolveu RTF                                  | Já tratado; instale `striprtf` para mais. |

---

## 8. Próximos passos sugeridos

1. **Filtros por relator/turma específicos** — você pode descobrir os IDs
   chamando `GET https://jurisprudencia-backend2.tst.jus.br/rest/ministros` e
   `/rest/orgaos-judicantes`. Depois preencha as listas no `SEARCH_FILTERS`.
2. **Salvar HTML em pastas separadas** — ideal para inteiros teores muito
   longos. Posso adaptar o script para fazer isso com 1 arquivo `.html` por
   processo.
3. **Cruzar com sua base de processos** — o `numero_unico` em formato CNJ
   facilita o JOIN com sistemas internos (Projudi, e-Saj, etc.).
4. **Análise textual** — com a coluna "Inteiro Teor (texto)" pronta, é simples
   rodar busca por palavra-chave, contagens de ocorrências, ou alimentar uma
   pipeline de NLP.

---

*Quaisquer dúvidas ou ajustes (ex.: filtrar uma turma específica, salvar PDFs,
acrescentar uma coluna de citação ABNT), é só pedir.*
