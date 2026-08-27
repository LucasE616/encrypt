# `sasha.html` — Documentação técnica

Documentação referente **exclusivamente** ao arquivo original `sasha.html` (single-file, ~200 linhas, HTML + CSS + JS puro, sem dependências externas e sem build step).

## Índice

- [O que o arquivo faz](#o-que-o-arquivo-faz)
- [Estrutura do arquivo](#estrutura-do-arquivo)
- [Estado embutido (`DATA_*`)](#estado-embutido-data_)
- [Interface (HTML)](#interface-html)
- [Estilos (CSS)](#estilos-css)
- [Lógica (JavaScript)](#lógica-javascript)
  - [Verificação de suporte do navegador](#verificação-de-suporte-do-navegador)
  - [Exibição condicional da caixa "Open this file"](#exibição-condicional-da-caixa-open-this-file)
  - [Funções principais](#funções-principais)
  - [Funções de fluxo](#funções-de-fluxo)
  - [Funções utilitárias (`util`)](#funções-utilitárias-util)
- [Fluxo de dados ponta a ponta](#fluxo-de-dados-ponta-a-ponta)
- [Considerações de segurança](#considerações-de-segurança)

## O que o arquivo faz

`sasha.html` é ao mesmo tempo o programa e o "molde" do arquivo criptografado que ele gera. Ao ser aberto no navegador, permite:

1. Criptografar um arquivo local com uma senha (AES-GCM), gerando um **novo arquivo HTML** (`EncryptedFile.html`) que contém o dado cifrado embutido em seu próprio código-fonte.
2. Se o próprio `sasha.html` já contiver dados embutidos (ou seja, se for um `EncryptedFile.html` gerado anteriormente), descriptografar e baixar/abrir o conteúdo original a partir da senha correta.

Não há backend, servidor ou envio de rede: toda a criptografia roda no navegador via `window.crypto.subtle` (Web Crypto API).

## Estrutura do arquivo

```
<html>
├─ <head>
│   ├─ <style>                  → CSS inline (linha 5)
│   └─ <script>                 → declaração das 3 variáveis DATA_* (linhas 7-11)
└─ <body>
    ├─ <div class="box">        → formulário "New file" (linhas 14-23)
    ├─ <div id="thisData">      → formulário "Open this file" (linhas 24-32)
    └─ <script>                 → toda a lógica da aplicação (linhas 33-199)
```

## Estado embutido (`DATA_*`)

Declaradas vazias no `<head>` do arquivo original (linhas 8-10):

```js
var DATA_STRING_HEX = ''; // dado criptografado, guardado em base64 (apesar do nome sugerir hex)
var DATA_IV = '';         // IV do AES-GCM, em base64
var DATA_TYPE = '';       // MIME type original do arquivo (ex.: image/png)
```

Essas três linhas são o alvo de uma substituição por regex (função `createVaultFile`, linhas 94-102): quando um arquivo é criptografado, o script relê o `innerHTML` da tag `<html>` inteira e troca essas três declarações pelos valores gerados, produzindo a string completa de um novo HTML — que é então baixado. É assim que `sasha.html` "clona a si mesmo" a cada uso.

## Interface (HTML)

Duas caixas (`<div class="box">`):

1. **"New file"** (linhas 14-23)
   - `<input id="password-new" type="password">` — senha a ser usada na criptografia.
   - `<input type="file" id="fileRead" onChange="sashaIn(this);" multiple>` — dispara `sashaIn()` ao selecionar arquivo(s). Apesar do atributo `multiple`, o código de `getFile` (linha 66/68) só lê `element.files[0]` — ou seja, **apenas o primeiro arquivo selecionado é processado**.

2. **"Open this file"** (linhas 24-32, `id="thisData"`)
   - `<input id="password-file" type="password">` — senha para descriptografar.
   - `<input type="button" onClick="sashaOut()" value="View">` — dispara `sashaOut()`.
   - Esta caixa é escondida/mostrada dinamicamente (ver abaixo) dependendo se o HTML atual já tem dados embutidos.

## Estilos (CSS)

Bloco único, minificado, na linha 5:

```css
body, html { display:flex; align-items:center; justify-content:center; height:100%; font-family:sans-serif }
.box { background-color:#ccc; text-align:center; margin:4px; padding:30px }
input[type="button"] { margin:4px }
```

Centraliza as duas caixas na tela (flexbox) e dá um fundo cinza (`#ccc`) com padding a cada `.box`.

## Lógica (JavaScript)

### Verificação de suporte do navegador

```js
if (!window.File && !window.FileReader && !window.FileList && !window.Blob) {
	alert('Erro: Your browser does not suport Sasha.');
}
```

Linha 36-38. Nota: os operadores são `&&`, então o alerta só dispara se **todas** as quatro APIs estiverem ausentes — se só uma faltar, nenhum aviso aparece (comportamento provavelmente não intencional, mas é o que o código original faz).

### Exibição condicional da caixa "Open this file"

```js
DATA_STRING_HEX=='' ? document.getElementById('thisData').hidden = true : document.getElementById('thisData').hidden = false
```

Linha 40. Se `DATA_STRING_HEX` estiver vazio (arquivo "limpo", sem dado embutido), a caixa "Open this file" fica oculta. Se já houver dado embutido (arquivo é um cofre gerado), a caixa aparece.

### Funções principais

| Função | Assinatura | Linha | Descrição |
|---|---|---|---|
| `sashaIn(element)` | `(HTMLInputElement) => void` | 43-49 | Orquestra a criptografia: `getFile → encriptFile → createVaultFile → downloadVaultFile`, com `.catch` final que apenas loga o erro no console. |
| `sashaOut()` | `() => void` | 51-59 | Orquestra a descriptografia: `decriptFile → downloadFile`. Tem um `.catch` intermediário que dispara `alert("Incorrect password.")` caso a descriptografia falhe. |

### Funções de fluxo

| Função | Assinatura | Linha | Descrição |
|---|---|---|---|
| `getFile(element)` | `(HTMLInputElement) => Promise<{filedata, type}>` | 62-70 | Lê `element.files[0]` como `ArrayBuffer` via `FileReader.readAsArrayBuffer`, resolvendo com os bytes e o MIME `type`. |
| `encriptFile(file)` | `({filedata, type}) => Promise<{data, iv, type}>` | 72-81 | Lê a senha do campo `#password-new`; gera IV aleatório de 12 bytes; deriva a chave com `SHA-256(senha)`; criptografa com AES-GCM; converte resultado e IV para hex e depois base64. |
| `decriptFile(data)` | `({data, iv, type}) => Promise<ArrayBuffer>` | 83-92 | Lê a senha do campo `#password-file`; reconstrói o IV a partir do base64 embutido; deriva a mesma chave `SHA-256(senha)`; chama `crypto.subtle.decrypt`. Se a senha estiver errada, a Promise rejeita (falha de integridade do AES-GCM). |
| `createVaultFile(encriptedData)` | `({data, iv, type}) => string` | 94-102 | Pega `document.getElementsByTagName('html')[0].innerHTML` e substitui, via regex, as três declarações `DATA_STRING_HEX`, `DATA_IV` e `DATA_TYPE` pelos valores recém-gerados. Retorna a string do novo HTML completo. |
| `downloadVaultFile(data)` | `(string) => void` | 104-111 | Cria um elemento `<a>` com `href="data:/html;charset=utf-8," + encodeURIComponent(data)` e `download="EncryptedFile.html"`, clica nele programaticamente e o remove do DOM. |
| `downloadFile(decryptedData)` | `(ArrayBuffer) => void` | 113-120 | Monta um `Blob` com o `type` original (`window.DATA_TYPE`), cria uma Object URL (`URL.createObjectURL`) e clica em um `<a>` para baixar/abrir o conteúdo descriptografado. |

### Funções utilitárias (`util`)

Objeto `util` (linha 123) com wrappers finos sobre a Web Crypto API e funções de conversão de formato:

| Função | Linha | Descrição |
|---|---|---|
| `util.getRandomBytes(bytes)` | 125-127 | `crypto.getRandomValues(new Uint8Array(bytes))` — gera bytes aleatórios criptograficamente seguros (usado para o IV). |
| `util.sha256(data)` | 129-131 | `crypto.subtle.digest({name:"SHA-256"}, data)` — usado para transformar a senha (já em bytes) em uma chave de 256 bits. |
| `util.importKey(type, key, algo, isExtractable, applications)` | 133-135 | Wrapper de `crypto.subtle.importKey`. |
| `util.encrypt(algo, password, data)` / `util.decrypt(algo, password, data)` | 137-143 | Wrappers de `crypto.subtle.encrypt` / `crypto.subtle.decrypt` (o parâmetro chamado `password` aqui já é a `CryptoKey`, não a senha em texto). |
| `util.textEncoder(string)` / `util.textDecoder(ArrayBuffer)` | 145-151 | Conversão string ↔ bytes UTF-8 via `TextEncoder`/`TextDecoder`. |
| `util.buf2hex(arrayBuffer)` | 153-168 | Converte `ArrayBuffer` em string hexadecimal, byte a byte, com validação de tipo (lança `TypeError` se a entrada não for um `ArrayBuffer`). |
| `util.hex2buf(hex)` | 170-182 | Converte string hex de volta para `ArrayBuffer` (`Uint8Array.buffer`), validando tipo e paridade do comprimento da string. |
| `util.hex2base64(hex)` | 184-188 | Converte string hex em base64 (via `btoa` + manipulação de string com regex). |
| `util.base642hex(base64)` | 190-197 | Converte base64 de volta em string hex (via `atob`, byte a byte). |

## Fluxo de dados ponta a ponta

**Criptografia** (`sashaIn`):

```
arquivo selecionado
  → FileReader.readAsArrayBuffer            (getFile)
  → AES-GCM encrypt, chave = SHA-256(senha)  (encriptFile)
  → ArrayBuffer cifrado → hex → base64       (encriptFile)
  → injeta DATA_STRING_HEX/DATA_IV/DATA_TYPE
    no innerHTML da própria página           (createVaultFile)
  → download como EncryptedFile.html         (downloadVaultFile)
```

**Descriptografia** (`sashaOut`):

```
DATA_STRING_HEX/DATA_IV/DATA_TYPE (globais já embutidos no HTML)
  → base64 → hex → ArrayBuffer               (decriptFile)
  → AES-GCM decrypt, chave = SHA-256(senha)  (decriptFile)
  → Blob com o MIME type original            (downloadFile)
  → download/abertura via Object URL         (downloadFile)
```

## Considerações de segurança

Observações derivadas diretamente do código de `sasha.html`:

- **Derivação de chave**: a chave AES é `SHA-256(senha)` calculado diretamente (linhas 76 e 86) — não há salt nem KDF com custo computacional (PBKDF2, scrypt, Argon2). Isso deixa o esquema exposto a ataques de força bruta/dicionário offline contra senhas curtas ou comuns.
- **Sem iteração/custo artificial**: não existe fator de trabalho para tornar tentativas de senha mais lentas.
- **Integridade**: por usar AES-GCM, uma senha errada faz `crypto.subtle.decrypt` rejeitar a Promise (tratado em `sashaOut` com `alert("Incorrect password.")`) em vez de gerar dado corrompido silenciosamente.
- **IV**: gerado aleatoriamente por criptografia (`crypto.getRandomValues`, 12 bytes) a cada operação de `encriptFile`, como recomendado para AES-GCM.
- **Duplicação de tamanho**: a conversão para hex antes do base64 (em vez de ir direto de bytes para base64) praticamente dobra o tamanho do dado embutido em relação ao arquivo original.
- **Somente o primeiro arquivo**: apesar do `<input>` aceitar múltiplos arquivos (`multiple`), apenas `files[0]` é lido — arquivos adicionais selecionados são ignorados silenciosamente.
