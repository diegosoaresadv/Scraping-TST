# Como investigar a API de um site — método passo-a-passo

Este documento desconstrói o que foi feito para descobrir a API da Pesquisa de
Jurisprudência do TST. O objetivo é que você consiga repetir esse processo em
qualquer site (e-SAJ, TJs, JusBrasil, Receita, sistemas internos do escritório,
etc.). A lógica é sempre a mesma; só os detalhes mudam.

---

## 1. O modelo mental: HTML estático × SPA × API

Antes de qualquer código, é preciso entender em que tipo de site você está.

### 1.1 Site "tradicional" (HTML server-side)
O servidor monta a página inteira em HTML e envia pronta. Quando você dá
`Ctrl + U` (ver código-fonte), os dados que aparecem na tela já estão lá.
Exemplo clássico: a busca pública do TJSP, em parte, é assim. Para esses
sites, basta usar `requests` + `BeautifulSoup` — pegar o HTML e extrair com
seletores CSS.

### 1.2 SPA (Single Page Application)
O servidor entrega uma "casca" mínima de HTML e um arquivo JavaScript grande.
Esse JavaScript roda no navegador, faz uma chamada (fetch) a uma **API**, e só
depois "pinta" a tela com os dados. Quando você faz `Ctrl + U`, vê quase nada
— porque os dados ainda nem chegaram. **O TST é uma SPA em React.**

> **Sinal claro de SPA:** o `Ctrl + U` mostra um `<div id="root"></div>` quase
> vazio e uma tag `<script>` apontando para `main.[hash].js`.

### 1.3 API (Application Programming Interface)
É um endpoint HTTP que devolve dados estruturados (geralmente JSON), em vez de
HTML. É *o que a SPA usa por baixo*. **Quando você descobre a API, descobre a
maneira mais rápida e estável de raspar o site** — porque você fala com o
servidor na mesma linguagem em que ele se comunica com o JS dele.

> **Regra de ouro:** sempre que possível, raspe a API, não a tela.

---

## 2. As ferramentas

| Ferramenta              | Para quê                                      |
|-------------------------|------------------------------------------------|
| **DevTools do navegador** (F12) | Ver requisições, payloads e respostas.   |
| **`curl`** (terminal)   | Replicar uma chamada HTTP fora do navegador.   |
| **`grep`** / `find` / editor | Buscar dentro de arquivos (inclusive do JS).   |
| **Python** (`requests`, `json`) | Automatizar e iterar páginas.            |
| **Postman/Insomnia** (opcional) | GUI para testar APIs sem escrever curl.  |

Você não precisa de tudo de uma vez. Comece com **DevTools + curl + Python**.

---

## 3. Roteiro de investigação aplicado ao TST

Vou recriar a sequência exata de passos. Cada subseção tem o **comando que
rodei** e o **insight que ganhei**.

### Passo 3.1 — Confirmar que é uma SPA

```bash
curl -s -L "https://jurisprudencia.tst.jus.br/" | head -c 1500
```

Saída (resumida):
```html
<!doctype html><html lang="pt-br">...
<title>Pesquisa de jurisprudência</title>
<script defer src="/static/js/main.be6b1d66.js"></script>
...
<div id="root"></div>
```

**Conclusão:** SPA confirmada. O HTML não tem dados; eles virão via JavaScript.
A "alma" do site está em `main.be6b1d66.js`.

### Passo 3.2 — Olhar o tráfego de rede no navegador

Esse é o passo mais importante. Sem código.

1. Abra a página no Firefox/Chrome.
2. Pressione **F12**, vá na aba **Rede / Network**.
3. Marque o filtro **Fetch/XHR** (só requisições de dados).
4. Clique em "Pesquisar" no formulário.
5. Cada linha que aparecer é uma chamada à API.

Você verá algo como:
```
POST  rest/pesquisa-textual/1/100?a=0.345...   200 OK   1.4 MB   2.3s
```

Clique nela e olhe três abas:

- **Cabeçalhos / Headers** — URL completa, método, headers enviados.
- **Carga útil / Payload** — o JSON que o navegador enviou (filtros).
- **Resposta / Response** — o JSON que voltou.

> **Esse é o "santo graal" da investigação.** Você acabou de mapear:
> URL + método + headers + body + resposta. É tudo que o Python precisa
> imitar. (No nosso caso, o user descobriu o endpoint de inteiro teor pela
> aba **Inspetor** olhando a `<a>` do botão "Inteiro Teor".)

### Passo 3.3 — Replicar a chamada com `curl`

Com a chamada selecionada no DevTools, clique com o botão direito → **Copiar
como cURL**. Cole no terminal. Funciona? Ótimo — você reproduziu um cliente
fora do navegador.

No nosso caso, eu fiz o curl à mão para ter mais controle:

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
| `-s`                           | "silent" — esconde a barra de progresso              |
| `-X POST`                      | método HTTP (default é GET)                          |
| `-H "..."`                     | adiciona um header                                   |
| `--data @payload.json`         | envia o conteúdo do arquivo como body                |
| `-w "%{http_code}"`            | imprime o código de status no final                  |

> **Headers que costumam ser exigidos:** `Content-Type`, `Origin`, `Referer`,
> `User-Agent`. Quando uma API recusa (HTTP 400/403), 90% das vezes é porque
> falta um deles ou o `Content-Type` está errado.

### Passo 3.4 — Quando não há um botão evidente: ler o JavaScript

Às vezes o que você precisa só aparece em uma chamada que não é trivial de
disparar pelo navegador. Aí vale a pena ler o JS bundled. Ele está minificado
(uma única linha gigante), mas é texto comum:

```bash
# Baixa o bundle e mede o tamanho
curl -s "https://jurisprudencia.tst.jus.br/static/js/main.be6b1d66.js" -o /tmp/main.js
wc -c /tmp/main.js     # 1.155.840 bytes (~1MB)

# Lista todos os endpoints REST mencionados
grep -oE "(rest|api)/[a-zA-Z_/-]+" /tmp/main.js | sort -u
```

Saída:
```
rest/assuntos
rest/classes-processuais
rest/convocados
rest/indicadores
rest/ministros
rest/orgaos-judicantes
rest/pesquisa-textual/
```

**Em 5 segundos eu já tinha um mapa da API toda.** Esse é um padrão que se
aplica em quase todo SPA: as URLs ficam como strings literais dentro do JS.
Variantes desse mesmo `grep`:

```bash
# Pega URLs absolutas
grep -oE "https?://[^\"']+" /tmp/main.js | sort -u

# Pega trechos com "fetch(" ou "axios."
grep -oE "fetch\([^)]+\)" /tmp/main.js | head
grep -oE "axios\.[a-z]+\([^)]+\)" /tmp/main.js | head
```

### Passo 3.5 — Encontrar a base_url (configuração)

No JS aparecia a expressão `config_easy.get("base_url")`, mas a string da URL
não estava lá. Isso é um padrão comum: a URL base fica em um arquivo de
configuração para permitir ambiente de homologação/produção.

```bash
# Procurar pela leitura de configuração
grep -oE "fetch\(\"/[a-z._-]+" /tmp/main.js
# → fetch("/config.json")

curl -s "https://jurisprudencia.tst.jus.br/config.json"
```

Saída:
```json
{
  "base_url": "https://jurisprudencia-backend2.tst.jus.br",
  ...
}
```

> **Padrão recorrente:** quando o JS não tem a URL hard-coded, procure por
> `/config.json`, `/env.js`, `/settings.json`, `meta` tags com
> `data-api-url`, ou o `localStorage`.

### Passo 3.6 — Reconstruir o body de um POST

A `pesquisa-textual` é POST com JSON. Para descobrir o formato eu fiz duas
coisas em paralelo:

**(a)** Olhei o **Payload** no DevTools quando o site fez a busca. Isso é o
caminho mais rápido. **Cole o JSON copiado direto no Python.**

**(b)** Procurei no JS onde o body é construído. Isso ajuda a entender quais
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

**Três coisas críticas reveladas aqui:**
1. **Estrutura da URL:** `pesquisa-textual/{startIndex}/{pageSize}` com
   `startIndex = página * tamanho + 1` (1-indexed!).
2. **Cache-buster:** o `?a=Math.random()` impede caching pelo navegador. No
   Python, eu reproduzi com `random.random()`.
3. **Forma do body:** dicionário com 18 chaves; cada uma vinha do estado React.

### Passo 3.7 — Mapear cada campo do body com seu valor "default"

Quando errei o body na primeira tentativa (HTTP 400), voltei ao JS para achar
o estado inicial:

```bash
python3 -c "
content = open('/tmp/main.js').read()
i = content.find('this.state={indicePaginaAtual:0')
print(content[i:i+1500])
"
```

Saída relevante:
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
- `numeracaoUnica` é um **objeto**, não string vazia.
- Datas vazias devem ser **`null`**, não `""`.

> **Padrão:** quando uma API retorna 400, *quase sempre* é um problema de
> tipagem (lista × string × objeto × null). Olhe como o JS prepara os campos.

### Passo 3.8 — Testar uma chamada real e validar

Com tudo corrigido:

```bash
curl -s -X POST ".../pesquisa-textual/1/3?a=0.123" \
     -H "Content-Type: application/json" \
     -H "Origin: https://jurisprudencia.tst.jus.br" \
     --data @payload.json -o resp.json -w "%{http_code}"
# → 200
```

Aí inspecionei a resposta para entender a estrutura:

```python
import json
d = json.load(open('resp.json'))
print(list(d.keys()))                   # ['totalRegistros','registros','agregacoes',...]
print(len(d['registros']))              # 3
print(list(d['registros'][0].keys()))   # ['registro','destaques']
print(list(d['registros'][0]['registro'].keys()))
# → ['id', 'numero', 'tipo', 'numeracaoUnica', 'dtaPublicacao',
#    'txtConteudoDecisao', 'orgaoJudicante', 'nomRelator', ...]
```

**Achado de ouro:** o campo `txtConteudoDecisao` já contém o inteiro teor. Eu
ia precisar de uma chamada por documento; agora não preciso.

### Passo 3.9 — Iterar (paginar) com segurança

Agora que uma chamada funciona, é só iterar:

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

> **Por que aleatório?** Se você dorme exatamente 1.0s, fica óbvio que é um
> bot e alguns servidores ativam rate limit mais agressivo. Variar entre
> 0,8 e 1,6 simula tempo humano.

---

## 4. Padrões que se repetem em outros sites

### 4.1 Como descobrir se um site tem API

Use sempre essa sequência:
1. F12 → Rede → recarregar a página → filtrar por **Fetch/XHR**.
2. Se aparecem chamadas com `application/json`, há API.
3. Se só aparece `text/html`, é provável que seja HTML server-side.

### 4.2 Convenções comuns de URL

| URL                                      | O que costuma ser                |
|------------------------------------------|----------------------------------|
| `/api/v1/...`                            | API REST versionada              |
| `/rest/...`                              | API REST (Java EE, Spring)       |
| `/graphql`                               | GraphQL — body único, queries    |
| `/_next/data/...`                        | Next.js (React server-side)      |
| `/wp-json/...`                           | WordPress                        |

### 4.3 Métodos HTTP

| Método  | Para quê                                                |
|---------|----------------------------------------------------------|
| GET     | Buscar dados (parâmetros vão na URL via querystring)     |
| POST    | Criar / consultas que precisam de body grande            |
| PUT/PATCH | Atualizar (raro em raspagem pública)                  |
| DELETE  | Apagar (idem)                                            |

### 4.4 Códigos de status úteis

| Código | Significado                                              |
|--------|-----------------------------------------------------------|
| 200    | OK                                                        |
| 204    | OK, sem conteúdo                                          |
| 301/302| Redirecionamento (use `-L` no curl ou siga manualmente)   |
| 400    | Body errado (tipos, campos faltando)                     |
| 401/403| Falta autenticação ou sem permissão                       |
| 404    | URL errada                                                |
| 406    | Content-Type pedido não é suportado                       |
| 429    | Rate-limit — diminua o ritmo                              |
| 500/503| Servidor falhou                                           |

### 4.5 Mecanismos de autenticação que você vai encontrar

| Mecanismo            | Como reconhecer                              | Como reproduzir em Python              |
|----------------------|------------------------------------------------|------------------------------------------|
| Cookie de sessão     | Header `Set-Cookie` no login                   | `requests.Session()`                     |
| `Authorization: Bearer ...` | Header com JWT                          | Add header manualmente                   |
| CSRF token           | Campo escondido no HTML + header `X-CSRF-...` | Pegar do HTML antes do POST              |
| reCAPTCHA            | Página com checkbox do Google                  | Geralmente bloqueia raspagem; reavaliar  |

---

## 5. Como se transforma isso tudo em Python organizado

A regra é **separar responsabilidades**:

```python
# 1. CLIENTE: só sabe falar HTTP, sem regras de negócio
class TSTClient:
    def search_page(self, payload, inicio, tamanho): ...
    def get_document_html(self, doc_id): ...

# 2. ITERADOR: paginação e controle de fluxo
def iter_records(client, filtros, page_size, max_records): ...

# 3. NORMALIZAÇÃO: transformar o JSON cru em dicionário "limpo"
def flatten_record(reg): ...

# 4. SAÍDA: como gravar (Excel, CSV, banco...)
def write_workbook(rows, output): ...

# 5. ORQUESTRAÇÃO: o "main" só amarra os outros
def main():
    client = TSTClient()
    rows = [flatten_record(r) for r in iter_records(client, FILTROS)]
    write_workbook(rows, "saida.xlsx")
```

**Por quê?** Se amanhã o TST mudar o nome de um campo, você corrige só em
`flatten_record`. Se você quiser CSV em vez de Excel, troca só `write_workbook`.
Essa separação evita o "bolo de código" que fica intratável.

---

## 6. Boas práticas e armadilhas

### 6.1 Ético e jurídico
- **Leia o `robots.txt`** do site (`https://site/robots.txt`). Ele não tem
  força legal direta, mas sinaliza a vontade do publicador.
- **Respeite os Termos de Uso.** Sites públicos costumam permitir consulta
  individual; raspagem em massa pode ser vedada.
- **Não derrube o servidor.** Concorrência alta + intervalos curtos = abuso.
  Para a maioria dos casos, 1 requisição a cada 1–2 segundos é educado.
- **Identifique-se.** Configure um `User-Agent` que diga quem você é (ex.:
  `"Escritorio Soares Webscraper / contato@example.com"`). Em sistemas
  públicos, isso evita bloqueio reflexivo.

### 6.2 Robustez técnica
- **Sempre use `requests.Session()`** — reaproveita conexão TCP.
- **Sempre faça retry com backoff exponencial** para erros transitórios.
- **Log em arquivo** (módulo `logging`) — você quer saber o que aconteceu se a
  raspagem rodar à noite.
- **Salve em modo "append" / por lotes** — se o script cair na requisição 800
  de 1.137, você não quer perder as 800.
- **Hash idempotente** — use o `id` do registro como chave única para detectar
  duplicatas em re-execuções.

### 6.3 Armadilhas frequentes
- **Caracteres invisíveis em copy/paste de payload** — ao colar o JSON do
  DevTools, às vezes vêm aspas curvas (`"` em vez de `"`). Sempre passe pelo
  `json.loads()`.
- **Datas em formatos diferentes** — `null`, `""`, `"2024-01-01"`,
  `"01/01/2024"`. Veja como o front envia.
- **Limite de células do Excel** — 32.767 caracteres. Para inteiros teores,
  trunque ou salve em arquivo.
- **Cache do navegador iludindo o teste** — quando o curl funciona mas o JS
  parece travado, pode ser cache. Use `?a=random` ou `Cache-Control: no-cache`.

---

## 7. Aprofundamento sugerido

| Tema                           | Onde estudar                                         |
|--------------------------------|------------------------------------------------------|
| HTTP por baixo                 | "HTTP: The Definitive Guide" (Gourley) ou MDN         |
| Requests em Python             | <https://docs.python-requests.org/>                  |
| BeautifulSoup (quando o HTML é o produto) | <https://beautiful-soup-4.readthedocs.io/> |
| Regex (úteis em todo lugar)    | <https://regex101.com> (com tester ao vivo)          |
| Selenium/Playwright (quando NÃO há API) | <https://playwright.dev/python/>            |
| Tribunais & dados públicos     | Manual da CNJ / DataJud para padronização de dados   |

> **Prática que mais consolida:** escolha um site jurídico que você usa
> diariamente, abra o DevTools, e faça o **Passo 3.2** (mapear o que ele
> chama). Em 30 minutos você vai conseguir descrever a "API por trás" de
> qualquer um dos tribunais que mais te interessa.

---

## 8. Checklist mental (decore esse)

Quando for raspar um site novo, siga sempre nessa ordem:

1. [ ] É site tradicional ou SPA? (`Ctrl+U` mostra dados ou só `<div id="root">`?)
2. [ ] Existe API? (F12 → Rede → tem chamadas Fetch/XHR retornando JSON?)
3. [ ] Listei todas as URLs do tipo `/api/...` ou `/rest/...`?
4. [ ] Identifiquei o método (GET/POST), os headers e o body de cada uma?
5. [ ] Reproduzi pelo menos uma chamada via `curl` no terminal?
6. [ ] Entendi a paginação? Onde vem `total`, `page`, `offset`?
7. [ ] Validei a resposta com um JSON pequeno antes de iterar?
8. [ ] Tenho retry, sleep e log no script Python?
9. [ ] Salvo o resultado em formato útil para *meu* fluxo (Excel, CSV, BD)?
10. [ ] Testei com `-n 5` antes de soltar a rodada cheia?

Se você marcou todos, está pronto para rodar.

---

*Este é o método. A diferença entre "saber programar Python" e "conseguir
raspar qualquer site" está nessa investigação prévia, não em truques de
código. Boa caça.*
