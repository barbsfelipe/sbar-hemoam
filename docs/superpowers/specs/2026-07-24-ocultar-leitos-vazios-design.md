# Ocultar leitos vazios

## Contexto

O `site/index.html` renderiza sempre os mesmos 8 leitos fixos (409 a 416,
constante `beds` em `index.html`), um cartão (`.bed-card`) por leito, cada
um virando uma página A4 paisagem na impressão. Não existe hoje nenhuma
noção de leito "vazio" ou "ocupado" — os 8 cartões sempre aparecem, cheios
ou não.

Na prática, em boa parte dos plantões nem todos os 8 leitos estão
ocupados. Rolar a tela por cartões inteiros em branco (cada um bem
grande, pensado para impressão) atrapalha quem está preenchendo os
leitos que de fato têm paciente. O pedido é poder esconder os leitos sem
nenhum dado preenchido, tanto na tela quanto na impressão.

## Objetivos

1. Um botão único na toolbar liga/desliga um filtro que oculta os leitos
   sem nenhum campo preenchido — sem precisar marcar leito por leito.
2. A ocultação vale tanto para a tela quanto para a impressão/PDF: com o
   filtro ligado, leitos vazios não geram página impressa.
3. A visibilidade se atualiza sozinha conforme os dados mudam (digitação,
   "Limpar leito", carregar arquivo, restaurar autosave) — nunca fica um
   leito com dado escondido, nem um leito vazio visível por engano depois
   que o filtro foi ligado.
4. A preferência (filtro ligado/desligado) persiste entre recarregamentos
   da página, para não precisar religar toda vez.

## Não-objetivos

- Ocultar leitos manualmente por escolha do usuário, independente de
  terem dado ou não (ex.: um botão "esconder" por cartão). Fica de fora
  porque o pedido é especificamente sobre leitos vazios.
- Mudar a lista fixa de 8 leitos (409–416) ou permitir adicionar/remover
  leitos.
- Mudar o que conta como "salvar dados" — a detecção de vazio é só para
  efeito de exibição, não altera `coletarEstado()`, o JSON salvo/carregado
  nem o autosave em arquivo.

## O que conta como "leito vazio"

Um leito é vazio quando **nenhum** dos seus elementos `[data-field]` tem
valor: nenhum checkbox marcado e nenhum input de texto com conteúdo
(depois de `trim()`). Isso cobre nome, mãe, diagnóstico, todos os
checkboxes de S/B/A/R, campos de ventilação, assinaturas etc. — tudo que
já é lido por `coletarEstado()` hoje.

Função nova `leitoEstaVazio(card)`, reaproveitando o mesmo seletor
`card.querySelectorAll('[data-field]')` já usado em `limparLeito` e
`coletarEstado`.

## Mecanismo

### Botão na toolbar

Novo botão `#toggle-vazios-btn`, ao lado do botão "Imprimir / Salvar PDF"
existente, com texto que alterna conforme o estado:

- Filtro desligado: `Ocultar leitos vazios`
- Filtro ligado: `Mostrar leitos vazios (N ocultos)` — o número deixa
  claro, antes de imprimir, quantos leitos não vão sair no papel com o
  filtro ligado.

### Aplicação visual

Classe CSS nova `.bed-hidden { display: none; }`, **sem** exceção de
mídia de impressão (diferente de `.no-print`, que só esconde ao
imprimir) — precisa esconder nos dois casos.

### Atualização de visibilidade

Função nova `atualizarVisibilidadeLeitosVazios()`:

1. Para cada `.bed-card`, calcula `leitoEstaVazio(card)`.
2. Se o filtro estiver ligado e o leito for vazio, adiciona
   `bed-hidden`; senão remove.
3. Atualiza o texto do botão com a contagem atual de leitos ocultos.

Chamada nos seguintes pontos:

- Ao clicar em `#toggle-vazios-btn` (inverte o estado do filtro antes de
  chamar).
- A cada evento `input`/`change` no documento (mesmo listener que já
  dispara `salvarEstadoLocalDebounced` — chamada direta, não debounced,
  já que iterar 8 cartões é barato e o usuário se beneficia de ver a
  atualização imediata).
- Depois de `limparLeito(bed)`.
- Depois de `restaurarEstado(dados)` (cobre autosave local, retomada via
  File System Access API e "Carregar dados (.json)").
- Uma vez no carregamento inicial da página, depois de restaurar o
  estado salvo.

### Persistência da preferência

Estado do filtro (`true`/`false`) guardado em `localStorage`, chave
própria `hemoam-isbar-ocultar-vazios` — separada da chave de dados
(`hemoam-isbar-utip-rascunho`) e do JSON salvo/carregado, porque é
preferência de visualização, não dado clínico. Lida uma vez no
carregamento da página para definir o estado inicial do botão.

## Tratamento de erros

- Leitura/escrita da preferência no `localStorage` segue o mesmo padrão
  já usado em `salvarEstadoLocal` — `try/catch` silencioso; se
  indisponível, o filtro simplesmente começa desligado a cada carga, sem
  quebrar o restante da página.

## Teste

Validar manualmente (Chrome, via Playwright ou navegador):

1. Com todos os 8 leitos vazios, clicar em "Ocultar leitos vazios" →
   todos os cartões somem da tela, botão passa a mostrar
   `Mostrar leitos vazios (8 ocultos)`.
2. Preencher um campo de um leito oculto não é possível diretamente (o
   cartão está escondido) — clicar em "Mostrar leitos vazios", preencher
   um campo de texto em um leito, confirmar que ele **não** some ao
   religar o filtro.
3. Com o filtro ligado e um leito preenchido, clicar em "Limpar leito"
   nesse leito → o cartão some imediatamente e a contagem do botão sobe.
4. Com o filtro ligado, usar "Carregar dados (.json)" apontando para um
   arquivo com dados em leitos que hoje estão vazios/ocultos → confirmar
   que esses leitos reaparecem automaticamente após a restauração.
5. Com o filtro ligado e ao menos um leito oculto, clicar em
   "Imprimir / Salvar PDF" (ou `window.print()` via preview) → confirmar
   que o leito oculto não gera página na pré-visualização de impressão.
6. Recarregar a página com o filtro ligado → confirmar que ele continua
   ligado (preferência persistida) e que a contagem exibida bate com os
   leitos realmente vazios no momento da carga.
7. Desligar o filtro → todos os 8 leitos voltam a aparecer, inclusive os
   vazios.
