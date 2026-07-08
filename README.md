# Celestian Browser

Navegador de desktop construido com **Electron** (Chromium + Node.js), com bloqueador de anuncios nativo, atualizacao automatica e build multiplataforma automatizado via GitHub Actions.

---

## Visao geral da arquitetura

O app roda em dois "mundos" que se comunicam de forma segura:

- **Processo principal (`src/main.js`)** — o cerebro. Roda em Node.js e controla janelas, abas, sessoes, cabecalhos de rede e o bloqueador de anuncios.
- **Processo de interface (renderer)** — a barra do navegador, as abas e os popups que o usuario ve.
- **Ponte segura (`src/preload.js`)** — o unico canal entre os dois mundos, via **IPC** (mensagens), com `contextIsolation` ligado. A pagina nunca acessa o Node diretamente.

Cada aba nao e uma janela separada: e uma **WebContentsView** anexada a janela principal, todas compartilhando a mesma sessao (cookies e logins validos entre abas).

---

## Bloqueio de anuncios

Usa o motor **`@cliqz/adblocker-electron`** (a mesma engine usada pelo Brave/Ghostery), em **duas camadas**:

### 1. Bloqueio de rede
Intercepta cada requisicao e cancela as que batem com listas de anuncios e rastreadores (EasyList + EasyPrivacy, via `fromPrebuiltAdsAndTracking`). O anuncio nem chega a ser baixado — economia de dados e mais velocidade.

### 2. Filtragem cosmetica
Remove os anuncios que fazem parte da propria pagina (banners, overlays no player de video, blocos "Patrocinado"), que o bloqueio de rede sozinho nao pega:

```js
blocker.config.loadCosmeticFilters = true
blocker.config.enableMutationObserver = true // pega anuncios injetados depois
```

### Cache em disco
Na primeira execucao, as listas sao baixadas e salvas em `adblock-engine.bin` na pasta do usuario. Nas aberturas seguintes o motor carrega do arquivo local — abertura mais rapida e sem depender da rede.

### Otimizacao de performance
Uma pagina pode gerar centenas de bloqueios por segundo. Atualizar a interface a cada bloqueio travaria o navegador, entao o contador e agrupado (debounce) e a UI e atualizada no maximo a cada 800ms:

```js
function scheduleBlockedFlush() {
  if (blockedFlushTimer) return
  blockedFlushTimer = setTimeout(() => {
    blockedFlushTimer = null
    store.set("blockedCount", blockedCount)
    sendState()
  }, 800)
}
```

O usuario liga/desliga o bloqueio pelo icone de escudo, e o estado e persistido (`enableBlockingInSession` / `disableBlockingInSession`).

---

## Atualizacao automatica

Usa **`electron-updater`**, com o feed servido pelos **GitHub Releases** (repo `inside-exe/celestian-browser-releases`). Quem ja instalou **nao precisa baixar de novo**.

1. **Verificacao** — no arranque do app e depois **a cada 3 horas** (so funciona no app empacotado, nao em `npm start`):

   ```js
   const check = () => autoUpdater.checkForUpdates().catch(() => {})
   check()
   setInterval(check, 3 * 60 * 60 * 1000)
   ```

2. **Download automatico** em segundo plano assim que encontra versao nova (`autoDownload = true`), sem interromper o usuario.

3. **Notificacao nao intrusiva** — em vez de dialogos nativos, o app emite estados por IPC (`checking` -> `available` -> `downloading` -> `ready`) que um painel dentro do navegador exibe: versao atual, novidades e barra de progresso.

4. **Instalacao** — ao clicar em "Reiniciar agora", ou automaticamente ao fechar o app (`autoInstallOnAppQuit = true`):

   ```js
   function quitAndInstall() {
     autoUpdater.quitAndInstall()
   }
   ```

As notas de versao (`releaseNotes`) sao normalizadas para texto limpo (sem HTML) antes de aparecerem no painel.

---

## Build e publicacao (CI/CD)

Automacao via **GitHub Actions** (`.github/workflows/build-browser.yml`). Nao e preciso instalar nada localmente. Para lancar uma versao:

1. Aumente a `version` em `electron-app/package.json` (ex.: `1.2.5` -> `1.2.6`).
2. Crie e envie a tag:

   ```bash
   git tag v1.2.6 && git push origin v1.2.6
   ```

Isso dispara um build em **tres sistemas operacionais ao mesmo tempo** (matrix strategy):

| Sistema         | Instalador gerado      |
| --------------- | ---------------------- |
| windows-latest  | `.exe` (Windows)       |
| macos-14        | `.dmg` (macOS)         |
| ubuntu-latest   | `.AppImage` / `.deb`   |

Cada maquina instala as dependencias, compila e publica automaticamente (`--publish always`) os instaladores **e** os metadados de update (`latest.yml`, `latest-mac.yml`, `latest-linux.yml`) no repositorio de releases. O `electron-updater` de todos os apps instalados le esse mesmo feed: **publicou a tag, todos os usuarios recebem a atualizacao nas 3 horas seguintes**.

---

## Compatibilidade e seguranca

- **Sites reais:** ajusta cabecalhos (User-Agent e Client Hints) para se apresentar como Chrome e trata corretamente popups de login/SSO (`window.open`), preservando a ligacao `window.opener` — essencial para logins corporativos e fluxos OAuth.
- **AJAX:** preserva o cabecalho `X-Requested-With: XMLHttpRequest` para que bibliotecas como jQuery/DataTables recebam JSON corretamente.
- **Seguranca:** `contextIsolation` ligado, `nodeIntegration` desligado nas paginas, comunicacao apenas via IPC.

---

## Recursos

- Abas com sessao compartilhada
- Bloqueador de anuncios com contador
- Controle de volume por pagina (Web Audio)
- Zoom com Ctrl + roda do mouse
- Busca por voz
- Atalhos: F5, Ctrl+R (recarregar), F12 / Ctrl+Shift+I (DevTools)
- Painel de atualizacao integrado

---

## Desenvolvimento local

```bash
cd electron-app
npm install
npm start          # roda em modo desenvolvimento (auto-update desativado)
```

Gerar instalador manualmente (compila para o SO onde roda):

```bash
npm run dist:win    # Windows (.exe)
npm run dist:mac    # macOS (.dmg)
npm run dist:linux  # Linux (AppImage/deb)
```

O instalador aparece na pasta `dist/`.

> Sem certificado de assinatura de codigo, o Windows SmartScreen pode alertar na primeira
> instalacao ("editor desconhecido"). O app funciona normalmente, e as atualizacoes
> seguintes via `electron-updater` sao aplicadas sem aviso. Isso vale para qualquer app
> Electron nao assinado.

---

## Stack

- **Electron** — runtime (Chromium + Node.js)
- **@cliqz/adblocker-electron** — motor de bloqueio
- **electron-updater** — atualizacao automatica
- **electron-builder** — empacotamento multiplataforma
- **GitHub Actions** — CI/CD
