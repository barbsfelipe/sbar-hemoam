# Ocultar leitos vazios Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** No `site/index.html`, adicionar um botão na toolbar que oculta, na tela e na impressão, os leitos sem nenhum campo preenchido, atualizando sozinho conforme os dados mudam.

**Architecture:** Página estática única, sem build. Reaproveita o seletor `[data-field]` já usado por `limparLeito`/`coletarEstado` para detectar leito vazio. Adiciona uma classe CSS `.bed-hidden` (sem exceção de impressão) aplicada via JS a `.bed-card` e, quando os dois leitos de uma mesma `.page` impressa ficam vazios, também à `.page` inteira (com renumeração de "Página X/Y"). Preferência do filtro persistida em `localStorage`, separada da chave de dados existente.

**Tech Stack:** HTML/CSS/JS vanilla (sem dependências, sem framework, sem build). Sem framework de testes — verificação manual via navegador (Playwright ou Chrome), igual ao padrão já usado nos planos anteriores.

## Global Constraints

- Um leito é vazio quando nenhum de seus elementos `[data-field]` tem valor: nenhum checkbox marcado, nenhum input de texto com conteúdo depois de `trim()`.
- A classe `.bed-hidden { display: none; }` não tem exceção de mídia — precisa esconder tanto na tela quanto na impressão (diferente de `.no-print`).
- Cada `.page` impressa contém até 2 `.bed-card` (`beds[i]`/`beds[i+1]`). Uma `.page` só recebe `bed-hidden` quando **todos** os seus `.bed-card` já têm `bed-hidden`.
- Exceção: a primeira `.page` (a que contém `#page-date`) nunca recebe `bed-hidden` por inteiro, mesmo com os dois leitos vazios.
- O texto de cada `.pageno` das páginas que continuam visíveis é recalculado como `Página <posição>/<total de páginas visíveis>` sempre que a visibilidade muda.
- Preferência do filtro (ligado/desligado) persistida em `localStorage`, chave `hemoam-isbar-ocultar-vazios` — separada de `hemoam-isbar-utip-rascunho` (dados clínicos) e do JSON salvo/carregado.
- `atualizarVisibilidadeLeitosVazios()` roda no clique do botão, em todo `input`/`change` do documento, depois de `limparLeito`, depois de `restaurarEstado`, e uma vez no carregamento inicial da página após os dados serem restaurados.

---

## Task 1: CSS de ocultação

**Files:**
- Modify: `site/index.html` (bloco `<style>`, logo após a regra `.bed-card`)

**Interfaces:**
- Produces: classe CSS `.bed-hidden` — usada por todos os tasks seguintes.

- [ ] **Step 1: Adicionar a regra `.bed-hidden`**

Substituir (por volta das linhas 162-172):

```css
  .bed-card {
    flex: 1;
    display: flex;
    flex-direction: column;
    border: 0.4mm solid var(--line-strong);
    min-height: 0;
    break-inside: avoid;
    page-break-inside: avoid;
  }

  .row {
```

por:

```css
  .bed-card {
    flex: 1;
    display: flex;
    flex-direction: column;
    border: 0.4mm solid var(--line-strong);
    min-height: 0;
    break-inside: avoid;
    page-break-inside: avoid;
  }

  .bed-hidden { display: none; }

  .row {
```

- [ ] **Step 2: Verificar manualmente no navegador**

Abrir `file:///Users/felipebarbosa/Desktop/Claude/Hemoam/site/index.html`, abrir o DevTools/console e rodar:

```js
document.querySelector('.bed-card[data-bed="409"]').classList.add('bed-hidden');
```

Expected: o cartão do leito 409 some da tela imediatamente. Rodar `classList.remove('bed-hidden')` no mesmo elemento para confirmar que ele volta a aparecer.

- [ ] **Step 3: Commit**

```bash
cd "/Users/felipebarbosa/Desktop/Claude/Hemoam/site"
git add index.html
git commit -m "Adiciona classe CSS para ocultar leitos vazios (tela e impressão)"
```

---

## Task 2: Botão na toolbar

**Files:**
- Modify: `site/index.html` (toolbar HTML)

**Interfaces:**
- Produces: elemento `#toggle-vazios-btn` no DOM — usado pelos Tasks 3, 5 e 6.

- [ ] **Step 1: Adicionar o botão na toolbar**

Substituir (por volta das linhas 337-347):

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
    <button type="button" id="toggle-vazios-btn">Ocultar leitos vazios</button>
    <button type="button" id="print-btn">Imprimir / Salvar PDF</button>
  </div>
```

- [ ] **Step 2: Verificar manualmente no navegador**

Abrir `file:///Users/felipebarbosa/Desktop/Claude/Hemoam/site/index.html`. Confirmar que o botão "Ocultar leitos vazios" aparece na toolbar, com o mesmo estilo dos outros botões, entre o status de erro e "Imprimir / Salvar PDF". Clicar nele não faz nada ainda (comportamento vem no Task 3).

- [ ] **Step 3: Commit**

```bash
cd "/Users/felipebarbosa/Desktop/Claude/Hemoam/site"
git add index.html
git commit -m "Adiciona botão Ocultar leitos vazios na toolbar (ainda sem comportamento)"
```

---

## Task 3: Detecção de leito vazio e ocultação por leito

**Files:**
- Modify: `site/index.html` (bloco `<script>`, logo após `limparLeito`)

**Interfaces:**
- Consumes: elemento `#toggle-vazios-btn` (Task 2); classe `.bed-hidden` (Task 1).
- Produces: `leitoEstaVazio(card: Element): boolean`, `atualizarVisibilidadeLeitosVazios(): void`, variável `ocultarVaziosAtivo: boolean` — usados pelos Tasks 4, 5 e 6.

- [ ] **Step 1: Adicionar `leitoEstaVazio` e `atualizarVisibilidadeLeitosVazios` (nível leito)**

Logo depois da função `limparLeito` (por volta da linha 511, antes de `const AUTOSAVE_KEY`), adicionar:

```js
  let ocultarVaziosAtivo = false;

  function leitoEstaVazio(card) {
    let vazio = true;
    card.querySelectorAll('[data-field]').forEach(el => {
      if (el.type === 'checkbox') {
        if (el.checked) vazio = false;
      } else if (el.value.trim() !== '') {
        vazio = false;
      }
    });
    return vazio;
  }

  function atualizarVisibilidadeLeitosVazios() {
    let ocultos = 0;
    document.querySelectorAll('.bed-card').forEach(card => {
      const oculta = ocultarVaziosAtivo && leitoEstaVazio(card);
      card.classList.toggle('bed-hidden', oculta);
      if (oculta) ocultos++;
    });
    const btn = document.getElementById('toggle-vazios-btn');
    btn.textContent = ocultarVaziosAtivo
      ? `Mostrar leitos vazios (${ocultos} ocultos)`
      : 'Ocultar leitos vazios';
  }

  document.getElementById('toggle-vazios-btn').addEventListener('click', () => {
    ocultarVaziosAtivo = !ocultarVaziosAtivo;
    atualizarVisibilidadeLeitosVazios();
  });
```

- [ ] **Step 2: Verificar manualmente no navegador**

Abrir `file:///Users/felipebarbosa/Desktop/Claude/Hemoam/site/index.html` (todos os leitos começam vazios):

1. Clicar em "Ocultar leitos vazios" → os 8 cartões somem, o botão passa a mostrar `Mostrar leitos vazios (8 ocultos)`.
2. Clicar de novo → os 8 cartões reaparecem, botão volta a `Ocultar leitos vazios`.
3. Preencher o campo "Nome:" do leito 409 com `Teste`. Clicar em "Ocultar leitos vazios" → confirmar que o leito 409 **continua visível** e os outros 7 somem, com o botão mostrando `Mostrar leitos vazios (7 ocultos)`.

Expected: todos os itens acima conferem.

- [ ] **Step 3: Commit**

```bash
cd "/Users/felipebarbosa/Desktop/Claude/Hemoam/site"
git add index.html
git commit -m "Adiciona deteccao e ocultacao de leitos vazios por cartao"
```

---

## Task 4: Ocultar página inteira quando os 2 leitos dela estão vazios

**Files:**
- Modify: `site/index.html` (bloco `<script>`, função `atualizarVisibilidadeLeitosVazios`)

**Interfaces:**
- Consumes: `leitoEstaVazio(card)`, `ocultarVaziosAtivo` (Task 3); classe `.bed-hidden` (Task 1); elemento `#page-date` e `.pageno` gerados por `bedCard`/o loop de páginas (já existentes em `index.html`).
- Produces: `atualizarVisibilidadeLeitosVazios()` passa a também ocultar `.page` inteiras e renumerar `.pageno` — comportamento consumido pelos Tasks 5 e 6 (mesma função, assinatura inalterada).

- [ ] **Step 1: Estender `atualizarVisibilidadeLeitosVazios` com a lógica de página**

Substituir a função adicionada no Task 3:

```js
  function atualizarVisibilidadeLeitosVazios() {
    let ocultos = 0;
    document.querySelectorAll('.bed-card').forEach(card => {
      const oculta = ocultarVaziosAtivo && leitoEstaVazio(card);
      card.classList.toggle('bed-hidden', oculta);
      if (oculta) ocultos++;
    });
    const btn = document.getElementById('toggle-vazios-btn');
    btn.textContent = ocultarVaziosAtivo
      ? `Mostrar leitos vazios (${ocultos} ocultos)`
      : 'Ocultar leitos vazios';
  }
```

por:

```js
  function atualizarVisibilidadeLeitosVazios() {
    let ocultos = 0;
    document.querySelectorAll('.bed-card').forEach(card => {
      const oculta = ocultarVaziosAtivo && leitoEstaVazio(card);
      card.classList.toggle('bed-hidden', oculta);
      if (oculta) ocultos++;
    });

    const paginasVisiveis = [];
    document.querySelectorAll('.page').forEach(page => {
      const temDataPlantao = !!page.querySelector('#page-date');
      const cards = page.querySelectorAll('.bed-card');
      const todosOcultos = cards.length > 0 &&
        Array.from(cards).every(card => card.classList.contains('bed-hidden'));
      const oculta = todosOcultos && !temDataPlantao;
      page.classList.toggle('bed-hidden', oculta);
      if (!oculta) paginasVisiveis.push(page);
    });
    paginasVisiveis.forEach((page, indice) => {
      const pagenoEl = page.querySelector('.pageno');
      if (pagenoEl) pagenoEl.textContent = `Página ${indice + 1}/${paginasVisiveis.length}`;
    });

    const btn = document.getElementById('toggle-vazios-btn');
    btn.textContent = ocultarVaziosAtivo
      ? `Mostrar leitos vazios (${ocultos} ocultos)`
      : 'Ocultar leitos vazios';
  }
```

- [ ] **Step 2: Verificar manualmente no navegador**

Abrir `file:///Users/felipebarbosa/Desktop/Claude/Hemoam/site/index.html` (`beds = [409, 410, 411, 412, 413, 414, 415, 416]`, 4 páginas de 2 leitos: 409+410, 411+412, 413+414, 415+416):

1. Sem preencher nada, clicar em "Ocultar leitos vazios" → confirmar que a **primeira página** (409+410) continua visível (mesmo com os dois leitos vazios, por causa do campo de data), mas as páginas 2, 3 e 4 somem inteiras.
2. Clicar de novo para desligar o filtro → as 4 páginas voltam, cada uma mostrando sua numeração original (`Página 1/4` até `Página 4/4`).
3. Preencher o campo "Nome:" do leito 413 e o campo "Nome:" do leito 415, deixando 411, 412, 414 e 416 vazios. Ligar o filtro → confirmar que a página de 411+412 (os dois vazios) some inteira, mas as páginas de 413+414 e 415+416 continuam visíveis (cada uma tem pelo menos um leito com dado), e que a numeração das páginas visíveis fica `Página 1/3`, `Página 2/3`, `Página 3/3` (sem pular número).

Expected: todos os itens acima conferem.

- [ ] **Step 3: Commit**

```bash
cd "/Users/felipebarbosa/Desktop/Claude/Hemoam/site"
git add index.html
git commit -m "Oculta pagina inteira quando os dois leitos dela estao vazios e renumera Pagina X/Y"
```

---

## Task 5: Persistência da preferência do filtro

**Files:**
- Modify: `site/index.html` (bloco `<script>`)

**Interfaces:**
- Consumes: `ocultarVaziosAtivo`, `atualizarVisibilidadeLeitosVazios()` (Tasks 3/4).
- Produces: `salvarPreferenciaOcultarVazios(): void`, `restaurarPreferenciaOcultarVazios(): void` — usadas pelo Task 6.

- [ ] **Step 1: Adicionar a chave de storage e os helpers de leitura/escrita**

Substituir (por volta da linha 513):

```js
  const AUTOSAVE_KEY = 'hemoam-isbar-utip-rascunho';
```

por:

```js
  const AUTOSAVE_KEY = 'hemoam-isbar-utip-rascunho';

  const OCULTAR_VAZIOS_KEY = 'hemoam-isbar-ocultar-vazios';

  function salvarPreferenciaOcultarVazios() {
    try {
      localStorage.setItem(OCULTAR_VAZIOS_KEY, ocultarVaziosAtivo ? '1' : '0');
    } catch (e) { /* localStorage indisponível ou cheio: ignora */ }
  }

  function restaurarPreferenciaOcultarVazios() {
    try {
      ocultarVaziosAtivo = localStorage.getItem(OCULTAR_VAZIOS_KEY) === '1';
    } catch (e) { ocultarVaziosAtivo = false; }
  }

  restaurarPreferenciaOcultarVazios();
  atualizarVisibilidadeLeitosVazios();
```

(As duas últimas linhas são temporárias, só para dar pra testar a persistência já neste task — o Task 6 remove essas duas linhas daqui e coloca a chamada equivalente no lugar certo, depois dos dados serem restaurados no carregamento da página.)

- [ ] **Step 2: Salvar a preferência ao clicar no botão**

Substituir o listener adicionado no Task 3:

```js
  document.getElementById('toggle-vazios-btn').addEventListener('click', () => {
    ocultarVaziosAtivo = !ocultarVaziosAtivo;
    atualizarVisibilidadeLeitosVazios();
  });
```

por:

```js
  document.getElementById('toggle-vazios-btn').addEventListener('click', () => {
    ocultarVaziosAtivo = !ocultarVaziosAtivo;
    salvarPreferenciaOcultarVazios();
    atualizarVisibilidadeLeitosVazios();
  });
```

- [ ] **Step 3: Verificar manualmente no navegador**

Usar uma aba anônima/privada (para começar com `localStorage` limpo) e abrir `file:///Users/felipebarbosa/Desktop/Claude/Hemoam/site/index.html`:

1. Clicar em "Ocultar leitos vazios" (fica ligado, todos os 8 leitos vazios somem).
2. Recarregar a página (F5) → confirmar que o botão já aparece como `Mostrar leitos vazios (8 ocultos)` assim que a página carrega, sem precisar clicar de novo.
3. Clicar para desligar o filtro → recarregar de novo → confirmar que volta a aparecer `Ocultar leitos vazios` (desligado), preferência também persistida.

Expected: os 3 itens conferem. (Não testar ainda com dados preenchidos entre recarregamentos — a leitura de dados salvos só é sincronizada com este filtro no Task 6.)

- [ ] **Step 4: Commit**

```bash
cd "/Users/felipebarbosa/Desktop/Claude/Hemoam/site"
git add index.html
git commit -m "Persiste a preferencia do filtro de leitos vazios em localStorage"
```

---

## Task 6: Atualização automática em todos os pontos de mudança de dados

**Files:**
- Modify: `site/index.html` (bloco `<script>`: listener global de `input`/`change`, listener de `.clear-bed-btn`, função `restaurarEstado`, IIFE final)

**Interfaces:**
- Consumes: `atualizarVisibilidadeLeitosVazios()` (Task 4), `restaurarPreferenciaOcultarVazios()` (Task 5), `limparLeito`, `restaurarEstado`, `restaurarEstadoLocal`, `fsaTentarRetomarAutomatico`, `preencherDataPadrao`, `salvarEstadoTudo` (já existentes em `index.html`).

- [ ] **Step 1: Remover as duas linhas temporárias do Task 5**

Remover, do trecho adicionado no Task 5 Step 1, as duas últimas linhas:

```js
  restaurarPreferenciaOcultarVazios();
  atualizarVisibilidadeLeitosVazios();
```

(A função `restaurarPreferenciaOcultarVazios` continua definida ali — só a chamada imediata sai daqui e vai para o Step 5 deste task.)

- [ ] **Step 2: Atualizar visibilidade a cada `input`/`change` no documento**

Substituir (por volta das linhas 756-757):

```js
  document.addEventListener('input', salvarEstadoLocalDebounced);
  document.addEventListener('change', salvarEstadoLocalDebounced);
```

por:

```js
  document.addEventListener('input', salvarEstadoLocalDebounced);
  document.addEventListener('input', atualizarVisibilidadeLeitosVazios);
  document.addEventListener('change', salvarEstadoLocalDebounced);
  document.addEventListener('change', atualizarVisibilidadeLeitosVazios);
```

- [ ] **Step 3: Atualizar visibilidade depois de "Limpar leito"**

Substituir (por volta das linhas 746-754):

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

por:

```js
  document.addEventListener('click', (e) => {
    const clearBtn = e.target.closest('.clear-bed-btn');
    if (!clearBtn) return;
    const bed = clearBtn.dataset.clearBed;
    if (confirm(`Limpar todos os dados do leito ${bed}? Essa ação não pode ser desfeita.`)) {
      limparLeito(bed);
      atualizarVisibilidadeLeitosVazios();
      salvarEstadoTudo();
    }
  });
```

- [ ] **Step 4: Atualizar visibilidade depois de restaurar dados (autosave, arquivo compartilhado ou "Carregar dados")**

Substituir (por volta das linhas 685-703):

```js
  function restaurarEstado(dados) {
    if (!dados || typeof dados !== 'object') return;
    const pageDateEl = document.getElementById('page-date');
    if (pageDateEl && typeof dados.pageDate === 'string') pageDateEl.value = dados.pageDate;
    if (dados.beds) {
      document.querySelectorAll('.bed-card').forEach(card => {
        const bed = card.dataset.bed;
        const campos = dados.beds[bed];
        if (!campos) return;
        card.querySelectorAll('[data-field]').forEach(el => {
          if (!(el.dataset.field in campos)) return;
          const valor = campos[el.dataset.field];
          if (el.type === 'checkbox') el.checked = !!valor;
          else el.value = valor;
        });
      });
    }
    sincronizarDetalhes();
  }
```

por:

```js
  function restaurarEstado(dados) {
    if (!dados || typeof dados !== 'object') return;
    const pageDateEl = document.getElementById('page-date');
    if (pageDateEl && typeof dados.pageDate === 'string') pageDateEl.value = dados.pageDate;
    if (dados.beds) {
      document.querySelectorAll('.bed-card').forEach(card => {
        const bed = card.dataset.bed;
        const campos = dados.beds[bed];
        if (!campos) return;
        card.querySelectorAll('[data-field]').forEach(el => {
          if (!(el.dataset.field in campos)) return;
          const valor = campos[el.dataset.field];
          if (el.type === 'checkbox') el.checked = !!valor;
          else el.value = valor;
        });
      });
    }
    sincronizarDetalhes();
    atualizarVisibilidadeLeitosVazios();
  }
```

- [ ] **Step 5: Atualizar visibilidade uma vez no carregamento inicial, depois dos dados restaurados**

Substituir (por volta das linhas 908-912):

```js
  (async () => {
    restaurarEstadoLocal();
    await fsaTentarRetomarAutomatico();
    if (preencherDataPadrao()) salvarEstadoTudo();
  })();
```

por:

```js
  (async () => {
    restaurarPreferenciaOcultarVazios();
    restaurarEstadoLocal();
    await fsaTentarRetomarAutomatico();
    if (preencherDataPadrao()) salvarEstadoTudo();
    atualizarVisibilidadeLeitosVazios();
  })();
```

- [ ] **Step 6: Verificar manualmente no navegador**

Usar uma aba anônima/privada e abrir `file:///Users/felipebarbosa/Desktop/Claude/Hemoam/site/index.html`:

1. Ligar o filtro "Ocultar leitos vazios" (todos os 8 leitos somem, exceto a primeira página que continua visível por causa da data).
2. Desligar o filtro, preencher o campo "Nome:" do leito 411 e do leito 412, ligar o filtro de novo → confirmar que a página 411+412 continua visível (tem dado agora) e a contagem de ocultos reflete os outros 6 leitos.
3. Com o filtro ligado, apagar o texto digitado no leito 411 e no leito 412 (deixando os dois vazios de novo) → confirmar que a página some **imediatamente**, sem precisar clicar em nada.
4. Com o filtro ligado, clicar em "Limpar leito" num leito que tinha dado (ex.: 413, depois de preenchê-lo) → confirmar que ele some assim que a limpeza é confirmada.
5. Com o filtro ligado, usar "Salvar dados (.json)" para baixar um arquivo com pelo menos um leito preenchido (ex.: 415), depois "Limpar leito" nesse mesmo leito para ele ficar vazio e oculto, e então "Carregar dados (.json)" apontando para o arquivo baixado → confirmar que o leito 415 reaparece automaticamente (tinha dado no arquivo carregado).
6. Recarregar a página com o filtro ligado e dados reais salvos (alguns leitos preenchidos, outros vazios) → confirmar que, assim que a página termina de carregar, os leitos vazios já aparecem ocultos corretamente (sem precisar digitar nada para "corrigir" o estado inicial) — isso valida que a ordem de restauração (preferência → dados → atualizar visibilidade) ficou certa.

Expected: todos os itens acima conferem, sem erros no console do navegador.

- [ ] **Step 7: Commit**

```bash
cd "/Users/felipebarbosa/Desktop/Claude/Hemoam/site"
git add index.html
git commit -m "Atualiza visibilidade de leitos vazios automaticamente conforme os dados mudam"
```

---

## Task 7: Verificação final ponta a ponta (checklist da spec)

**Files:**
- None (apenas verificação manual; se algo falhar, corrigir em `site/index.html` e commitar a correção antes de prosseguir).

- [ ] **Step 1: Rodar o checklist completo da spec (seção "Teste")**

Numa aba anônima/privada do Chrome, apontando para `file:///Users/felipebarbosa/Desktop/Claude/Hemoam/site/index.html`, repetir em sequência, numa única sessão:

1. Com todos os 8 leitos vazios, clicar em "Ocultar leitos vazios" → todos os cartões preenchíveis somem, exceto a primeira página (fica visível por causa do campo de data), botão mostra a contagem de ocultos.
2. Clicar em "Mostrar leitos vazios", preencher um campo de texto em um leito, religar o filtro → confirmar que esse leito não some.
3. Com o filtro ligado e um leito preenchido, clicar em "Limpar leito" nesse leito → o cartão some imediatamente e a contagem do botão sobe.
4. Com o filtro ligado, usar "Carregar dados (.json)" apontando para um arquivo com dados em leitos hoje vazios/ocultos → confirmar que esses leitos reaparecem automaticamente.
5. Com o filtro ligado e ao menos um leito oculto (mas não os dois de uma mesma página), abrir a pré-visualização de impressão (Ctrl+P) → confirmar que a página correspondente ainda aparece, com o leito visível ocupando a folha inteira.
6. Deixar os dois leitos de uma mesma página (ex.: 411 e 412) vazios com o filtro ligado → confirmar que essa página some inteira da pré-visualização de impressão, e que a numeração `Página X/Y` das páginas restantes fica contígua.
7. Deixar os dois leitos da primeira página (409 e 410) vazios com o filtro ligado → confirmar que a primeira página continua aparecendo, com o campo de data visível.
8. Recarregar a página com o filtro ligado → confirmar que ele continua ligado e que a contagem exibida bate com os leitos realmente vazios no momento da carga.
9. Desligar o filtro → todos os 8 leitos e as 4 páginas voltam a aparecer, com a numeração original `Página X/4`.

Expected: todos os 9 itens conferem, sem erros inesperados no console do navegador.

- [ ] **Step 2: Registrar o resultado**

Se todos os itens do Step 1 passarem, nenhuma ação adicional é necessária — a feature está completa. Se algum item falhar, corrigir o código relevante em `site/index.html`, repetir o Step 1 a partir do item que falhou, e commitar a correção:

```bash
cd "/Users/felipebarbosa/Desktop/Claude/Hemoam/site"
git add index.html
git commit -m "Corrige <descrição específica do problema encontrado na verificação final>"
```
