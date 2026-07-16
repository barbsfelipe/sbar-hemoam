# Continuidade de dados no ISBAR (autosave + salvar/carregar arquivo) + limpar leito

## Contexto

O site (`site/index.html`) é um formulário ISBAR de passagem de plantão da UTI
Pediátrica (8 leitos: 409–416), pensado originalmente como formulário de
impressão: os campos digitados de manhã são impressos e completados a
lápis/caneta ao longo do dia. Hoje não existe nenhuma persistência — ao
recarregar a página, todos os campos voltam em branco.

O ISBAR é um instrumento de continuidade assistencial: os dados de um leito
devem poder ser reaproveitados/editados no plantão seguinte, e um leito
precisa ser esvaziado rapidamente quando o paciente recebe alta, sem afetar
os demais leitos.

O site é hospedado como página estática no GitHub Pages
(`github.com/barbsfelipe/sbar-hemoam`), sem backend. A equipe acessa de
aparelhos diferentes ao longo do dia (não é sempre o mesmo computador), então
uma sincronização automática entre aparelhos exigiria um servidor com banco
de dados — fora de escopo aqui por ser um projeto maior (hospedagem,
autenticação, cuidados de LGPD para dados de saúde). A solução abaixo evita
essa complexidade mantendo os dados sempre sob controle do usuário
(navegador local ou arquivo baixado), sem enviar nada a terceiros.

Há um precedente no repositório irmão `Admissão e evolução/index.html`: um
padrão de autosave em `localStorage` com debounce, restauração ao carregar a
página, e um botão "Limpar formulário" com `confirm()`. Reaproveitamos o
mesmo padrão aqui, adaptado à estrutura por leito.

## Objetivos

1. Uma opção para salvar os dados preenchidos e poder editá-los no dia/plantão
   seguinte, inclusive em outro aparelho.
2. Um botão por leito que limpe individualmente os dados daquele leito (em
   caso de alta do paciente), sem afetar os outros leitos.

## Não-objetivos

- Sincronização automática em tempo real entre aparelhos/usuários.
- Backend, banco de dados ou autenticação.
- Persistir os campos de "Intercorrências / evolução — TARDE/NOITE": esses
  continuam sendo blocos pautados apenas para preenchimento manual a caneta
  após a impressão (não são `<input>`/`<textarea>`, então não fazem parte do
  estado digital).

## Arquitetura de dados

Cada campo editável (inputs de texto e checkboxes) dentro de um `.bed-card`
recebe um atributo `data-field` com uma chave curta e semântica, única
dentro do card (ex.: `nome`, `nomeMae`, `dataInternacao`, `idadeDN`,
`alergias_sim`, `alergias_nao`, `disp_cvc`, `disp_cvc_val`, `diagnosticos`,
`atb_sim`, `atb_nao`, `vent_fio2`, `condutas`, `alta_sim`, `alta_leito`,
`assinatura`, etc.). Cada `.bed-card` recebe `data-bed="409"` (etc.),
permitindo isolar leitura/escrita/limpeza por leito.

O campo de data no topo da página 1 (`.page-date input`) recebe um id
próprio (`page-date`) e é tratado como campo global, fora do laço de leitos.

**Coleta de estado** (`coletarEstado()`): percorre cada `.bed-card`,
monta `{ pageDate, beds: { "409": { campo: valor, ... }, ... } }`, com
valores de texto como string e checkboxes como booleano.

**Restauração de estado** (`restaurarEstado(dados)`): para cada leito
presente em `dados.beds`, localiza `.bed-card[data-bed=...]` e aplica os
valores aos elementos com o `data-field` correspondente. Leitos ausentes no
objeto carregado permanecem como estão na tela (não apaga o que já foi
digitado).

## Mecanismo 1 — Autosave local (localStorage)

- Em todo evento `input`/`change` no documento, agenda salvamento
  (debounce ~400ms) da chave `hemoam-isbar-utip-rascunho` com
  `JSON.stringify(coletarEstado())`.
- Ao carregar a página, tenta ler essa chave e restaurar automaticamente,
  antes de qualquer outra interação do usuário.
- Falhas de `localStorage` (indisponível/cheio) são ignoradas silenciosamente
  (`try/catch`), sem quebrar o formulário — mesmo padrão do site irmão.
- Serve como rede de segurança contra fechar a aba sem querer; **não**
  resolve o caso de troca de aparelho (localStorage é local ao navegador).

## Mecanismo 2 — Salvar/Carregar arquivo (.json)

Dois novos botões na `.toolbar` (ao lado de "Imprimir / Salvar PDF"):

- **"Salvar dados (.json)"**: chama `coletarEstado()`, gera um Blob JSON e
  dispara o download como `isbar-utip-hemoam-AAAA-MM-DD.json` (data de hoje).
  Mostra uma mensagem breve confirmando o nome do arquivo salvo.
- **"Carregar dados (.json)"**: abre um `<input type="file" accept=".json">`
  oculto. Ao selecionar um arquivo:
  1. Pergunta `confirm("Isso vai substituir os dados atuais na tela pelos dados do arquivo carregado. Continuar?")`.
  2. Se confirmado, lê o arquivo (`FileReader`), faz `JSON.parse`; em caso de
     erro de leitura/parse, mostra aviso de erro e não altera nada na tela.
  3. Se válido, chama `restaurarEstado(dados)` e, em seguida, força um
     salvamento imediato no `localStorage` (para que o autosave local
     também reflita o arquivo recém-carregado).

Esses botões ficam dentro da `.toolbar`, que já é ocultada via
`@media print { .toolbar { display: none; } }`, então não aparecem na
impressão.

## Mecanismo 3 — Botão "Limpar leito" (por card)

Cada `.bed-card` ganha um botão pequeno **"Limpar leito"** na
`.header-strip` do próprio card (ao lado do rótulo "LEITO 409"), com classe
`no-print` (nova regra `@media print { .no-print { display: none; } }` para
reuso).

Comportamento ao clicar:
1. `confirm("Limpar todos os dados do leito 409? Essa ação não pode ser desfeita.")`.
2. Se confirmado: percorre apenas os elementos `[data-field]` **dentro
   daquele `.bed-card`** e zera inputs de texto (`value = ''`) e desmarca
   checkboxes (`checked = false`). O número do leito (fixo no HTML, fora do
   `data-field`) não é afetado.
3. Dispara salvamento imediato no `localStorage` para refletir a limpeza.
4. Não reescreve nenhum arquivo `.json` já baixado anteriormente (não é
   possível a partir do navegador); cabe ao usuário clicar em "Salvar dados"
   de novo quando quiser gerar um arquivo atualizado sem aquele paciente.

## Tratamento de erros

- `localStorage` indisponível/cheio → autosave falha silenciosamente
  (try/catch), formulário continua funcional.
- Arquivo `.json` inválido/corrompido no "Carregar" → aviso de erro, estado
  atual da tela preservado.
- Ações destrutivas (Carregar substituindo tela, Limpar leito) sempre pedem
  `confirm()` antes de executar.

## Teste

Após a implementação, validar manualmente no navegador (via skill
`webapp-testing`/Playwright ou Chrome):
1. Preencher campos de um leito, recarregar a página → dados restaurados
   (autosave).
2. Clicar "Salvar dados" → arquivo `.json` baixado com o conteúdo esperado.
3. Limpar os campos na tela (recarregar sem autosave ou usar outro
   perfil/aba anônima) e "Carregar" o arquivo salvo → dados restaurados
   corretamente em todos os leitos.
4. Preencher dois leitos, clicar "Limpar leito" em um deles → apenas aquele
   leito é zerado, o outro permanece intacto, e isso persiste após reload
   (autosave atualizado).
5. Tentar carregar um arquivo `.json` inválido → aviso de erro, nada muda na
   tela.
6. Conferir que os botões novos (toolbar e por leito) não aparecem na
   pré-visualização de impressão.
