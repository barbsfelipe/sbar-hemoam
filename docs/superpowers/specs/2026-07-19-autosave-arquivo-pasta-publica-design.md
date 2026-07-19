# Salvamento automático em arquivo na pasta pública (continuidade entre logins)

## Contexto

O `site/index.html` já implementa (spec `2026-07-15-continuidade-isbar-design.md`):
autosave silencioso em `localStorage` a cada campo digitado, e dois botões
manuais na toolbar — "Salvar dados (.json)" (baixa um arquivo) e "Carregar
dados (.json)" (lê um arquivo escolhido pelo usuário).

O arquivo `index.html` fica salvo numa pasta de rede compartilhada
("pública") de um computador do Hemoam, e é aberto por duplo clique
(`file://...`) — não há servidor. Vários plantonistas usam o mesmo
computador físico, cada um com seu próprio login do Windows. Como
`localStorage` fica isolado por perfil de navegador (e cada login do
Windows tem seu próprio perfil), o autosave atual **não aparece** para
quem entra com outro login: a pessoa precisa lembrar de clicar em
"Carregar dados (.json)" e escolher manualmente o arquivo salvo por quem
usou o computador antes.

O pedido agora é reduzir esse atrito: ao trocar de login no mesmo
computador, abrir o `index.html` da pasta pública deve mostrar o que foi
escrito antes, com o mínimo de passos manuais possível. Como a máquina
roda tanto Chrome quanto Firefox (varia por plantonista) e o arquivo é
aberto via `file://` (sem servidor), duas restrições de segurança do
navegador entram em jogo:

- Uma página `file://` não pode ler/escrever arquivos em disco sozinha,
  sem alguma ação explícita do usuário.
- Só o Chrome/Edge (via *File System Access API*) permite que uma página
  peça permissão, uma vez, para ler e escrever repetidamente num arquivo
  específico do disco. O Firefox não implementa essa API.

Rodar um servidor local (para ter autosave 100% automático em qualquer
navegador) foi considerado e descartado por ora: exigiria instalar e
manter um serviço rodando nesse computador do Hemoam independente de
quem está logado, o que é um desvio maior da decisão já tomada
(spec anterior, seção "Contexto") de não ter backend para evitar
complexidade e riscos de LGPD com dados de saúde. Fica registrado como
alternativa possível para o futuro, não como parte deste trabalho.

## Objetivos

1. No Chrome/Edge, permitir que o plantonista ative, com um único clique
   por login do Windows, um salvamento automático que grava as
   alterações direto num arquivo dentro da própria pasta pública —
   assim, ao trocar de login e reabrir o `index.html`, os dados do
   plantão anterior aparecem sozinhos (sem precisar clicar em "Carregar"
   de novo), desde que a permissão do navegador ainda esteja válida.
2. Manter o Firefox funcionando exatamente como hoje (autosave em
   `localStorage` + botões manuais "Salvar/Carregar .json"), já que a
   API necessária não existe nesse navegador.
3. Usar um nome de arquivo fácil de reconhecer na pasta pública por
   alguém sem intimidade com informática.

## Não-objetivos

- Sincronização automática entre computadores diferentes (mantém-se a
  limitação já documentada na spec anterior).
- Autosave 100% automático (zero cliques) no Firefox — sem a API, não é
  tecnicamente possível sem servidor.
- Mesclagem de edições concorrentes: se dois logins tiverem o mesmo
  arquivo "aberto" ao mesmo tempo, quem salvar por último sobrescreve o
  outro. Aceito como risco baixo (mesmo computador físico, uso
  sequencial por troca de turno).
- Rodar um servidor/serviço local nesse computador (ver "Contexto").
- Renomear ou alterar o comportamento do autosave em `localStorage` já
  existente — ele continua ativo e intacto como rede de segurança,
  independente do mecanismo novo.

## Nome do arquivo

Um único arquivo por instalação, criado uma vez com o nome
`plantão (dd-mm-aaaa).json` — a data de **criação**, não a de hoje;
o nome não muda mais depois de criado, o conteúdo é que vai sendo
sobrescrito. Esse mesmo padrão de nome passa a valer também para o
download do botão manual "Salvar dados (.json)" já existente (troca o
nome atual `isbar-utip-hemoam-AAAA-MM-DD.json` por
`plantão (dd-mm-aaaa).json`), para não haver dois padrões de nome
diferentes confundindo quem navega pela pasta.

O conteúdo do arquivo é o mesmo JSON já produzido por `coletarEstado()`
(spec anterior) — não muda o formato dos dados, só o mecanismo de
gravação/leitura e o nome do arquivo.

## Mecanismo — Salvamento automático via File System Access API

### Detecção de suporte

Ao carregar a página, verifica `'showSaveFilePicker' in window`. Se
`false` (Firefox e navegadores sem suporte), o botão "Ativar salvamento
automático" simplesmente não é criado/exibido — o restante da toolbar
(botões já existentes) funciona normalmente.

### Ativação (1 clique por login)

Botão **"Ativar salvamento automático"** na toolbar, ao lado dos botões
já existentes. Ao clicar:

1. Chama `window.showSaveFilePicker({ suggestedName: 'plantão (<data de hoje dd-mm-aaaa>).json', types: [{ description: 'Dados ISBAR', accept: { 'application/json': ['.json'] } }] })`.
2. O plantonista navega até a pasta pública (o navegador lembra a última
   pasta usada por perfil) e:
   - seleciona o arquivo `plantão (...)` que já existe lá — o picker
     trata isso como "salvar sobre esse arquivo"; ou
   - mantém o nome sugerido para criar um novo, se for a primeira vez
     nessa pasta.
3. Lê o conteúdo atual do arquivo escolhido (`handle.getFile()` →
   `.text()`).
   - Se o conteúdo for um JSON válido e não vazio: mostra
     `confirm("Isso vai substituir os dados atuais na tela pelos dados do arquivo selecionado. Continuar?")` — mesmo texto/padrão do "Carregar" já existente. Se confirmado, chama
     `restaurarEstado(dados)`. Se cancelado, mantém a tela como está
     (mas a ativação segue adiante — ver próximo passo).
   - Se o arquivo estiver vazio/novo (sem conteúdo JSON): grava o estado
     atual da tela nele imediatamente (`coletarEstado()` →
     `handle.createWritable()` → `write` → `close`), sem perguntar nada.
4. Guarda o `FileSystemFileHandle` no `IndexedDB` (banco/object store
   dedicado, chave fixa) para reuso nas próximas cargas de página do
   mesmo perfil/login.
5. Marca o autosave-em-arquivo como "ativo" (variável em memória) e
   mostra um texto de status na toolbar, ex.: `Salvamento automático
   ativo (plantão (19-07-2026).json)`.
6. Troca o próprio botão "Ativar salvamento automático" por um botão
   discreto "Desativar", que só limpa o handle guardado e o estado
   "ativo" em memória (não apaga nem altera o arquivo em disco).

### Autosave contínuo

Reaproveita o mesmo debounce (~400ms) do autosave em `localStorage`
já existente: a cada `input`/`change`, além de gravar em
`localStorage`, se o autosave-em-arquivo estiver ativo, grava também
nele (`coletarEstado()` → `createWritable()` → `write` → `close()`).

### Retomada automática ao carregar a página (mesmo login)

Ao carregar `index.html`:

1. Tenta ler o handle salvo no `IndexedDB`.
2. Se existir, chama `handle.queryPermission({ mode: 'readwrite' })`
   (não exige gesto do usuário).
   - Se `'granted'`: lê o arquivo, chama `restaurarEstado(dados)`
     automaticamente (sem nenhum clique) e ativa o autosave contínuo.
   - Se `'prompt'` (permissão precisa ser reconfirmada, ex. navegador
     reiniciado): mostra um aviso discreto e não bloqueante — "Clique
     para retomar o salvamento automático neste arquivo" — que, ao
     clicar, chama `handle.requestPermission({ mode: 'readwrite' })`
     (isso sim exige gesto do usuário) e, se concedido, segue como no
     caso `'granted'`.
   - Se o handle não existir mais (arquivo movido/apagado) ou
     `getFile()` falhar: trata como "sem autosave em arquivo ativado
     ainda" — remove o handle inválido do `IndexedDB` e mostra o botão
     normal de ativação.
3. Se não existir handle salvo (primeiro acesso desse login/perfil):
   mostra só o botão "Ativar salvamento automático", nada acontece
   automaticamente.

Em todos os casos, o autosave em `localStorage` e a restauração dele ao
carregar a página continuam funcionando exatamente como hoje, em
paralelo — servem de rede de segurança independente do arquivo.

## Tratamento de erros

- Qualquer falha ao escrever no arquivo (pasta de rede indisponível,
  arquivo movido/apagado, permissão revogada no meio do uso) é
  capturada (`try/catch`) e não interrompe o formulário: mostra um
  aviso discreto "Não foi possível salvar no arquivo compartilhado — os
  dados continuam salvos neste navegador" e mantém o `localStorage`
  como rede de segurança. Não trava a digitação nem lança exceção não
  tratada.
- Falha ao **ler** o arquivo na ativação ou na retomada automática (ex.:
  JSON corrompido): mesmo padrão do "Carregar" já existente — aviso de
  erro, estado da tela preservado, autosave em arquivo não é ativado
  até nova tentativa.
- Clicar em "Ativar" e cancelar o seletor nativo do navegador (sem
  escolher arquivo): não faz nada, formulário segue como estava, botão
  continua disponível para nova tentativa.

## Teste

Validar manualmente no Chrome (via Playwright ou navegador):

1. Preencher um leito, clicar "Ativar salvamento automático", criar um
   arquivo novo `plantão (data).json` numa pasta de teste → confirmar
   que grava sem perguntar nada (arquivo era novo) e que o status na
   toolbar aparece.
2. Preencher outro campo, aguardar ~1s, abrir o arquivo no disco (fora
   do navegador) e confirmar que o conteúdo já reflete a alteração.
3. Fechar a aba/recarregar a página (mesmo perfil) → confirmar que os
   dados aparecem automaticamente, sem nenhum clique (permissão
   `'granted'` silenciosa).
4. Simular um novo "login" abrindo o mesmo `index.html` num contexto de
   navegador diferente (ex.: perfil separado ou aba anônima) apontando
   para o mesmo arquivo → confirmar que aparece o botão "Ativar", e que
   ao clicar e escolher o arquivo já existente (com dados), aparece o
   `confirm()` de substituição; confirmando, os dados do arquivo
   aparecem na tela.
5. Renomear/mover o arquivo em disco enquanto o autosave está ativo,
   digitar algo novo → confirmar que aparece o aviso de erro discreto e
   que o formulário continua funcional (dado continua salvo no
   `localStorage`).
6. Conferir que no Firefox (ou forçando `'showSaveFilePicker' in
   window === false` via DevTools) o botão "Ativar salvamento
   automático" não aparece, e que os botões "Salvar/Carregar .json"
   continuam funcionando como antes, agora baixando com o nome
   `plantão (dd-mm-aaaa).json`.
7. Confirmar que os elementos novos (botão de ativar/desativar, aviso de
   status/erro) ficam ocultos na pré-visualização de impressão (mesma
   regra `.no-print`/`.toolbar` já existente).
