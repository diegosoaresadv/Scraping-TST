# Como investigar a API de um site â mÃ©todo passo-a-passo

Este documento desconstrÃ³i o que foi feito para descobrir a API da Pesquisa de
JurisprudÃªncia do TST. O objetivo Ã© que vocÃª consiga repetir esse processo em
qualquer site (e-SAJ, TJs, JusBrasil, Receita, sistemas internos do escritÃ³rio,
etc.). A lÃ³gica Ã© sempre a mesma; sÃ³ os detalhes mudam.

---

## 1. O modelo mental: HTML estÃ¡tico Ã SPA Ã API

Antes de qualquer cÃ³digo, Ã© preciso entender em que tipo de site vocÃª estÃ¡.

### 1.1 Site "tradicional" (HTML server-side)
O servidor monta a pÃ¡gina inteira em HTML e envia pronta. Quando vocÃª dÃ¡
`Ctrl + U` (ver cÃ³digo-fonte), os dados que aparecem na tela jÃ¡ estÃ£o lÃ¡.
Exemplo clÃ¡ssico: a busca pÃºblica do TJSP, em parte, Ã© assim. Para esses
sites, basta usar `requests` + `BeautifulSoup` â pegar o HTML e extrair com
seletores CSS.

### 1.2 SPA (Single Page Application)
O servidor entrega uma "casca" mÃ­nima de HTML e um arquivo JavaScript grande.
Esse JavaScript roda no navegador, faz uma chamada (fetch) a uma **API**, e sÃ³
depois "pinta" a tela com os dados. Quando vocÃª faz `Ctrl + U`, vÃª quase nada
â porque os dados ainda nem chegaram. **O TST Ã© uma SPA em React.**

> **Sinal claro de SPA:** o `Ctrl + U` mostra um `<div id="root"></div>` quase
> vazio e uma tag `<script>` apontando para `main.[hash].js`.

### 1.3 API (Application Programming Interface)
Ã um endpoint HTTP que devolve dados estruturados (geralmente JSON), em vez de
HTML. Ã *o que a SPA usa por baixo*. **Quando vocÃª descobre a API, descobre a
maneira mais rÃ¡pida e estÃ¡vel de raspar o site** â porque vocÃª fala com o
servidor na mesma linguagem em que ele se comunica com o JS dele.

> **Regra de ouro:** sempre que possÃ­vel, raspe a API, nÃ£o a tela.

---

## 2. As ferramentas

| Ferramenta              | Para quÃª                                      |
|-------------------------|------------------------------------------------|
| **DevTools do navegador** (F12) | Ver requisiÃ§Ãµes, payloads e respostas.   |
| **`curl`** (terminal)   | Replicar uma chamada HTTP fora do navegador.   |
| **`grep`** / `find` / editor | Buscar dentro de arquivos (inclusive do JS).   |
| **Python** (`requests`, `json`) | Automatizar e iterar pÃ¡ginas.            |
| **Postman/Insomnia** (opcional) | GUI para testar APIs sem escrever curl.  |

VocÃª nÃ£o precisa de tudo de uma vez. Comece com **DevTools + curl + Python**.

---

## 3. Roteiro de investigaÃ§Ã£o aplicado ao TST

Vou recriar a sequÃªncia exata de passos. Cada subseÃ§Ã£o tem o **comando que
rodei** e o **insight que ganhei**.

### Passo 3.1 â Confirmar que Ã© uma SPA

```bash
curl -s -L "https://jurisprudencia.tst.jus.br/" | head -c 1500
```

SaÃ­da (resumida):
```html
<!doctype html><html lang="pt-br">...
<title>Pesquisa de jurisprudÃªncia</title>
<script defer src="/static/js/main.be6b1d66.js"></script>
...
<div id="root"></div>
```

**ConclusÃ£o:** SPA confirmada. O HTML nÃ£o tem dados; eles virÃ£o via JavaScript.
A "alma" do site estÃ¡ em `main.be6b1d66.js`.

### Passo 3.2 â Olhar o trÃ¡fego de rede no navegador

Esse Ã© o passo mais importante. Sem cÃ³digo.

1. Abra a pÃ¡gina no Firefox/Chrome.
2. Pressione **F12**, vÃ¡ na aba **Rede / Network**.
3. Marque o filtro **Fetch/XHR** (sÃ³ requisiÃ§Ãµes de dados).
4. Clique em "Pesquisar" no formulÃ¡rio.
5. Cada linha que aparecer Ã© uma chamada Ã  API.

VocÃª verÃ¡ algo como:
```
POST  rest/pesquisa-textual/1/100?a=0.345...   200 OK   1.4 MB   2.3s
```

Clique nela e olhe trÃªs abas:

- **CabeÃ§alhos / Headers** â URL completa, mÃ©todo, headers enviados.
- **Carga Ãºtil / Payload** â o JSON que o navegador enviou (filtros).
- **Resposta / Response** â o JSON que voltou.

> **Esse Ã© o "santo graal" da investigaÃ§Ã£o.** VocÃª acabou de mapear:
> URL + mÃ©todo + headers + body + resposta. Ã tudo que o Python precisa
> imitar. (No nosso caso, o user descobriu o endpoint de inteiro teor pela
> aba **Inspetor** olhando a `<a>` do botÃ£o "Inteiro Teor".)

### Passo 3.3 â Replicar a chamada com `curl`

Com a chamada selecionada no DevTools, clique com o botÃ£o direito â **Copiar
como cURL**. Cole no terminal. Funciona? Ãtimo â vocÃª reproduziu um cliente
fora do navegador.

No nosso caso, eu fiz o curl Ã  mÃ£o para ter mais controle:

```bash
curl -s -X POST "https://jurisprudencia-backend2.tst.jus.br/rest/pesquisa-textual/1/3?a=0.123" \
  -H "Content-Type: application/json" \
  -H "Origin: https://jurisprudencia.tst.jus.br" \
  -H "Referer: https://jurisprudencia.tst.jus.br/" \
  --data @payload.json
```

Anatomia do comando:

| Parte                          | O que faz                                          |
|--------------------------------|------------------------------------------------------|
| `-s`                           | "silent" â esconde a barra de progresso              |
| `-X POST`                      | mÃ©todo HTTP (default Ã© GET)                          |
| `-H "..."`                     | adiciona um header                                   |
| `--data @payload.json`         | envia o conteÃºdo do arquivo como body                |
| `-w "%{http_code}"`            | imprime o cÃ³digo de status no final                  |

> **Headers que costumam ser exigidos:** `Content-Type`, `Origin`, `Referer`,
> `User-Agent`. Quando uma API recusa (HTTP 400/403), 90% das vezes Ã© porque
> falta um deles ou o `Content-Type` estÃ¡ errado.

### Passo 3.4 â Quando nÃ£o hÃ¡ um botÃ£o evidente: ler o JavaScript

Ãs vezes o que vocÃª precisa sÃ³ aparece em uma chamada que nÃ£o Ã© trivial de
disparar pelo navegador. AÃ­ vale a pena ler o JS bundled. Ele estÃ¡ minificado
(uma Ãºnica linha gigante), mas Ã© texto comum:

```bash
# Baixa o bundle e mede o tamanho
curl -s "https://jurisprudencia.tst.jus.br/static/js/main.be6b1d66.js" -o /tmp/main.js
wc -c /tmp/main.js     # 1.155.840 bytes (~1MB)

# Lista todos os endpoints REST mencionados
grep -oE "(rest|api)/[a-zA-Z_/-]+" /tmp/main.js | sort -u
```

SaÃ­da:
```
rest/assuntos
rest/classes-processuais
rest/convocados
rest/indicadores
rest/ministros
rest/orgaos-judicantes
rest/pesquisa-textual/
```

**Em 5 segundos eu jÃ¡ tinha um mapa da API toda.** Esse Ã© um padrÃ£o que se
aplica em quase todo SPA: as URLs ficam como strings literais dentro do JS.
Variantes desse mesmo `grep`:

```bash
# Pega URLs absolutas
grep -oE "https?://[^\"']+" /tmp/main.js | sort -u

# Pega trechos com "fetch(" ou "axios."
grep -oE "fetch\([^)]+\)" /tmp/main.js | head
grep -oE "axios\.[a-z]+\([^)]+\)" /tmp/main.js | head
```

### Passo 3.5 â Encontrar a base_url (configuraÃ§Ã£o)

No JS aparecia a expressÃ£o `config_easy.get("base_url")`, mas a string da URL
nÃ£o estava lÃ¡. Isso Ã© um padrÃ£o comum: a URL base fica em um arquivo de
configuraÃ§Ã£o para permitir ambiente de homologaÃ§Ã£o/produÃ§Ã£o.

```bash
# Procurar pela leitura de configuraÃ§Ã£o
grep -oE "fetch\(\"/[a-z._-]+" /tmp/main.js
# â fetch("/config.json")

curl -s "https://jurisprudencia.tst.jus.br/config.json"
```

SaÃ­da:
```json
{
  "base_url": "https://jurisprudencia-backend2.tst.jus.br",
  ...
}
```

> **PadrÃ£o recorrente:** quando o JS nÃ£o tem a URL hard-coded, procure por
> `/config.json`, `/env.js`, `/settings.json`, `meta` tags com
> `data-api-url`, ou o `localStorage`.

### Passo 3.6 â Reconstruir o body de um POST

A `pesquisa-textual` Ã© POST com JSON. Para descobrir o formato eu fiz duas
coisas em paralelo:

**(a)** Olhei o **Payload** no DevTools quando o site fez a busca. Isso Ã© o
caminho mais rÃ¡pido. **Cole o JSON copiado direto no Python.**

**(b)** Procurei no JS onde o body Ã© construÃ­do. Isso ajuda a entender quais
campos podem mudar:

```bash
python3 -c "
content = open('/tmp/main.js').read()
i = content.find('pesquisa-textual')
print(content[i-300:i+400])
"
```

Trecho relevante:
```js
let s = {
  ou: this.state.operadorOu,
  e: this.state.operadorE,
  termoExato: this.state.operadorExpressaoExata,
  ...
  tipos: o,
  publicacaoInicial: this.state.publicacaoInicial,
  ...
};
fetch(`${base_url}/rest/pesquisa-textual/` + (n*r+1) + "/" + r + "?a=" + Math.random(), {
  method: "POST",
  headers: {"Content-Type":"application/json"},
  body: JSON.stringify(s)
}).then(...)
```

**TrÃªs coisas crÃ­ticas reveladas aqui:**
1. **Estrutura da URL:** `pesquisa-textual/{startIndex}/{pageSize}` com
   `startIndex = pÃ¡gina * tamanho + 1` (1-indexed!).
2. **Cache-buster:** o `?a=Math.random()` impede caching pelo navegador. No
   Python, eu reproduzi com `random.random()`.
3. **Forma do body:** dicionÃ¡rio com 18 chaves; cada uma vinha do estado React.

### Passo 3.7 â Mapear cada campo do body com seu valor "default"

Quando errei o body na primeira tentativa (HTTP 400), voltei ao JS para achar
o estado inicial:

```bash
python3 -c "
content = open('/tmp/main.js').read()
i = content.find('this.state={indicePaginaAtual:0')
print(content[i:i+1500])
"
```

SaÃ­da relevante:
```js
this.state = {
  ...
  processo: {numero:"", ano:"", digito:"", orgao:"5", tribunal:"", vara:""},
  publicacaoInicial: this.getValorVazioParaCampoDeData(),
  ...
}
// E:
getValorVazioParaCampoDeData = () => isDataSuportada ? "" : null
```

**Descobri dois bugs do meu primeiro payload:**
- `numeracaoUnica` Ã© um **objeto**, nÃ£o string vazia.
- Datas vazias devem ser **`null`**, nÃ£o `""`.

> **PadrÃ£o:** quando uma API retorna 400, *quase sempre* Ã© um problema de
> tipagem (lista Ã string Ã objeto Ã null). Olhe como o JS prepara os campos.

### Passo 3.8 â Testar uma chamada real e validar

Com tudo corrigido:

```bash
curl -s -X POST ".../pesquisa-textual/1/3?a=0.123" \
     -H "Content-Type: application/json" \
     -H "Origin: https://jurisprudencia.tst.jus.br" \
     --data @payload.json -o resp.json -w "%{http_code}"
# â 200
```

AÃ­ inspecionei a resposta para entender a estrutura:

```python
import json
d = json.load(open('resp.json'))
print(list(d.keys()))                   # ['totalRegistros','registros','agregacoes',...]
print(len(d['registros']))              # 3
print(list(d['registros'][0].keys()))   # ['registro','destaques']
print(list(d['registros'][0]['registro'].keys()))
# â ['id', 'numero', 'tipo', 'numeracaoUnica', 'dtaPublicacao',
#    'txtConteudoDecisao', 'orgaoJudicante', 'nomRelator', ...]
```

**Achado de ouro:** o campo `txtConteudoDecisao` jÃ¡ contÃ©m o inteiro teor. Eu
ia precisar de uma chamada por documento; agora nÃ£o preciso.

### Passo 3.9 â Iterar (paginar) com seguranÃ§a

Agora que uma chamada funciona, Ã© sÃ³ iterar:

```python
inicio = 1
tamanho = 100
while True:
    resp = requests.post(
        f"{BASE}/rest/pesquisa-textual/{inicio}/{tamanho}",
        json=filtros, headers=HEADERS, params={"a": random.random()}
    ).json()
    for envelope in resp["registros"]:
        yield envelope["registro"]
    inicio += len(resp["registros"])
    if inicio - 1 >= resp["totalRegistros"]:
        break
    time.sleep(random.uniform(0.8, 1.6))   # respeita o servidor
```

> **Por que aleatÃ³rio?** Se vocÃª dorme exatamente 1.0s, fica Ã³bvio que Ã© um
> bot e alguns servidores ativam rate limit mais agressivo. Variar entre
> 0,8 e 1,6 simula tempo humano.

---

## 4. PadrÃµes que se repetem em outros sites

### 4.1 Como descobrir se um site tem API

Use sempre essa sequÃªncia:
1. F12 â Rede â recarregar a pÃ¡gina â filtrar por **Fetch/XHR**.
2. Se aparecem chamadas com `application/json`, hÃ¡ API.
3. Se sÃ³ aparece `text/html`, Ã© provÃ¡vel que seja HTML server-side.

### 4.2 ConvenÃ§Ãµes comuns de URL

| URL                                      | O que costuma ser                |
|------------------------------------------|----------------------------------|
| `/api/v1/...`                            | API REST versionada              |
| `/rest/...`                              | API REST (Java EE, Spring)       |
| `/graphql`                               | GraphQL â body Ãºnico, queries    |
| `/_next/data/...`                        | Next.js (React server-side)      |
| `/wp-json/...`                           | WordPress                        |

### 4.3 MÃ©todos HTTP

| MÃ©todo  | Para quÃª                                                |
|---------|----------------------------------------------------------|
| GET     | Buscar dados (parÃ¢metros vÃ£o na URL via querystring)     |
| POST    | Criar / consultas que precisam de body grande            |
| PUT/PATCH | Atualizar (raro em raspagem pÃºblica)                  |
| DELETE  | Apagar (idem)                                            |

### 4.4 CÃ³digos de status Ãºteis

| CÃ³digo | Significado                                              |
|--------|-----------------------------------------------------------|
| 200    | OK                                                        |
| 204    | OK, sem conteÃºdo                                          |
| 301/302| Redirecionamento (use `-L` no curl ou siga manualmente)   |
| 400    | Body errado (tipos, campos faltando)                     |
| 401/403| Falta autenticaÃ§Ã£o ou sem permissÃ£o                       |
| 404    | URL errada                                                |
| 406    | Content-Type pedido nÃ£o Ã© suportado                       |
| 429    | Rate-limit â diminua o ritmo                              |
| 500/503| Servidor falhou                                           |

### 4.5 Mecanismos de autenticaÃ§Ã£o que vocÃª vai encontrar

| Mecanismo            | Como reconhecer                              | Como reproduzir em Python              |
|----------------------|------------------------------------------------|------------------------------------------|
| Cookie de sessÃ£o     | Header `Set-Cookie` no login                   | `requests.Session()`                     |
| `Authorization: Bearer ...` | Header com JWT                          | Add header manualmente                   |
| CSRF token           | Campo escondido no HTML + header `X-CSRF-...` | Pegar do HTML antes do POST              |
| reCAPTCHA            | PÃ¡gina com checkbox do Google                  | Geralmente bloqueia raspagem; reavaliar  |

---

## 5. Como se transforma isso tudo em Python organizado

A regra Ã© **separar responsabilidades**:

```python
# 1. CLIENTE: sÃ³ sabe falar HTTP, sem regras de negÃ³cio
class TSTClient:
    def search_page(self, payload, inicio, tamanho): ...
    def get_document_html(self, doc_id): ...

# 2. ITERADOR: paginaÃ§Ã£o e controle de fluxo
def iter_records(client, filtros, page_size, max_records): ...

# 3. NORMALIZAÃÃO: transformar o JSON cru em dicionÃ¡rio "limpo"
def flatten_record(reg): ...

# 4. SAÃDA: como gravar (Excel, CSV, banco...)
def write_workbook(rows, output): ...

# 5. ORQUESTRAÃÃO: o "main" sÃ³ amarra os outros
def main():
    client = TSTClient()
    rows = [flatten_record(r) for r in iter_records(client, FILTROS)]
    write_workbook(rows, "saida.xlsx")
```

**Por quÃª?** Se amanhÃ£ o TST mudar o nome de um campo, vocÃª corrige sÃ³ em
`flatten_record`. Se vocÃª quiser CSV em vez de Excel, troca sÃ³ `write_workbook`.
Essa separaÃ§Ã£o evita o "bolo de cÃ³digo" que fica intratÃ¡vel.

---

## 6. Boas prÃ¡ticas e armadilhas

### 6.1 Ãtico e jurÃ­dico
- **Leia o `robots.txt`** do site (`https://site/robots.txt`). Ele nÃ£o tem
  forÃ§a legal direta, mas sinaliza a vontade do publicador.
- **Respeite os Termos de Uso.** Sites pÃºblicos costumam permitir consulta
  individual; raspagem em massa pode ser vedada.
- **NÃ£o derrube o servidor.** ConcorrÃªncia alta + intervalos curtos = abuso.
  Para a maioria dos casos, 1 requisiÃ§Ã£o a cada 1â2 segundos Ã© educado.
- **Identifique-se.** Configure um `User-Agent` que diga quem vocÃª Ã© (ex.:
  `"Escritorio Soares Webscraper / contato@example.com"`). Em sistemas
  pÃºblicos, isso evita bloqueio reflexivo.

### 6.2 Robustez tÃ©cnica
- **Sempre use `requests.Session()`** â reaproveita conexÃ£o TCP.
- **Sempre faÃ§a retry com backoff exponencial** para erros transitÃ³rios.
- **Log em arquivo** (mÃ³dulo `logging`) â vocÃª quer saber o que aconteceu se a
  raspagem rodar Ã  noite.
- **Salve em modo "append" / por lotes** â se o script cair na requisiÃ§Ã£o 800
  de 1.137, vocÃª nÃ£o quer perder as 800.
- **Hash idempotente** â use o `id` do registro como chave Ãºnica para detectar
  duplicatas em re-execuÃ§Ãµes.

### 6.3 Armadilhas frequentes
- **Caracteres invisÃ­veis em copy/paste de payload** â ao colar o JSON do
  DevTools, Ã s vezes vÃªm aspas curvas (`"` em vez de `"`). Sempre passe pelo
  `json.loads()`.
- **Datas em formatos diferentes** â `null`, `""`, `"2024-01-01"`,
  `"01/01/2024"`. Veja como o front envia.
- **Limite de cÃ©lulas do Excel** â 32.767 caracteres. Para inteiros teores,
  trunque ou salve em arquivo.
- **Cache do navegador iludindo o teste** â quando o curl funciona mas o JS
  parece travado, pode ser cache. Use `?a=random` ou `Cache-Control: no-cache`.

---

## 7. Aprofundamento sugerido

| Tema                           | Onde estudar                                         |
|--------------------------------|------------------------------------------------------|
| HTTP por baixo                 | "HTTP: The Definitive Guide" (Gourley) ou MDN         |
| Requests em Python             | <https://docs.python-requests.org/>                  |
| BeautifulSoup (quando o HTML Ã© o produto) | <https://beautiful-soup-4.readthedocs.io/> |
| Regex (Ãºteis em todo lugar)    | <https://regex101.com> (com tester ao vivo)          |
| Selenium/Playwright (quando NÃO hÃ¡ API) | <https://playwright.dev/python/>            |
| Tribunais & dados pÃºblicos     | Manual da CNJ / DataJud para padronizaÃ§Ã£o de dados   |

> **PrÃ¡tica que mais consolida:** escolha um site jurÃ­dico que vocÃª usa
> diariamente, abra o DevTools, e faÃ§a o **Passo 3.2** (mapear o que ele
> chama). Em 30 minutos vocÃª vai conseguir descrever a "API por trÃ¡s" de
> qualquer um dos tribunais que mais te interessa.

---

## 8. Checklist mental (decore esse)

Quando for raspar um site novo, siga sempre nessa ordem:

1. [ ] Ã site tradicional ou SPA? (`Ctrl+U` mostra dados ou sÃ³ `<div id="root">`?)
2. [ ] Existe API? (F12 â Rede â tem chamadas Fetch/XHR retornando JSON?)
3. [ ] Listei todas as URLs do tipo `/api/...` ou `/rest/...`?
4. [ ] Identifiquei o mÃ©todo (GET/POST), os headers e o body de cada uma?
5. [ ] Reproduzi pelo menos uma chamada via `curl` no terminal?
6. [ ] Entendi a paginaÃ§Ã£o? Onde vem `total`, `page`, `offset`?
7. [ ] Validei a resposta com um JSON pequeno antes de iterar?
8. [ ] Tenho retry, sleep e log no script Python?
9. [ ] Salvo o resultado em formato Ãºtil para *meu* fluxo (Excel, CSV, BD)?
10. [ ] Testei com `-n 5` antes de soltar a rodada cheia?

Se vocÃª marcou todos, estÃ¡ pronto para rodar.

---

*Este Ã© o mÃ©todo. A diferenÃ§a entre "saber programar Python" e "conseguir
raspar qualquer site" estÃ¡ nessa investigaÃ§Ã£o prÃ©via, nÃ£o em truques de
cÃ³digo. Boa caÃ§a.*
