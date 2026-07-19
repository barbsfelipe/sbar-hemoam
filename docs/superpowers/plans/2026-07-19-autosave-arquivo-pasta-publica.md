# Salvamento automático em arquivo na pasta pública Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** No `site/index.html`, permitir que o plantonista ative (1 clique por login do Windows, só no Chrome/Edge) um salvamento automático que grava as alterações direto num arquivo dentro da pasta pública, para que ao trocar de login e reabrir o arquivo os dados do plantão anterior apareçam sozinhos.

**Architecture:** Página estática única, sem build. Reaproveita `coletarEstado()`/`restaurarEstado(dados)` já existentes. Adiciona uma camada nova baseada na *File System Access API* (`showSaveFilePicker`, `FileSystemFileHandle`), com o handle do arquivo persistido em `IndexedDB` para reuso entre recarregamentos da página no mesmo perfil de navegador. `localStorage` e os botões manuais "Salvar/Carregar .json" continuam intactos como rede de segurança e como caminho único para navegadores sem suporte (Firefox).

**Tech Stack:** HTML/CSS/JS vanilla (sem dependências, sem framework, sem build). Sem framework de testes — verificação manual via navegador (Playwright ou Chrome), igual ao padrão já usado no plano anterior (`2026-07-15-continuidade-isbar.md`).

## Global Constraints

- Nome do arquivo do mecanismo novo: `plantão (dd-mm-aaaa).json`, usando a data de **criação** — não muda mais depois de criado.
- O botão manual "Salvar dados (.json)" já existente passa a usar esse mesmo padrão de nome (troca `isbar-utip-hemoam-AAAA-MM-DD.json` por `plantão (dd-mm-aaaa).json`).
- Detecção de suporte: `'showSaveFilePicker' in window`. Se `false`, nenhum botão do mecanismo novo é exibido — apenas os botões manuais já existentes.
- Ação destrutiva (substituir os dados da tela pelos do arquivo escolhido, quando ele já tem conteúdo) sempre pede `confirm()` antes, com o mesmo texto já usado pelo "Carregar dados (.json)": `'Isso vai substituir os dados atuais na tela pelos dados do arquivo selecionado. Continuar?'`.
- Falhas de leitura/escrita no arquivo compartilhado (rede caiu, arquivo movido/apagado, permissão revogada) nunca travam o formulário: aviso discreto + o `localStorage` continua funcionando como rede de segurança.
- O autosave em `localStorage` já existente (chave `hemoam-isbar-utip-rascunho`, debounce de 400ms) permanece intocado e ativo em paralelo, independente do mecanismo novo.
- Elementos novos ficam dentro da `.toolbar` (já oculta na impressão via `@media print { .toolbar { display: none; } }`) — não precisam de classe `.no-print` adicional.

---

## Task 1: Nome de arquivo amigável no botão "Salvar dados (.json)"

**Files:**
- Modify: `site/index.html` (bloco `<script>`, função do listener `save-btn`)

**Interfaces:**
- Produces: `nomeArquivoPlantao(data: Date): string` — usada por este task e reaproveitada no Task 4 (ativação do salvamento automático).

- [ ] **Step 1: Adicionar a função `nomeArquivoPlantao`**

Em `site/index.html`, logo depois da linha `const AUTOSAVE_KEY = 'hemoam-isbar-utip-rascunho';` (por volta da linha 482), adicionar:

```js
  function nomeArquivoPlantao(data) {
    const dd = String(data.getDate()).padStart(2, '0');
    const mm = String(data.getMonth() + 1).padStart(2, '0');
    const aaaa = data.getFullYear();
    return `plantão (${dd}-${mm}-${aaaa}).json`;
  }
```

- [ ] **Step 2: Usar a nova função no listener do botão "Salvar dados (.json)"**

Substituir (por volta das linhas 554-568):

```js
  document.getElementById('save-btn').addEventListener('click', () => {
    const dados = coletarEstado();
    const blob = new Blob([JSON.stringify(dados, null, 2)], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    const hoje = new Date().toISOString().slice(0, 10);
    const nomeArquivo = `isbar-utip-hemoam-${hoje}.json`;
    const a = document.createElement('a');
    a.href = url;
    a.download = nomeArquivo;
    document.body.appendChild(a);
    a.click();
    a.remove();
    URL.revokeObjectURL(url);
    alert(`Dados salvos em ${nomeArquivo}`);
  });
```

por:

```js
  document.getElementById('save-btn').addEventListener('click', () => {
    const dados = coletarEstado();
    const blob = new Blob([JSON.stringify(dados, null, 2)], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    const nomeArquivo = nomeArquivoPlantao(new Date());
    const a = document.createElement('a');
    a.href = url;
    a.download = nomeArquivo;
    document.body.appendChild(a);
    a.click();
    a.remove();
    URL.revokeObjectURL(url);
    alert(`Dados salvos em ${nomeArquivo}`);
  });
```

- [ ] **Step 3: Verificar manualmente no navegador**

Abrir `file:///Users/felipebarbosa/Desktop/Claude/Hemoam/site/index.html` no navegador:

1. Preencher qualquer campo e clicar em "Salvar dados (.json)".
2. Confirmar que o arquivo baixado se chama `plantão (<dia>-<mês>-<ano com 4 dígitos>).json` com a data de hoje (ex.: `plantão (19-07-2026).json`), e que o alerta mostra esse mesmo nome.
3. Abrir o arquivo baixado (ex.: `cat`) e confirmar que o conteúdo continua um JSON válido no mesmo formato de antes.

Expected: nome do arquivo no novo padrão, conteúdo JSON inalterado.

- [ ] **Step 4: Commit**

```bash
cd "/Users/felipebarbosa/Desktop/Claude/Hemoam/site"
git add index.html
git commit -m "Usa nome de arquivo amigável (plantão dd-mm-aaaa) no botão Salvar dados"
```

---

## Task 2: Helpers de IndexedDB para guardar o handle do arquivo

**Files:**
- Modify: `site/index.html` (bloco `<script>`)

**Interfaces:**
- Produces: `fsaSalvarHandle(handle: FileSystemFileHandle): Promise<void>`, `fsaCarregarHandle(): Promise<FileSystemFileHandle|null>`, `fsaRemoverHandle(): Promise<void>` — usadas pelos Tasks 4, 5 e 6.

- [ ] **Step 1: Adicionar os helpers de IndexedDB**

Logo depois da função `nomeArquivoPlantao` adicionada no Task 1, adicionar:

```js
  const FSA_DB_NAME = 'hemoam-isbar-fsa';
  const FSA_STORE_NAME = 'handles';
  const FSA_HANDLE_KEY = 'plantao-arquivo';

  function fsaAbrirDB() {
    return new Promise((resolve, reject) => {
      const req = indexedDB.open(FSA_DB_NAME, 1);
      req.onupgradeneeded = () => {
        req.result.createObjectStore(FSA_STORE_NAME);
      };
      req.onsuccess = () => resolve(req.result);
      req.onerror = () => reject(req.error);
    });
  }

  async function fsaSalvarHandle(handle) {
    const db = await fsaAbrirDB();
    return new Promise((resolve, reject) => {
      const tx = db.transaction(FSA_STORE_NAME, 'readwrite');
      tx.objectStore(FSA_STORE_NAME).put(handle, FSA_HANDLE_KEY);
      tx.oncomplete = () => resolve();
      tx.onerror = () => reject(tx.error);
    });
  }

  async function fsaCarregarHandle() {
    const db = await fsaAbrirDB();
    return new Promise((resolve, reject) => {
      const tx = db.transaction(FSA_STORE_NAME, 'readonly');
      const req = tx.objectStore(FSA_STORE_NAME).get(FSA_HANDLE_KEY);
      req.onsuccess = () => resolve(req.result || null);
      req.onerror = () => reject(req.error);
    });
  }

  async function fsaRemoverHandle() {
    const db = await fsaAbrirDB();
    return new Promise((resolve, reject) => {
      const tx = db.transaction(FSA_STORE_NAME, 'readwrite');
      tx.objectStore(FSA_STORE_NAME).delete(FSA_HANDLE_KEY);
      tx.oncomplete = () => resolve();
      tx.onerror = () => reject(tx.error);
    });
  }
```

- [ ] **Step 2: Verificar manualmente no console do navegador**

Abrir `file:///Users/felipebarbosa/Desktop/Claude/Hemoam/site/index.html` no Chrome, abrir o DevTools (F12) e no console rodar:

```js
await fsaSalvarHandle({ nome: 'teste' });
await fsaCarregarHandle();
```

Expected: a segunda chamada retorna `{ nome: 'teste' }` (o `put`/`get` do IndexedDB funciona com qualquer valor clonável, então serve pra validar o mecanismo antes de usar com um `FileSystemFileHandle` real nos próximos tasks). Depois rodar:

```js
await fsaRemoverHandle();
await fsaCarregarHandle();
```

Expected: a segunda chamada retorna `null`.

- [ ] **Step 3: Commit**

```bash
cd "/Users/felipebarbosa/Desktop/Claude/Hemoam/site"
git add index.html
git commit -m "Adiciona helpers de IndexedDB para persistir o handle do arquivo compartilhado"
```

---

## Task 3: Botões e status do salvamento automático na toolbar

**Files:**
- Modify: `site/index.html` (toolbar HTML, bloco `<style>`)

**Interfaces:**
- Produces: elementos `#fsa-ativar-btn`, `#fsa-retomar-btn`, `#fsa-desativar-btn`, `#fsa-status`, `#fsa-erro` no DOM — usados pelos Tasks 4, 5 e 6.

- [ ] **Step 1: Adicionar os elementos na toolbar**

Substituir (por volta das linhas 313-318):

```html
  <div class="toolbar-actions">
    <button type="button" id="save-btn">Salvar dados (.json)</button>
    <button type="button" id="load-btn">Carregar dados (.json)</button>
    <input type="file" id="load-input" accept="application/json,.json" style="display:none">
    <button type="button" id="print-btn">Imprimir / Salvar PDF</button>
  </div>
```

por:

```html
  <div class="toolbar-actions">
    <button type="button" id="save-btn">Salvar dados (.json)</button>
    <button type="button" id="load-btn">Carregar dados (.json)</button>
    <input type="file" id="load-input" accept="application/json,.json" style="display:none">
    <button type="button" id="fsa-ativar-btn" style="display:none">Ativar salvamento automático</button>
    <button type="button" id="fsa-retomar-btn" style="display:none">Clique para retomar o salvamento automático</button>
    <button type="button" id="fsa-desativar-btn" style="display:none">Desativar salvamento automático</button>
    <span id="fsa-status" style="display:none"></span>
    <span id="fsa-erro" style="display:none"></span>
    <button type="button" id="print-btn">Imprimir / Salvar PDF</button>
  </div>
```

- [ ] **Step 2: Adicionar CSS para o texto de status e de erro**

Logo depois da regra `.toolbar-actions { ... }` (por volta da linha 71), adicionar:

```css
  #fsa-status {
    font-size: 12px;
    opacity: 0.9;
  }
  #fsa-erro {
    font-size: 12px;
    font-weight: 700;
    color: #ffd166;
  }
```

- [ ] **Step 3: Verificar manualmente no navegador**

Abrir `file:///Users/felipebarbosa/Desktop/Claude/Hemoam/site/index.html`:

1. Confirmar visualmente que nada muda em relação a antes (todos os elementos novos começam com `style="display:none"`, então ficam invisíveis).
2. Abrir o DevTools/console e rodar `document.getElementById('fsa-ativar-btn').style.display = ''` — confirmar que o botão "Ativar salvamento automático" aparece na toolbar com o mesmo estilo dos outros botões. Rodar de novo com `'none'` pra esconder.
3. Abrir a pré-visualização de impressão (Ctrl+P) e confirmar que nenhum desses elementos aparece (a `.toolbar` inteira já é ocultada na impressão).

Expected: todos os itens acima conferem.

- [ ] **Step 4: Commit**

```bash
cd "/Users/felipebarbosa/Desktop/Claude/Hemoam/site"
git add index.html
git commit -m "Adiciona botões e status do salvamento automático na toolbar (ainda sem comportamento)"
```

---

## Task 4: Ativação do salvamento automático

**Files:**
- Modify: `site/index.html` (bloco `<script>`)

**Interfaces:**
- Consumes: `coletarEstado()`, `restaurarEstado(dados)`, `salvarEstadoLocal()` (existentes); `nomeArquivoPlantao(data)` (Task 1); `fsaSalvarHandle(handle)` (Task 2); elementos `#fsa-ativar-btn`, `#fsa-status`, `#fsa-erro` (Task 3).
- Produces: `FSA_SUPORTADO: boolean`, `fsaEstadoAtivo: boolean` (variável), `fsaHandleAtual: FileSystemFileHandle|null` (variável), `fsaMostrarStatus(msg)`, `fsaMostrarErro(msg)`, `fsaAtualizarBotoes(opcoes)`, `fsaEscreverArquivo(handle, dados)`, `fsaLerArquivo(handle)`, `fsaAtivarAutosave(handle)` — usadas pelos Tasks 5 e 6.

- [ ] **Step 1: Adicionar o estado e as funções auxiliares de UI**

Logo depois dos helpers de IndexedDB adicionados no Task 2, adicionar:

```js
  const FSA_SUPORTADO = 'showSaveFilePicker' in window;
  let fsaEstadoAtivo = false;
  let fsaHandleAtual = null;

  function fsaMostrarStatus(mensagem) {
    const el = document.getElementById('fsa-status');
    el.textContent = mensagem;
    el.style.display = mensagem ? '' : 'none';
  }

  function fsaMostrarErro(mensagem) {
    const el = document.getElementById('fsa-erro');
    el.textContent = mensagem;
    el.style.display = mensagem ? '' : 'none';
  }

  function fsaAtualizarBotoes({ mostrarAtivar = false, mostrarRetomar = false, mostrarDesativar = false } = {}) {
    document.getElementById('fsa-ativar-btn').style.display = mostrarAtivar ? '' : 'none';
    document.getElementById('fsa-retomar-btn').style.display = mostrarRetomar ? '' : 'none';
    document.getElementById('fsa-desativar-btn').style.display = mostrarDesativar ? '' : 'none';
  }

  async function fsaEscreverArquivo(handle, dados) {
    const writable = await handle.createWritable();
    await writable.write(JSON.stringify(dados, null, 2));
    await writable.close();
  }

  async function fsaLerArquivo(handle) {
    const file = await handle.getFile();
    const texto = await file.text();
    if (!texto.trim()) return null;
    return JSON.parse(texto);
  }

  function fsaAtivarAutosave(handle) {
    fsaHandleAtual = handle;
    fsaEstadoAtivo = true;
    fsaMostrarErro('');
    fsaMostrarStatus(`Salvamento automático ativo (${handle.name})`);
    fsaAtualizarBotoes({ mostrarDesativar: true });
  }
```

- [ ] **Step 2: Adicionar o listener de ativação**

Logo depois das funções do Step 1, adicionar:

```js
  document.getElementById('fsa-ativar-btn').addEventListener('click', async () => {
    let handle;
    try {
      handle = await window.showSaveFilePicker({
        suggestedName: nomeArquivoPlantao(new Date()),
        types: [{ description: 'Dados ISBAR', accept: { 'application/json': ['.json'] } }]
      });
    } catch (e) {
      return; // usuário cancelou o seletor nativo
    }

    let dadosExistentes = null;
    try {
      dadosExistentes = await fsaLerArquivo(handle);
    } catch (e) {
      fsaMostrarErro('Não foi possível ler o arquivo selecionado. O salvamento automático não foi ativado.');
      return;
    }

    if (dadosExistentes) {
      const substituir = confirm('Isso vai substituir os dados atuais na tela pelos dados do arquivo selecionado. Continuar?');
      if (substituir) {
        restaurarEstado(dadosExistentes);
        salvarEstadoLocal();
      }
    }

    fsaAtivarAutosave(handle);
    try {
      await fsaEscreverArquivo(handle, coletarEstado());
    } catch (e) {
      fsaMostrarErro('Não foi possível salvar no arquivo compartilhado — os dados continuam salvos neste navegador.');
    }
    await fsaSalvarHandle(handle).catch(() => {});
  });
```

- [ ] **Step 3: Mostrar o botão "Ativar" quando o navegador suporta o recurso**

Este step só liga a exibição inicial do botão; a lógica completa de decidir entre "Ativar"/"Retomar"/silencioso fica no Task 6. Por enquanto, logo antes do fechamento `</script>`, junto da chamada `restaurarEstadoLocal();` já existente (linha ~597), adicionar uma linha temporária que o Task 6 vai substituir:

```js
  if (FSA_SUPORTADO) fsaAtualizarBotoes({ mostrarAtivar: true });
```

- [ ] **Step 4: Verificar manualmente no navegador (Chrome)**

Abrir `file:///Users/felipebarbosa/Desktop/Claude/Hemoam/site/index.html` no Chrome:

1. Confirmar que o botão "Ativar salvamento automático" aparece na toolbar (Chrome suporta a API).
2. Preencher o campo "Nome:" do leito 409 com `Teste Ativação`.
3. Clicar em "Ativar salvamento automático" — o seletor nativo do sistema operacional abre. Navegar até uma pasta de teste (pode ser qualquer pasta local para este teste, não precisa ser a pasta de rede) e digitar um nome novo (ex.: aceitar o nome sugerido `plantão (data de hoje).json`) e confirmar criação.
4. Confirmar que **não** aparece nenhum `confirm()` (arquivo era novo/vazio) e que o texto de status muda para `Salvamento automático ativo (plantão (...).json)`.
5. Abrir o arquivo criado fora do navegador (ex.: `cat`) e confirmar que contém `"nome": "Teste Ativação"` dentro do leito 409.
6. Recarregar a página, preencher o campo "Nome:" do leito 410 com `Outro Paciente`, clicar em "Ativar salvamento automático" de novo e desta vez selecionar o **mesmo arquivo** criado no passo 3 (que já tem conteúdo) — confirmar que aparece o `confirm()` de substituição. Aceitar.
7. Confirmar que o campo "Nome:" do leito 409 voltou a mostrar `Teste Ativação` (dado do arquivo) e que o leito 410 ficou vazio (não fazia parte do arquivo carregado).

Expected: todos os itens acima conferem.

- [ ] **Step 5: Commit**

```bash
cd "/Users/felipebarbosa/Desktop/Claude/Hemoam/site"
git add index.html
git commit -m "Implementa ativação do salvamento automático em arquivo (File System Access API)"
```

---

## Task 5: Autosave contínuo no arquivo + tratamento de erro

**Files:**
- Modify: `site/index.html` (bloco `<script>`, listener de `limparLeito`, listener de `load-input`, debounce de autosave)

**Interfaces:**
- Consumes: `fsaEstadoAtivo`, `fsaHandleAtual`, `fsaEscreverArquivo(handle, dados)`, `fsaMostrarErro(msg)` (Task 4); `coletarEstado()`, `salvarEstadoLocal()` (existentes).
- Produces: `fsaSalvarNoArquivo(): Promise<void>`, `salvarEstadoTudo()` — usadas pelo Task 6 e por todos os pontos que hoje chamam `salvarEstadoLocal()`.

- [ ] **Step 1: Adicionar `fsaSalvarNoArquivo` e `salvarEstadoTudo`**

Logo depois de `fsaAtivarAutosave` (adicionada no Task 4), adicionar:

```js
  async function fsaSalvarNoArquivo() {
    if (!fsaEstadoAtivo || !fsaHandleAtual) return;
    try {
      await fsaEscreverArquivo(fsaHandleAtual, coletarEstado());
      fsaMostrarErro('');
    } catch (e) {
      fsaMostrarErro('Não foi possível salvar no arquivo compartilhado — os dados continuam salvos neste navegador.');
    }
  }

  function salvarEstadoTudo() {
    salvarEstadoLocal();
    fsaSalvarNoArquivo();
  }
```

- [ ] **Step 2: Ligar o debounce de autosave a `salvarEstadoTudo`**

Substituir (por volta das linhas 525-529):

```js
  let salvarEstadoLocalTimeout = null;
  function salvarEstadoLocalDebounced() {
    clearTimeout(salvarEstadoLocalTimeout);
    salvarEstadoLocalTimeout = setTimeout(salvarEstadoLocal, 400);
  }
```

por:

```js
  let salvarEstadoLocalTimeout = null;
  function salvarEstadoLocalDebounced() {
    clearTimeout(salvarEstadoLocalTimeout);
    salvarEstadoLocalTimeout = setTimeout(salvarEstadoTudo, 400);
  }
```

- [ ] **Step 3: Atualizar o listener de "Limpar leito" para também salvar no arquivo**

Substituir (por volta das linhas 541-549):

```js
  document.addEventListener('click', (e) => {
    const clearBtn = e.target.closest('.clear-bed-btn');
    if (!clearBtn) return;
    const bed = clearBtn.dataset.clearBed;
    if (confirm(`Limpar todos os dados do leito ${bed}? Essa ação não pode ser desfeita.`)) {
      limparLeito(bed);
      salvarEstadoLocal();
    }
  });
```

por:

```js
  document.addEventListener('click', (e) => {
    const clearBtn = e.target.closest('.clear-bed-btn');
    if (!clearBtn) return;
    const bed = clearBtn.dataset.clearBed;
    if (confirm(`Limpar todos os dados do leito ${bed}? Essa ação não pode ser desfeita.`)) {
      limparLeito(bed);
      salvarEstadoTudo();
    }
  });
```

- [ ] **Step 4: Atualizar o listener de "Carregar dados (.json)" para também salvar no arquivo**

Substituir (por volta das linhas 574-595), a linha:

```js
      restaurarEstado(dados);
      salvarEstadoLocal();
```

por:

```js
      restaurarEstado(dados);
      salvarEstadoTudo();
```

(Essa ocorrência é dentro do listener `load-input`; a chamada equivalente dentro do listener `fsa-ativar-btn`, adicionada no Task 4 Step 2, continua usando `salvarEstadoLocal()` sozinha ali propositalmente — nesse ponto específico o arquivo ainda não foi ativado, então `fsaSalvarNoArquivo()` não teria efeito mesmo.)

- [ ] **Step 5: Verificar manualmente no navegador (Chrome)**

Continuando de onde o Task 4 Step 4 parou (com salvamento automático ativo apontando pra um arquivo de teste):

1. Digitar algo novo em qualquer campo e aguardar ~1s (debounce de 400ms).
2. Abrir o arquivo de teste fora do navegador e confirmar que já reflete a alteração, sem precisar clicar em nada.
3. Clicar em "Limpar leito" em algum leito preenchido e confirmar — abrir o arquivo de novo e confirmar que a limpeza também foi refletida nele.
4. Simular uma falha: depois de ativado, apagar ou mover o arquivo de teste pelo Finder/Explorador enquanto a página continua aberta, digitar algo novo num campo e aguardar ~1s — confirmar que aparece o texto de erro (`#fsa-erro`, "Não foi possível salvar no arquivo compartilhado...") na toolbar, e que o formulário continua funcionando normalmente (não trava, não lança erro não tratado no console além do esperado).

Expected: todos os itens acima conferem.

- [ ] **Step 6: Commit**

```bash
cd "/Users/felipebarbosa/Desktop/Claude/Hemoam/site"
git add index.html
git commit -m "Liga o autosave existente também à escrita no arquivo compartilhado"
```

---

## Task 6: Retomada automática ao carregar a página + desativação

**Files:**
- Modify: `site/index.html` (bloco `<script>`, final do script)

**Interfaces:**
- Consumes: `FSA_SUPORTADO`, `fsaHandleAtual`, `fsaAtivarAutosave(handle)`, `fsaAtualizarBotoes(opcoes)`, `fsaMostrarStatus(msg)`, `fsaMostrarErro(msg)`, `fsaLerArquivo(handle)` (Task 4); `fsaCarregarHandle()`, `fsaRemoverHandle()` (Task 2); `restaurarEstado(dados)` (existente).

- [ ] **Step 1: Remover a linha temporária do Task 4 Step 3**

Remover a linha adicionada no Task 4 Step 3:

```js
  if (FSA_SUPORTADO) fsaAtualizarBotoes({ mostrarAtivar: true });
```

- [ ] **Step 2: Adicionar `fsaTentarRetomarAutomatico` e `fsaDesativarAutosave`**

No lugar de onde a linha do Step 1 foi removida, adicionar:

```js
  function fsaDesativarAutosave() {
    fsaHandleAtual = null;
    fsaEstadoAtivo = false;
    fsaMostrarStatus('');
    fsaMostrarErro('');
    fsaRemoverHandle().catch(() => {});
    fsaAtualizarBotoes({ mostrarAtivar: true });
  }

  async function fsaTentarRetomarAutomatico() {
    if (!FSA_SUPORTADO) return;
    let handle;
    try {
      handle = await fsaCarregarHandle();
    } catch (e) {
      return;
    }
    if (!handle) {
      fsaAtualizarBotoes({ mostrarAtivar: true });
      return;
    }

    let permissao;
    try {
      permissao = await handle.queryPermission({ mode: 'readwrite' });
    } catch (e) {
      await fsaRemoverHandle().catch(() => {});
      fsaAtualizarBotoes({ mostrarAtivar: true });
      return;
    }

    if (permissao === 'granted') {
      try {
        const dados = await fsaLerArquivo(handle);
        if (dados) restaurarEstado(dados);
        fsaAtivarAutosave(handle);
      } catch (e) {
        await fsaRemoverHandle().catch(() => {});
        fsaAtualizarBotoes({ mostrarAtivar: true });
      }
      return;
    }

    fsaHandleAtual = handle;
    fsaAtualizarBotoes({ mostrarRetomar: true });
  }
```

- [ ] **Step 3: Adicionar os listeners de "Retomar" e "Desativar"**

Logo depois da função `fsaTentarRetomarAutomatico`, adicionar:

```js
  document.getElementById('fsa-retomar-btn').addEventListener('click', async () => {
    const handle = fsaHandleAtual;
    if (!handle) return;
    let permissao;
    try {
      permissao = await handle.requestPermission({ mode: 'readwrite' });
    } catch (e) {
      permissao = 'denied';
    }
    if (permissao !== 'granted') {
      fsaMostrarErro('Permissão não concedida — o salvamento automático não foi retomado.');
      return;
    }
    try {
      const dados = await fsaLerArquivo(handle);
      if (dados) restaurarEstado(dados);
      fsaAtivarAutosave(handle);
    } catch (e) {
      fsaMostrarErro('Não foi possível ler o arquivo compartilhado. O salvamento automático não foi retomado.');
    }
  });

  document.getElementById('fsa-desativar-btn').addEventListener('click', () => {
    fsaDesativarAutosave();
  });
```

- [ ] **Step 4: Chamar `fsaTentarRetomarAutomatico` ao carregar a página**

Substituir a última linha do script (linha ~597):

```js
  restaurarEstadoLocal();
```

por:

```js
  restaurarEstadoLocal();
  fsaTentarRetomarAutomatico();
```

- [ ] **Step 5: Verificar manualmente no navegador (Chrome) — retomada no mesmo perfil**

1. Com o salvamento automático já ativado num arquivo de teste (seguindo do Task 5), recarregar a página (F5).
2. Confirmar que os dados aparecem automaticamente, **sem nenhum clique**, e que o status "Salvamento automático ativo (...)" reaparece sozinho (permissão `'granted'` silenciosa).
3. Clicar em "Desativar salvamento automático" — confirmar que o status some e o botão "Ativar" volta a aparecer.
4. Recarregar a página de novo — confirmar que agora aparece só o botão "Ativar" (sem retomar sozinho, já que o handle foi removido do IndexedDB pela desativação).

- [ ] **Step 6: Verificar manualmente no navegador — simular "outro login" e Firefox**

1. Repetir a ativação (Task 4 Step 4) num contexto de navegador limpo (ex.: janela anônima do Chrome, ou outro perfil de usuário do Chrome) apontando para o **mesmo arquivo** de teste já usado antes — confirmar que aparece o botão "Ativar" (não existe handle salvo nesse perfil novo), clicar, selecionar o arquivo existente, aceitar o `confirm()` de substituição, e confirmar que os dados do arquivo aparecem na tela.
2. Abrir o mesmo `index.html` no Firefox (ou forçar `Object.defineProperty(window, 'showSaveFilePicker', { value: undefined })` no console do Chrome antes de recarregar, simulando falta de suporte) — confirmar que nenhum botão do mecanismo novo aparece, e que "Salvar dados (.json)"/"Carregar dados (.json)" continuam funcionando normalmente com o nome de arquivo `plantão (dd-mm-aaaa).json`.

Expected: todos os itens dos Steps 5 e 6 conferem.

- [ ] **Step 7: Commit**

```bash
cd "/Users/felipebarbosa/Desktop/Claude/Hemoam/site"
git add index.html
git commit -m "Adiciona retomada automática do salvamento em arquivo e botão de desativar"
```

---

## Task 7: Verificação final ponta a ponta

**Files:**
- None (apenas verificação manual; se algo falhar, corrigir em `site/index.html` e commitar a correção antes de prosseguir).

- [ ] **Step 1: Rodar o checklist completo da spec (seção "Teste")**

Usando o Chrome, apontando pra uma pasta de teste que simula a pasta pública (`file:///Users/felipebarbosa/Desktop/Claude/Hemoam/site/index.html`), repetir em sequência, numa única sessão:

1. Preencher um leito, clicar "Ativar salvamento automático", criar um arquivo novo → confirmar que grava sem perguntar nada (arquivo era novo) e que o status aparece.
2. Preencher outro campo, aguardar ~1s, abrir o arquivo no disco (fora do navegador) e confirmar que o conteúdo já reflete a alteração.
3. Recarregar a página (mesmo perfil) → confirmar que os dados aparecem automaticamente, sem nenhum clique.
4. Num contexto de navegador diferente (janela anônima) apontando pro mesmo arquivo → confirmar que aparece o botão "Ativar", e que ao clicar e escolher o arquivo já existente (com dados), aparece o `confirm()` de substituição; confirmando, os dados do arquivo aparecem na tela.
5. Mover o arquivo em disco enquanto o autosave está ativo, digitar algo novo → confirmar que aparece o aviso de erro discreto e que o formulário continua funcional (dado continua salvo no `localStorage`).
6. No Firefox (ou com `showSaveFilePicker` forçado como `undefined`): confirmar que o botão "Ativar salvamento automático" não aparece, e que "Salvar/Carregar .json" continuam funcionando, baixando com o nome `plantão (dd-mm-aaaa).json`.
7. Conferir que todos os elementos novos (botões, status, erro) ficam ocultos na pré-visualização de impressão.

Expected: todos os 7 itens conferem, sem erros inesperados no console do navegador.

- [ ] **Step 2: Registrar o resultado**

Se todos os itens do Step 1 passarem, nenhuma ação adicional é necessária — a feature está completa. Se algum item falhar, corrigir o código relevante em `site/index.html`, repetir o Step 1 a partir do item que falhou, e commitar a correção:

```bash
cd "/Users/felipebarbosa/Desktop/Claude/Hemoam/site"
git add index.html
git commit -m "Corrige <descrição específica do problema encontrado na verificação final>"
```
