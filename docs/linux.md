# Orbe no Linux

O Orbe usa Electron, React e SQLite, então a interface e a biblioteca podem ser compartilhadas com Windows. A base de portabilidade está preparada; o build e a validação real de Linux devem ser executados em Linux. Não há validação de execução Linux neste ambiente Windows.

## O que já foi preparado

- Resolução de yt-dlp, FFmpeg e FFprobe por plataforma. No Windows continuam os executáveis empacotados; no Linux são usadas ferramentas nativas no PATH ou definidas por `ORBE_YT_DLP_PATH`, `ORBE_FFMPEG_PATH` e `ORBE_FFPROBE_PATH`.
- Ícone PNG para janela, menu e empacotamento Linux.
- Alvos AppImage e `.deb`, inicialmente em x64. O instalador Windows continua independente.
- Workflow `.github/workflows/linux.yml`, com Node 24 e Ubuntu 24.04, testes unitários, testes de recuperação de pastas/vídeo em Xvfb, empacotamento e teste de abertura do aplicativo empacotado. Os arquivos são anexados como artefatos do Actions; o workflow não publica uma release automaticamente.
- Workflow `.github/workflows/arch.yml`, com um container oficial Arch e compilação com `makepkg` por um usuário sem privilégios. Gera o pacote pacman x86_64 e seu SHA-256, disponíveis como artefato do Actions. Pode ser iniciado pelo navegador, inclusive no Windows.

## Gerar em Ubuntu 24.04 x64

Instale Node.js 24 e npm. Instale as ferramentas nativas:

```bash
sudo apt-get update
sudo apt-get install -y ffmpeg yt-dlp libgtk-3-0t64 libnss3 libasound2t64 libgbm1 libxss1 libsecret-1-0
npm ci --ignore-scripts
node node_modules/electron/install.js
npm run tools:check
npm run dev
```

Para gerar os pacotes:

```bash
npm run dist:linux
```

O AppImage fica em `dist/`. Para gerar também um `.deb` localmente, defina `ORBE_PROJECT_URL` com a URL HTTPS real do projeto e `ORBE_MAINTAINER` com seu nome e contato antes de executar o comando. No GitHub Actions, a URL do repositório é usada automaticamente. Instale o `.deb` com `sudo apt install ./dist/<nome-do-pacote>.deb`, ou dê permissão de execução ao AppImage e abra-o. Esta primeira configuração Linux usa **FFmpeg, FFprobe e yt-dlp do sistema**; o AppImage não os inclui. O `.deb` declara as dependências. Se a distribuição fornecer um yt-dlp antigo, use uma versão atual do projeto e aponte `ORBE_YT_DLP_PATH` para ela.

Os nomes dos pacotes de sistema variam entre distribuições. Estes comandos descrevem Ubuntu 24.04; não são uma promessa de compatibilidade com todas as distribuições ou ARM.

## Arch Linux (e derivados: CachyOS, EndeavourOS, Manjaro)

`packaging/arch/PKGBUILD` compila a tag da versão a partir do código-fonte e gera um pacote pacman nativo, sem AppImage. Ele instala o aplicativo em `/opt/orbe`, o comando `orbe`, o ícone e o atalho do menu, e declara `ffmpeg` e `yt-dlp` do repositório oficial como dependências.

```bash
sudo pacman -Syu --needed base-devel git
git clone https://github.com/guilhermeSDB/Orbe.git
cd Orbe
cd packaging/arch
makepkg -s
```

Execute em Arch Linux x86_64 atualizado, com um usuário comum (não use `sudo makepkg`). Se você já clonou o projeto, use uma revisão que contenha `packaging/arch` e execute os dois últimos comandos. O `-s` instala as dependências necessárias e gera o pacote sem instalar o Orbe. A compilação precisa de internet para baixar o código da tag e as dependências npm/Electron. O PKGBUILD usa a tag `v0.2.1`, independentemente da branch local; alterações locais no aplicativo não entram nesse pacote.

Com a configuração padrão do `makepkg`, o arquivo gerado será `packaging/arch/orbe-0.2.1-1-x86_64.pkg.tar.zst`. Se você personalizou `PKGDEST` ou `PKGEXT`, o diretório ou a extensão podem mudar; `makepkg --packagelist` mostra o caminho exato. Para instalar e testar:

```bash
sudo pacman -U ./orbe-0.2.1-1-x86_64.pkg.tar.zst
orbe
```

O pacman instala também as dependências de execução declaradas, incluindo FFmpeg e yt-dlp. Para compilar e instalar em um único passo, use `makepkg -si`. Para remover: `sudo pacman -R orbe`.

### Anexar à release e habilitar no site

1. Teste o pacote no Arch antes de distribuí-lo. O suporte a derivados depende também das versões das bibliotecas disponíveis em cada distribuição.
2. No GitHub, abra a release `v0.2.1`, clique em **Edit** e anexe o arquivo `orbe-0.2.1-1-x86_64.pkg.tar.zst` à área de assets. Salve a release. O arquivo a distribuir é o pacote binário, não o `PKGBUILD`, `.SRCINFO` ou um ZIP do código-fonte.
3. Na Vercel, configure a URL pública do asset e faça um novo deploy:

   ```dotenv
   VITE_DOWNLOAD_LINUX_ARCH_URL=https://github.com/guilhermeSDB/orbe/releases/download/v0.2.1/orbe-0.2.1-1-x86_64.pkg.tar.zst
   ```

Sem essa variável, o site mostra **Download indisponível** apenas para esse formato. O workflow **Linux build** gera AppImage e DEB; o workflow **Arch Linux package** gera o pacote Arch. Os comandos de compilação local acima exigem um ambiente Arch Linux, mas o workflow Arch executa esse ambiente na nuvem e pode ser iniciado pelo Windows.

Ao lançar uma nova versão, publique a tag `v<versão>`, atualize `pkgver` no PKGBUILD, volte `pkgrel` para `1` e regenere o `.SRCINFO` com `makepkg --printsrcinfo > .SRCINFO`. Para revisar só o empacotamento da mesma versão, incremente `pkgrel`. Atualize também o nome do asset e a URL no site. Os arquivos `PKGBUILD`, `.SRCINFO` e `orbe.desktop` podem ser publicados no AUR separadamente; sua presença neste repositório não significa que o pacote já esteja no AUR.

Referências: [makepkg](https://man.archlinux.org/man/makepkg.8) e [pacman](https://man.archlinux.org/man/pacman.8.en).

## Gerar pelo GitHub Actions

Depois de enviar o repositório ao GitHub, abra **Actions → Linux build → Run workflow**. Baixe `orbe-linux-x64` em Artifacts. Tags `v*` também disparam a compilação. O workflow precisa passar antes de anunciar Linux como disponível na landing page.

O script de empacotamento passa `--publish never` ao electron-builder para impedir a publicação automática ao detectar o CI ou uma tag. Esse fluxo não precisa de `GH_TOKEN`: os instaladores são disponibilizados em Artifacts e podem ser anexados manualmente a uma release.

### Pacote Arch pelo navegador, inclusive no Windows

1. Envie as alterações para o GitHub, incluindo `.github/workflows/arch.yml`, `.gitattributes` e os arquivos de `packaging/arch`. O workflow com `workflow_dispatch` precisa estar na branch padrão do repositório para aparecer com o botão de execução manual.
2. Abra **Actions → Arch Linux package → Run workflow**. Escolha a branch que contém o empacotamento atualizado e clique em **Run workflow**.
3. Aguarde o job **Build pacman package (x86_64)** terminar. Ele atualiza um container Arch, confere se `.SRCINFO` corresponde ao `PKGBUILD` e executa `makepkg` com um usuário comum. Não é preciso instalar Arch ou Docker no seu computador nem configurar um token pessoal.
4. Abra a execução concluída e, em **Artifacts**, baixe **orbe-arch-x86_64**. Extraia o ZIP baixado: ele contém `orbe-0.2.1-1-x86_64.pkg.tar.zst` e `SHA256SUMS`.
5. Anexe o `.pkg.tar.zst` à release `v0.2.1`; você também pode anexar `SHA256SUMS`. Use a URL pública do `.pkg.tar.zst` em `VITE_DOWNLOAD_LINUX_ARCH_URL` e faça um novo deploy do site. Não use a URL temporária do artefato do Actions como link público de download.

O workflow também roda ao enviar uma **nova** tag `v*`. Antes de criar a tag, inclua o workflow, a atualização de `pkgver` e o `.SRCINFO` correspondente no commit. A compilação falha se a tag enviada não corresponder a `v<pkgver>`. A tag existente `v0.2.1` não recebe o workflow retroativamente; para gerar essa versão agora, execute manualmente na branch atualizada.

Mesmo na execução manual, o código do aplicativo vem da tag indicada no `PKGBUILD`, atualmente `v0.2.1`, e não do conteúdo da branch escolhida. A branch fornece a receita de empacotamento. O workflow inspeciona o pacote gerado, mas não executa testes gráficos no Arch nem publica a release automaticamente.

Referências: [execução manual de workflows](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/manually-run-a-workflow?tool=webui), [jobs em containers](https://docs.github.com/en/actions/how-tos/write-workflows/choose-where-workflows-run/run-jobs-in-a-container) e [imagens oficiais do Arch](https://github.com/archlinux/archlinux-docker).

## Verificações antes de distribuir

Teste importar arquivos e várias pastas, reprodução de áudio e vídeo, capa ao recolher o vídeo, conversão, downloads, recuperação de arquivos movidos/excluídos, temas e persistência após reiniciar. Confirme também Wayland/X11, áudio, permissões de unidades externas, sessão de navegador, integração com Discord e o cofre de credenciais do desktop. Contas devem ser conectadas novamente ao migrar de Windows: não copie tokens criptografados entre sistemas.

Para mover uma coleção do Windows, copie os arquivos pessoais e use **Pastas de mídia → Localizar pasta** para atualizar seus caminhos na biblioteca. Caminhos como `C:\Music` não são caminhos Linux.

Fontes primárias: [alvos Linux do electron-builder](https://www.electron.build/linux/), [arquivos nativos do yt-dlp](https://github.com/yt-dlp/yt-dlp#release-files), [armazenamento de credenciais no Electron](https://www.electronjs.org/docs/latest/api/safe-storage).
