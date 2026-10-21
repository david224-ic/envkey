# EnvKey — como buildar desktop, CLI e envkey-source (mac / linux / windows)

Este repo é o **espelho público** do EnvKey (projeto em "wind down"). Ele tem o código-fonte, mas **não tem** o
pipeline de release: o config do `electron-builder`, a notarização e o upload ficavam num repo privado.
Tudo abaixo foi reconstruído por engenharia reversa e **testado só em parte** (ver "O que NÃO foi verificado").

Artefatos que sabemos gerar, a partir de um Mac arm64 (cross-build):

| Artefato | Alvos |
|---|---|
| App desktop 2.4.18 | mac arm64 (`.app`), windows x64 (`.exe` NSIS + `.zip`), linux x64 (`.AppImage` + `.tar.gz`) |
| CLI 2.5.0 (binário `pkg`) | mac arm64/amd64, linux amd64/arm64, windows amd64 |
| envkey-source 2.4.3 | mac arm64/amd64, linux amd64/arm64, windows amd64 |

Todos **sem assinatura da Apple/Microsoft** (mac só ad-hoc).

## Fatos importantes (leia antes de buildar)

- **Não existem releases para baixar.** O bucket S3 `envkey-releases` foi apagado (`NoSuchBucket`). Os binários do CLI e do
  envkey-source têm que ser compilados daqui. O auto-update do app desktop também está morto (a checagem falha, é
  capturada e logada — não trava o app).
- **O app em produção chama o CLI como executável.** `electron/src/core_proc.ts` faz spawn de `BUNDLED_CLI_PATH core start`,
  então o app só funciona com o binário do CLI dentro de `bin/`. O CLI é um binário `pkg` (Node empacotado).
- **Layout de `bin/` dentro do app** (definido em `public/app/client/electron/src/platform.ts`; `cli-<versão>` vem de
  `public/app/client/cli/package.json`):
  - mac: `bin/mac/<arm64|amd64>/cli-2.5.0/{envkey,envkey-keytar.node}` + `bin/mac/<arch>/envkey-source`
  - windows: `bin/windows/cli-2.5.0/{envkey.exe,envkey-keytar.node}` + `bin/windows/envkey-source.exe`
  - linux: `bin/linux/cli-2.5.0/envkey` + `bin/linux/envkey-source` (**sem** subpasta de arquitetura e **sem** keytar:
    o keychain é desativado no Linux em `core/src/lib/client_store/key_store.ts`)
  - O CLI carrega o keychain de `./envkey-keytar.node` **ao lado do binário** (`key_store.ts`). Sem esse arquivo o
    keychain não funciona (mac e windows).
- **`/snapshot/...` hardcoded no CLI.** `cli/src/lib/core.ts` referencia
  `/snapshot/v2/private/ci_cd/cli-builder/build/workspace/{envkey-core.js,worker.js}` (o caminho do CI privado). O `pkg`
  monta o snapshot a partir do diretório do entry, então trocamos essa string no bundle pelo caminho real do nosso
  workspace (`/snapshot` + caminho absoluto do dir de build) com `sed`, ver passo 4.
- **`keytar` é N-API** (prebuilds `napi-v3`): o mesmo `keytar.node` serve para Node e Electron; basta um por plataforma.
- **Referência oficial:** se `/Applications/EnvKey.app` (2.4.18) estiver instalado, é o melhor modelo de layout
  (appId `com.envkey.appv2`, `asar: false`, entitlement `allow-unsigned-executable-memory`, `icon.icns`).
  O `electron/icon/` do repo só tem um PNG 64x64; o `icon.icns` vem desse app.
- **NÃO rode `envkey core start/stop` com o EnvKey instalado aberto.** O core process usa a porta 19047 e o CLI novo
  derruba/substitui o core mais antigo em execução. Para testar o app novo, feche o EnvKey instalado primeiro.
- Versões: Node **16.13.2** (`.nvmrc`), Electron **13.2.1**, `electron-builder` 22.14.13, `pkg` 5.8.1, Go ≥ 1.16
  (usamos 1.26). App desktop 2.4.18 (`releases/desktop/desktop-version.txt`).

## Armadilhas já encontradas

1. **npm 10 quebra com Node 16.** O `npm ci` de `public/app` falha com `ERESOLVE` (react-beautiful-dnd) usando o npm 10 do
   shim do proto. Use o **npm que vem junto com o node 16.13.2 (8.1.2)**. E atenção: `npm ci` **apaga `node_modules` antes de
   instalar** — se falhar no meio, a pasta fica vazia.
2. **Python ≥ 3.12 não tem `distutils`** → o `node-gyp` antigo (lzma-native etc.) falha. Solução: venv com `setuptools` e
   `npm_config_python` apontando para ele.
3. **O `pkg` baixa o Node base de cada alvo do GitHub (`vercel/pkg-fetch`, tag `v3.4`, Node `v16.16.0`).** Se o download falha
   (a CDN de release-assets do GitHub já deu 503 por horas), o `pkg` tenta **compilar o Node do zero** e dá erro de asm
   (`x86_64-gcc.c`) ao cross-buildar. Os arquivos precisam estar em `~/.pkg-cache/v3.4/fetched-v16.16.0-<alvo>`.
4. **Sem CDN, nada de win/linux sai:** Electron (`github.com/electron/electron/releases`), NSIS/AppImage
   (`electron-userland/electron-builder-binaries`), pkg base e keytar prebuilds vêm todos de GitHub release assets.
5. **NSIS exige ícone PNG ≥256 ou ICO** — `.icns` dá `image: unknown format`. Converta: `sips -s format png icon.icns --out icon.png`.
6. **Windows sem wine:** `signAndEditExecutable: false` é necessário no config `win` (não editamos recursos do `.exe`).
7. `electron/package.json` → `npm run build` faz `rm -rf dist` e `cp -R bin dist/`: a pasta `electron/bin` (está no `.gitignore`) tem
   que **existir** (pode ficar vazia; montamos o `bin/` de cada plataforma depois, no staging).
8. Disco: um build completo ocupa ~3 GB + ~600 MB de setup. Limpe `$B` depois de gerar o zip.
9. Ao rodar comandos pelo Claude Code: `rm -rf $VAR/...` é bloqueado pelo safety check — use caminhos literais ou
   `"${VAR:?}"/...`. No zsh, globs sem aspas (`--include=*.ts`) quebram o comando.

## Passo a passo

### 0. Setup único (idempotente)

```bash
public/scripts/setup_desktop_build          # --force reinstala node_modules
source ~/.envkey-build/env.sh               # em TODA sessão nova, antes de buildar
```

O script instala Node 16.13.2 (proto ou fnm), cria o venv de Python se precisar, faz `npm ci` na raiz e em `public/app`
(com o npm bundled do node 16), instala `pkg@5.8.1` em `~/.envkey-build/pkgtool`, baixa os 5 Node base do pkg para
`~/.pkg-cache` (com sha256 e retry), baixa os `keytar` N-API (darwin-arm64, darwin-x64, win32-x64, linux-x64) para
`~/.envkey-build/keytar/<plat>/build/Release/keytar.node` e roda `go mod download`. O `env.sh` exporta `PATH` (node 16),
`npm_config_python`, `NODE_OPTIONS=--max-old-space-size=8192`, `PKG_CACHE_PATH`, `ENVKEY_PKG_BIN`, `ENVKEY_KEYTAR_DIR`.

Variáveis usadas nos passos seguintes (saídas ficam **fora** do repo):

```bash
REPO=$HOME/Dev/src/envkey
B=$HOME/envkey-build            # intermediários e saída
W=$B/cliws                      # caminho ABSOLUTO real, sem symlinks (entra no snapshot do pkg)
mkdir -p $B/out $W
```

### 1. Bundle do worker (core)

```bash
$REPO/public/scripts/build_worker           # → public/app/core/build/worker.js
```

### 2. UI

```bash
(cd $REPO/public/app/client/ui && npm run build)   # → ui/dist: index.js, stripe_form.js, *.woff2 (+ .map)
```

### 3. Main + preload do Electron (produção)

```bash
mkdir -p $REPO/public/app/client/electron/bin
(cd $REPO/public/app/client/electron && NODE_ENV=production npm run build)
# → electron/dist: bundle.js, preload.js, index.html, stripe_form.html, worker.js, icon/ (+ *.dev.html, .map, bin/)
```

`NODE_ENV=production` é obrigatório (carrega `index.html` em vez de `index.dev.html` e usa os caminhos
`process.resourcesPath/app/{worker.js,bin,icon}`).

### 4. CLI 2.5.0 (webpack + pkg, todos os alvos)

```bash
(cd $REPO/public/app/client/cli && WEBPACK_OUTPUT_DIR=$W npm run build)
#   → $W/envkey-cli.js, envkey-core.js, keytar.node (+ .map)
cp $REPO/public/app/core/build/worker.js $W/

# troca o caminho do snapshot do CI privado pelo nosso (BSD sed; em GNU sed use `sed -i` sem '')
sed -i '' "s#/snapshot/v2/private/ci_cd/cli-builder/build/workspace#/snapshot$W#g" $W/envkey-cli.js $W/envkey-core.js

cat > $W/package.json <<'EOF'
{ "name": "envkey-cli", "version": "2.5.0", "bin": "envkey-cli.js",
  "pkg": { "scripts": ["envkey-core.js", "worker.js"], "assets": ["keytar.node"] } }
EOF

(cd $W && $ENVKEY_PKG_BIN . --no-bytecode --public \
  --targets node16-macos-arm64,node16-macos-x64,node16-linux-x64,node16-linux-arm64,node16-win-x64 \
  --out-path $B/out/cli-raw)
# → $B/out/cli-raw/envkey-cli-{macos-arm64,macos-x64,linux-x64,linux-arm64,win-x64.exe}
```

Os warnings do pkg sobre `xdg-open` e `fastlist.exe` são inofensivos. Teste rápido do binário do host:
`$B/out/cli-raw/envkey-cli-macos-arm64 --version` → `2.5.0` (**não** rode `core start` com o app instalado aberto).

### 5. envkey-source 2.4.3 (Go, cross-compile)

A versão precisa ser injetada, senão reporta `UNVERSIONED` (versão em `releases/envkeysource/envkeysource-version.txt`).

```bash
cd $REPO/public/sdks/envkey-source
for t in darwin/arm64 darwin/amd64 linux/amd64 linux/arm64 windows/amd64; do
  os=${t%/*}; ar=${t#*/}; ext=""; [ $os = windows ] && ext=".exe"
  mkdir -p $B/out/$os-$ar
  CGO_ENABLED=0 GOOS=$os GOARCH=$ar go build -trimpath \
    -ldflags "-s -w -X github.com/envkey/envkey/public/sdks/envkey-source/version.Version=2.4.3" \
    -o $B/out/$os-$ar/envkey-source$ext .
done
```

### 6. Staging do app (um diretório por plataforma)

```bash
E=$REPO/public/app/client/electron/dist; U=$REPO/public/app/client/ui/dist; S=$B/app
mkdir -p $S
cp -R $E/icon $S/
cp $E/bundle.js $E/preload.js $E/index.html $E/stripe_form.html $E/worker.js $S/
cp $U/index.js $U/stripe_form.js $U/*.woff2 $S/
cat > $S/package.json <<'EOF'
{ "name": "envkey", "version": "2.4.18", "description": "EnvKey UI",
  "author": "Envkey Inc <support@envkey.com>", "license": "MIT",
  "main": "bundle.js", "dependencies": { "keytar": "^7.7.0" } }
EOF
(cd $S && npm i --production)        # keytar (JS) + deps em node_modules

for p in mac win linux; do cp -R $S $B/app-$p; done
K=$ENVKEY_KEYTAR_DIR; O=$B/out; R=$O/cli-raw

# mac arm64
A=$B/app-mac; mkdir -p $A/bin/mac/arm64/cli-2.5.0
cp $R/envkey-cli-macos-arm64 $A/bin/mac/arm64/cli-2.5.0/envkey
cp $K/darwin-arm64/build/Release/keytar.node $A/bin/mac/arm64/cli-2.5.0/envkey-keytar.node
cp $O/darwin-arm64/envkey-source $A/bin/mac/arm64/envkey-source
cp $K/darwin-arm64/build/Release/keytar.node $A/node_modules/keytar/build/Release/keytar.node

# windows x64
A=$B/app-win; mkdir -p $A/bin/windows/cli-2.5.0
cp $R/envkey-cli-win-x64.exe $A/bin/windows/cli-2.5.0/envkey.exe
cp $K/win32-x64/build/Release/keytar.node $A/bin/windows/cli-2.5.0/envkey-keytar.node
cp $O/windows-amd64/envkey-source.exe $A/bin/windows/envkey-source.exe
cp $K/win32-x64/build/Release/keytar.node $A/node_modules/keytar/build/Release/keytar.node

# linux x64 (sem subpasta de arch, sem envkey-keytar.node)
A=$B/app-linux; mkdir -p $A/bin/linux/cli-2.5.0
cp $R/envkey-cli-linux-x64 $A/bin/linux/cli-2.5.0/envkey && chmod +x $A/bin/linux/cli-2.5.0/envkey
cp $O/linux-amd64/envkey-source $A/bin/linux/envkey-source
cp $K/linux-x64/build/Release/keytar.node $A/node_modules/keytar/build/Release/keytar.node
```

Observação: no mac fizemos originalmente o `npm i` do staging com
`npm_config_runtime=electron npm_config_target=13.2.1 npm_config_arch=arm64` (equivalente, já que o keytar é N-API); para
win/linux copiamos o `node_modules` e sobrescrevemos o `keytar.node` como acima.

### 7. Configs do electron-builder

Ícones: `icon.icns` vem de `/Applications/EnvKey.app/Contents/Resources/icon.icns` (ou forneça outro); para win/linux
gere o PNG (512x512).

```bash
mkdir -p $B/builder/resources
cp /Applications/EnvKey.app/Contents/Resources/icon.icns $B/builder/resources/icon.icns
sips -s format png $B/builder/resources/icon.icns --out $B/builder/resources/icon.png
```

Os três YAMLs compartilham: `appId: com.envkey.appv2`, `productName: EnvKey`, `electronVersion: 13.2.1`,
`asar: false` (o código usa `process.resourcesPath/app/...`), `npmRebuild: false`, `publish: null`,
`directories: { app: <staging>, buildResources: $B/builder/resources, output: <saída> }` (caminhos **absolutos**).

`$B/builder/electron-builder.yml` (mac):

```yaml
mac:
  category: public.app-category.developer-tools
  icon: icon.icns
  identity: null
  target: [{ target: dir, arch: arm64 }]
```

`$B/builder/electron-builder.win.yml`:

```yaml
win:
  icon: icon.png
  signAndEditExecutable: false
  target: [{ target: nsis, arch: x64 }, { target: zip, arch: x64 }]
```

`$B/builder/electron-builder.linux.yml`:

```yaml
linux:
  icon: icon.png
  category: Development
  executableName: envkey
  target: [{ target: AppImage, arch: x64 }, { target: tar.gz, arch: x64 }]
```

(`directories.app` = `$B/app-mac` / `$B/app-win` / `$B/app-linux`; `output` = `$B/release`, `$B/release-win`,
`$B/release-linux`.)

### 8. Gerar os apps

```bash
cd $REPO
node_modules/.bin/electron-builder --config $B/builder/electron-builder.yml        --mac dir --arm64
node_modules/.bin/electron-builder --config $B/builder/electron-builder.win.yml    --win --x64
node_modules/.bin/electron-builder --config $B/builder/electron-builder.linux.yml  --linux --x64

# mac: o Electron arm64 precisa de assinatura válida depois de modificado -> ad-hoc
cd $B/release/mac-arm64 && codesign --force --deep -s - EnvKey.app && codesign --verify --deep --strict EnvKey.app
```

Saídas: `$B/release/mac-arm64/EnvKey.app`, `$B/release-win/{EnvKey Setup 2.4.18.exe,EnvKey-2.4.18-win.zip}`,
`$B/release-linux/{EnvKey-2.4.18.AppImage,envkey-2.4.18.tar.gz}`. O `electron-builder` baixa Electron 13.2.1, NSIS e
AppImage (GitHub) para `~/Library/Caches/electron` e `~/Library/Caches/electron-builder`.

### 9. Juntar e zipar

```
dist/
  desktop/       EnvKey-mac-arm64.app, "EnvKey Setup 2.4.18.exe", EnvKey-2.4.18-win.zip, EnvKey-2.4.18.AppImage, envkey-2.4.18.tar.gz
  cli/           envkey-2.5.0-{darwin-arm64,darwin-amd64,linux-amd64,linux-arm64,windows-amd64.exe} + envkey-keytar-{darwin-arm64,darwin-amd64,windows-amd64}.node
  envkey-source/ envkey-source-2.4.3-{darwin-arm64,darwin-amd64,linux-amd64,linux-arm64,windows-amd64.exe}
```

```bash
cd $B && ditto -c -k --norsrc --noextattr --keepParent dist ~/Documents/envkey-artifacts-2.4.18.zip
unzip -tq ~/Documents/envkey-artifacts-2.4.18.zip
```

`--norsrc --noextattr` evita os arquivos `._*` (AppleDouble) que atrapalham em Windows/Linux. Ao abrir o `.app` baixado
num Mac, se o Gatekeeper bloquear: `xattr -cr EnvKey-mac-arm64.app`.

## O que NÃO foi verificado

- **Nenhum binário de Windows ou Linux foi executado** (CLI, envkey-source ou app).
- O app mac **novo não foi aberto** (conflito de porta com o EnvKey instalado). Só se testou `envkey --version` e um
  `core start` do CLI; **login e gravação no Keychain não foram testados** (primeiro lugar a olhar se o login falhar).
- **Windows:** o CLI procura arquivos em `/snapshot/...`; no Windows isso deve virar `C:\snapshot\...` e só funciona se o
  app rodar a partir do drive C:. Não confirmado.
- Não foi gerado app desktop para **mac x64**, **linux arm64** nem **windows arm64** (o CLI de mac x64 e linux arm64 foi).
- **Sem assinatura / notarização.** Mac: ad-hoc; Windows: SmartScreen vai avisar.
- O `keytar` do `public/app/package-lock.json` é 7.7.0 (é o que o setup baixa); o `npm i` do staging resolve 7.9.0 para o
  JS. Funcionou assim (N-API), mas se for mexer, mantenha as versões alinhadas.
