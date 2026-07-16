# Continuidade ISBAR (autosave + salvar/carregar arquivo + limpar leito) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Permitir que os dados preenchidos no formulário ISBAR (`site/index.html`) sejam salvos e reaproveitados no dia seguinte (inclusive em outro aparelho), e permitir limpar individualmente os dados de um leito quando o paciente recebe alta.

**Architecture:** Página estática única (`site/index.html`), sem backend nem build step. Cada campo editável de cada leito ganha um atributo `data-field` (chave curta, única dentro do card) e cada `.bed-card` ganha `data-bed`. Duas camadas de persistência client-side: (1) autosave silencioso em `localStorage`, restaurado ao carregar a página; (2) botões "Salvar dados (.json)" / "Carregar dados (.json)" na toolbar, que baixam/leem um arquivo JSON com o estado completo — é isso que resolve a continuidade entre aparelhos diferentes. Um botão "Limpar leito" por card zera apenas os campos daquele `.bed-card`.

**Tech Stack:** HTML/CSS/JS vanilla (sem dependências, sem framework, sem build). Não há framework de testes no projeto — a verificação é manual, via navegador (Playwright), abrindo o arquivo diretamente (`file://.../site/index.html`).

## Global Constraints

- Nenhum dado sai do dispositivo do usuário: sem chamadas de rede, sem serviços externos (ver spec, seção "Contexto" — decisão explícita para evitar riscos de LGPD com dados de saúde).
- Os blocos "Intercorrências / evolução — TARDE/NOITE" continuam sem `<input>`/`<textarea>` (preenchimento manual a caneta) — não fazem parte do estado digital (spec, "Não-objetivos").
- Botões novos (toolbar e "Limpar leito") devem ficar ocultos na impressão (`@media print`).
- Ações destrutivas (Carregar substituindo a tela, Limpar leito) sempre pedem `confirm()` antes de executar (spec, "Tratamento de erros").
- Falhas de `localStorage` (indisponível/cheio) devem ser ignoradas silenciosamente via `try/catch`, sem quebrar o formulário.
- Chave do autosave: `hemoam-isbar-utip-rascunho`.
- Nome do arquivo salvo: `isbar-utip-hemoam-AAAA-MM-DD.json` (data de hoje).

---

## Task 1: Atributos de dados por leito + botão "Limpar leito"

**Files:**
- Modify: `site/index.html` (função `chkPair`, função `bedCard`, bloco `<style>`, bloco `<script>`)

**Interfaces:**
- Produces: convenção de marcação `[data-bed="<numero>"]` em cada `.bed-card`, `[data-field="<chave>"]` em cada input/checkbox editável dentro do card, classe `.clear-bed-btn` com `data-clear-bed="<numero>"`, classe utilitária `.no-print`.

- [ ] **Step 1: Adicionar regra CSS `.no-print` e estilo do botão de limpar leito**

Em `site/index.html`, dentro do bloco `@media print { ... }` (linhas 267-282), adicionar a regra `.no-print` junto de `.toolbar`:

```css
  @media print {
    .toolbar,
    .no-print { display: none; }
    html, body { background: #fff; }
    .sheet-wrap { padding: 0; gap: 0; }
    .page {
      box-shadow: none;
      width: 100%;
      aspect-ratio: auto;
      break-after: page;
      page-break-after: always;
    }
    .page:last-child {
      break-after: auto;
      page-break-after: auto;
    }
  }
```

Logo depois da regra `.header-strip input[type="text"] { ... }` (linha 209), adicionar o estilo do botão:

```css
  .header-strip .clear-bed-btn {
    background: transparent;
    color: #fff;
    border: 0.25mm solid rgba(255,255,255,0.6);
    border-radius: 2px;
    padding: 0.3mm 2mm;
    font-size: 7.5px;
    font-weight: 700;
    cursor: pointer;
    white-space: nowrap;
  }
  .header-strip .clear-bed-btn:hover { background: rgba(255,255,255,0.15); }
```

- [ ] **Step 2: Reescrever `chkPair` para usar `data-field` sem prefixo de leito**

Substituir (linhas 302-307):

```js
  function chkPair(nameBase) {
    return `
      <label class="chk"><input type="checkbox" name="${nameBase}_n"> N</label>
      <label class="chk"><input type="checkbox" name="${nameBase}_s"> S</label>
    `;
  }
```

por:

```js
  function chkPair(field) {
    return `
      <label class="chk"><input type="checkbox" data-field="${field}_nao"> N</label>
      <label class="chk"><input type="checkbox" data-field="${field}_sim"> S</label>
    `;
  }
```

- [ ] **Step 3: Reescrever `bedCard` com `data-bed`, `data-field` em cada campo e o botão "Limpar leito"**

Substituir toda a função `bedCard` (linhas 309-417) por:

```js
  function bedCard(bed) {
    return `
    <div class="bed-card" data-bed="${bed}">
      <div class="row header-strip">
        <div class="cell"><span class="label">LEITO ${bed}</span></div>
        <div class="cell wide"><span class="label">Data internação:</span><input type="text" data-field="dataInternacao"></div>
        <div class="cell wide"><span class="label">Idade/DN:</span><input type="text" data-field="idadeDN"></div>
        <div class="cell" style="flex: 0 0 auto;"><button type="button" class="clear-bed-btn no-print" data-clear-bed="${bed}">Limpar leito</button></div>
      </div>

      <div class="row">
        <div class="cell xwide"><span class="label">Nome:</span><input type="text" data-field="nome"></div>
        <div class="cell xwide"><span class="label">Nome da mãe:</span><input type="text" data-field="nomeMae"></div>
      </div>

      <div class="row alert-row">
        <div class="cell"><span class="label">Alergias:</span>${chkPair('alergias')}</div>
        <div class="cell xwide">
          <span class="label">Dispositivos:</span>
          <label class="chk"><input type="checkbox" data-field="disp_cvc"> CVC</label>
          <input type="text" data-field="disp_cvc_val" style="max-width:10mm">
          <label class="chk"><input type="checkbox" data-field="disp_tot"> TOT</label>
          <input type="text" data-field="disp_tot_val" style="max-width:10mm">
          <label class="chk"><input type="checkbox" data-field="disp_svd"> SVD</label>
          <label class="chk"><input type="checkbox" data-field="disp_sne"> SNE</label>
          <label class="chk"><input type="checkbox" data-field="disp_sng"> SNG</label>
          <label class="chk"><input type="checkbox" data-field="disp_shiley"> Shiley</label>
          <label class="chk"><input type="checkbox" data-field="disp_tenkhoff"> Tenkhoff</label>
          <label class="chk"><input type="checkbox" data-field="disp_outro"> Outro:</label>
          <input type="text" data-field="disp_outro_val" style="max-width:16mm">
        </div>
      </div>

      <div class="row">
        <div class="cell xwide"><span class="label section">S</span>Diagnósticos:<input type="text" data-field="diagnosticos"></div>
      </div>

      <div class="row">
        <div class="cell"><span class="label section">B</span>Antibióticos:${chkPair('atb')}</div>
        <div class="cell">Culturas:${chkPair('cult')}</div>
        <div class="cell">QT:${chkPair('qt')}</div>
        <div class="cell">Diálise:${chkPair('dial')}</div>
        <div class="cell">Sedação/Analgesia:${chkPair('sed')}</div>
        <div class="cell">Corr. eletrólitos:${chkPair('elet')}</div>
      </div>

      <div class="row">
        <div class="cell"><span class="label section">A</span>Dieta:${chkPair('dieta')}</div>
        <div class="cell">O2:${chkPair('o2')}</div>
        <div class="cell">DVA:${chkPair('dva')}</div>
        <div class="cell wide">
          Transfusão:
          <label class="chk"><input type="checkbox" data-field="transf_ch"> CH</label>
          <label class="chk"><input type="checkbox" data-field="transf_cp"> CP</label>
          <label class="chk"><input type="checkbox" data-field="transf_pfc"> PFC</label>
          <label class="chk"><input type="checkbox" data-field="transf_crio"> CRIO</label>
        </div>
      </div>

      <div class="row vent-row">
        <div class="cell xwide">
          <span class="label">Suporte ventilatório:</span>
          <label class="chk"><input type="checkbox" data-field="vent_arambiente"> Ar ambiente</label>
          <label class="chk"><input type="checkbox" data-field="vent_o2"> O2</label>
          <label class="chk"><input type="checkbox" data-field="vent_vni"> VNI</label>
          <label class="chk"><input type="checkbox" data-field="vent_vpm"> VPM</label>
          <span class="vent-field">Modo:<input type="text" data-field="vent_modo"></span>
          <span class="vent-field">FiO2:<input type="text" data-field="vent_fio2"></span>
          <span class="vent-field">Pinsp/Epap:<input type="text" data-field="vent_pinsp_epap"></span>
          <span class="vent-field">Peep/Ipap:<input type="text" data-field="vent_peep_ipap"></span>
          <span class="vent-field">Tins:<input type="text" data-field="vent_tins"></span>
          <span class="vent-field">FR:<input type="text" data-field="vent_fr"></span>
        </div>
      </div>

      <div class="row">
        <div class="cell xwide"><span class="label section">R</span>Programação/Condutas:<input type="text" data-field="condutas"></div>
      </div>
      <div class="row">
        <div class="cell xwide r-cont"><input type="text" data-field="condutas2"></div>
      </div>
      <div class="row">
        <div class="cell wide">Exames pendentes:<input type="text" data-field="examesPendentes"></div>
        <div class="cell wide">Reavaliar:<input type="text" data-field="reavaliar"></div>
      </div>
      <div class="row">
        <div class="cell xwide">Pendências:<input type="text" data-field="pendencias"></div>
      </div>
      <div class="row">
        <div class="cell wide">Alta:${chkPair('alta')}<span class="vent-field">Leito:<input type="text" data-field="altaLeito" style="max-width:14mm"></span></div>
      </div>

      <div class="written-block">
        <div class="written-heading">
          <div>Intercorrências / evolução — TARDE</div>
          <div>Intercorrências / evolução — NOITE</div>
        </div>
        <div class="written-cols">
          <div class="written-col"></div>
          <div class="written-col"></div>
        </div>
      </div>

      <div class="sign-row">
        <span class="label">Assinatura/CRM do plantonista:</span>
        <input type="text" data-field="assinatura">
      </div>
    </div>
    `;
  }
```

- [ ] **Step 4: Adicionar `id="page-date"` ao campo de data da página 1**

Na montagem das páginas (por volta da linha 432), alterar:

```js
          <div class="page-date">
            <span class="label">Data:</span><input type="text" style="max-width:22mm">
          </div>` : '<div class="page-date"></div>'}
```

para:

```js
          <div class="page-date">
            <span class="label">Data:</span><input type="text" id="page-date" style="max-width:22mm">
          </div>` : '<div class="page-date"></div>'}
```

- [ ] **Step 5: Adicionar função `limparLeito` e o listener de clique do botão**

Logo após a linha `document.getElementById('print-btn').addEventListener('click', () => window.print());` (linha 446), adicionar:

```js
  function limparLeito(bed) {
    const card = document.querySelector(`.bed-card[data-bed="${bed}"]`);
    if (!card) return;
    card.querySelectorAll('[data-field]').forEach(el => {
      if (el.type === 'checkbox') el.checked = false;
      else el.value = '';
    });
  }

  document.addEventListener('click', (e) => {
    const clearBtn = e.target.closest('.clear-bed-btn');
    if (!clearBtn) return;
    const bed = clearBtn.dataset.clearBed;
    if (confirm(`Limpar todos os dados do leito ${bed}? Essa ação não pode ser desfeita.`)) {
      limparLeito(bed);
    }
  });
```

- [ ] **Step 6: Verificar manualmente no navegador**

Abrir `site/index.html` diretamente no navegador (ex: via Playwright, navegando para `file:///Users/felipebarbosa/Desktop/Claude/Hemoam/site/index.html`):

1. No card "LEITO 409", preencher o campo "Nome:" com `Teste Paciente` e marcar o checkbox "S" de Alergias.
2. No card "LEITO 410", preencher o campo "Nome:" com `Outro Paciente`.
3. Clicar em "Limpar leito" no card 409 → deve aparecer um `confirm()`; aceitar.
4. Confirmar que o campo "Nome:" do leito 409 ficou vazio e o checkbox de alergias desmarcou.
5. Confirmar que o card 410 continua com `Outro Paciente` no campo Nome, inalterado.
6. Abrir a pré-visualização de impressão (`window.print()` / Ctrl+P) e confirmar que o botão "Limpar leito" não aparece.

Expected: todos os itens acima conferem.

- [ ] **Step 7: Commit**

```bash
cd "/Users/felipebarbosa/Desktop/Claude/Hemoam/site"
git add index.html
git commit -m "Adiciona data-field/data-bed e botão Limpar leito por card"
```

---

## Task 2: Coleta/restauração de estado + autosave em localStorage

**Files:**
- Modify: `site/index.html` (bloco `<script>`)

**Interfaces:**
- Consumes: marcação `[data-bed]`/`[data-field]` produzida na Task 1; elemento `#page-date` produzido na Task 1, Step 4.
- Produces: `coletarEstado(): { pageDate: string, beds: { [bed: string]: { [field: string]: string|boolean } } }`, `restaurarEstado(dados: object): void`, `salvarEstadoLocal(): void`, `restaurarEstadoLocal(): void`. Essas quatro funções são usadas pela Task 3.

- [ ] **Step 1: Adicionar as funções de estado e autosave**

Logo após a função `limparLeito` e antes do listener `document.addEventListener('click', ...)` adicionado na Task 1 (ou logo depois dele — a ordem entre essas duas não importa), adicionar:

```js
  const AUTOSAVE_KEY = 'hemoam-isbar-utip-rascunho';

  function coletarEstado() {
    const beds = {};
    document.querySelectorAll('.bed-card').forEach(card => {
      const bed = card.dataset.bed;
      const campos = {};
      card.querySelectorAll('[data-field]').forEach(el => {
        campos[el.dataset.field] = el.type === 'checkbox' ? el.checked : el.value;
      });
      beds[bed] = campos;
    });
    const pageDateEl = document.getElementById('page-date');
    return {
      pageDate: pageDateEl ? pageDateEl.value : '',
      beds
    };
  }

  function restaurarEstado(dados) {
    if (!dados || typeof dados !== 'object') return;
    const pageDateEl = document.getElementById('page-date');
    if (pageDateEl && typeof dados.pageDate === 'string') pageDateEl.value = dados.pageDate;
    if (!dados.beds) return;
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

  function salvarEstadoLocal() {
    try {
      localStorage.setItem(AUTOSAVE_KEY, JSON.stringify(coletarEstado()));
    } catch (e) { /* localStorage indisponível ou cheio: ignora */ }
  }

  let salvarEstadoLocalTimeout = null;
  function salvarEstadoLocalDebounced() {
    clearTimeout(salvarEstadoLocalTimeout);
    salvarEstadoLocalTimeout = setTimeout(salvarEstadoLocal, 400);
  }

  function restaurarEstadoLocal() {
    let dados;
    try {
      const raw = localStorage.getItem(AUTOSAVE_KEY);
      if (!raw) return;
      dados = JSON.parse(raw);
    } catch (e) { return; }
    restaurarEstado(dados);
  }
```

- [ ] **Step 2: Atualizar `limparLeito` para acionar o autosave após limpar**

Alterar o listener de clique adicionado na Task 1 para chamar `salvarEstadoLocal()` após `limparLeito(bed)`:

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

- [ ] **Step 3: Ligar autosave aos eventos de input/change e restaurar ao carregar**

No final do `<script>`, logo antes da tag `</script>` de fechamento, adicionar:

```js
  document.addEventListener('input', salvarEstadoLocalDebounced);
  document.addEventListener('change', salvarEstadoLocalDebounced);

  restaurarEstadoLocal();
```

- [ ] **Step 4: Verificar manualmente no navegador**

Usando Playwright (ou o navegador) em `file:///Users/felipebarbosa/Desktop/Claude/Hemoam/site/index.html`:

1. Preencher o campo "Nome:" do leito 411 com `Paciente Autosave` e aguardar ~1s (debounce de 400ms).
2. Recarregar a página (F5).
3. Confirmar que o campo "Nome:" do leito 411 ainda mostra `Paciente Autosave` (restaurado do `localStorage`).
4. Abrir o DevTools/console e rodar `localStorage.getItem('hemoam-isbar-utip-rascunho')` — confirmar que retorna um JSON contendo `"nome":"Paciente Autosave"` dentro de `beds["411"]`.
5. Limpar o leito 411 (botão "Limpar leito", confirmar o `confirm()`), recarregar a página novamente, e confirmar que o campo "Nome:" permanece vazio (ou seja, a limpeza também foi persistida).

Expected: todos os itens acima conferem.

- [ ] **Step 5: Commit**

```bash
cd "/Users/felipebarbosa/Desktop/Claude/Hemoam/site"
git add index.html
git commit -m "Adiciona coleta/restauração de estado e autosave em localStorage"
```

---

## Task 3: Botões "Salvar dados (.json)" e "Carregar dados (.json)"

**Files:**
- Modify: `site/index.html` (toolbar HTML, bloco `<style>`, bloco `<script>`)

**Interfaces:**
- Consumes: `coletarEstado()` e `restaurarEstado(dados)` da Task 2; `salvarEstadoLocal()` da Task 2.

- [ ] **Step 1: Atualizar a toolbar com os novos botões e o input de arquivo oculto**

Substituir o bloco da toolbar (linhas 292-295):

```html
<div class="toolbar no-print">
  <div class="note">Protótipo — preencher os campos digitados de manhã, imprimir e completar a lápis/caneta à tarde e à noite.</div>
  <button id="print-btn">Imprimir / Salvar PDF</button>
</div>
```

por:

```html
<div class="toolbar no-print">
  <div class="note">Protótipo — preencher os campos digitados de manhã, imprimir e completar a lápis/caneta à tarde e à noite.</div>
  <div class="toolbar-actions">
    <button type="button" id="save-btn">Salvar dados (.json)</button>
    <button type="button" id="load-btn">Carregar dados (.json)</button>
    <input type="file" id="load-input" accept="application/json,.json" style="display:none">
    <button type="button" id="print-btn">Imprimir / Salvar PDF</button>
  </div>
</div>
```

- [ ] **Step 2: Adicionar CSS para agrupar os botões da toolbar**

Logo após a regra `.toolbar .note { opacity: 0.9; }` (linha 66), adicionar:

```css
  .toolbar-actions {
    display: flex;
    align-items: center;
    gap: 8px;
  }
```

- [ ] **Step 3: Adicionar os listeners de Salvar e Carregar**

Logo após a linha `restaurarEstadoLocal();` adicionada na Task 2, Step 3 (antes do fechamento de `</script>`), adicionar:

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

  document.getElementById('load-btn').addEventListener('click', () => {
    document.getElementById('load-input').click();
  });

  document.getElementById('load-input').addEventListener('change', (e) => {
    const file = e.target.files[0];
    e.target.value = '';
    if (!file) return;
    if (!confirm('Isso vai substituir os dados atuais na tela pelos dados do arquivo carregado. Continuar?')) return;
    const reader = new FileReader();
    reader.onload = () => {
      let dados;
      try {
        dados = JSON.parse(reader.result);
      } catch (err) {
        alert('Arquivo inválido: não foi possível ler os dados. Nada foi alterado.');
        return;
      }
      restaurarEstado(dados);
      salvarEstadoLocal();
    };
    reader.onerror = () => {
      alert('Não foi possível ler o arquivo selecionado. Nada foi alterado.');
    };
    reader.readAsText(file);
  });
```

- [ ] **Step 4: Verificar manualmente no navegador**

Usando Playwright em `file:///Users/felipebarbosa/Desktop/Claude/Hemoam/site/index.html`:

1. Preencher o campo "Nome:" do leito 412 com `Paciente Arquivo`.
2. Clicar em "Salvar dados (.json)" — confirmar que dispara o download de um arquivo chamado `isbar-utip-hemoam-<data de hoje>.json` e que aparece o alerta de confirmação.
3. Abrir o arquivo baixado (ex: com `cat` no terminal) e confirmar que contém `"nome": "Paciente Arquivo"` dentro de `"412"`.
4. Limpar o leito 412 ("Limpar leito" + confirmar).
5. Clicar em "Carregar dados (.json)", selecionar o arquivo baixado no passo 2, aceitar o `confirm()`.
6. Confirmar que o campo "Nome:" do leito 412 voltou a mostrar `Paciente Arquivo`.
7. Criar um arquivo de texto inválido (ex: `echo "não é json" > /tmp/invalido.json`) e tentar carregá-lo — confirmar que aparece o alerta de erro ("Arquivo inválido...") e que os dados na tela não mudam.
8. Abrir a pré-visualização de impressão e confirmar que os botões "Salvar dados (.json)" e "Carregar dados (.json)" não aparecem (a `.toolbar` inteira já é ocultada).

Expected: todos os itens acima conferem.

- [ ] **Step 5: Commit**

```bash
cd "/Users/felipebarbosa/Desktop/Claude/Hemoam/site"
git add index.html
git commit -m "Adiciona botões Salvar/Carregar dados (.json) na toolbar"
```

---

## Task 4: Verificação final ponta a ponta

**Files:**
- None (apenas verificação manual; nenhuma mudança de código esperada, a menos que a verificação encontre um problema — nesse caso, corrigir no arquivo `site/index.html` e commitar a correção antes de prosseguir).

- [ ] **Step 1: Rodar o checklist completo da spec (seção "Teste")**

Usando Playwright em `file:///Users/felipebarbosa/Desktop/Claude/Hemoam/site/index.html`, repetir em sequência, numa única sessão:

1. Preencher campos de dois leitos diferentes (ex: 413 e 414) com dados distintos, incluindo pelo menos um checkbox marcado em cada.
2. Recarregar a página → confirmar que ambos os leitos mantiveram os dados (autosave).
3. Clicar "Salvar dados (.json)" → confirmar o download do arquivo.
4. Limpar manualmente os campos de um dos leitos usando "Limpar leito" (ex: 413) → confirmar que só aquele leito foi afetado e que o outro (414) permanece intacto após um reload.
5. Carregar o arquivo salvo no passo 3 → confirmar que o leito 413 volta a ter os dados originais (o arquivo tinha o snapshot de antes da limpeza).
6. Tentar carregar um arquivo inválido → confirmar aviso de erro, sem alteração de dados.
7. Verificar a pré-visualização de impressão: nenhum dos botões novos (toolbar ou "Limpar leito") deve aparecer.

Expected: todos os 7 itens conferem, sem erros no console do navegador.

- [ ] **Step 2: Registrar o resultado**

Se todos os itens do Step 1 passarem, nenhuma ação adicional é necessária — a feature está completa. Se algum item falhar, corrigir o código relevante em `site/index.html`, repetir o Step 1 a partir do item que falhou, e commitar a correção:

```bash
cd "/Users/felipebarbosa/Desktop/Claude/Hemoam/site"
git add index.html
git commit -m "Corrige <descrição específica do problema encontrado na verificação final>"
```
