# Setup Desktop Moderno — Debian 13 Trixie + Hyprland
**Lorenzo Bernardini — Build orientata a estetica, performance e compatibilità UniBO**

---

## Cos'è e perché questa scelta

**Hyprland** è un compositor Wayland moderno: gestisce le finestre, gli effetti grafici e l'input in modo nativo su Wayland (il protocollo grafico che ha sostituito X11 sui sistemi Linux moderni). Non è un desktop environment completo come GNOME — è uno strato compositing altamente configurabile attorno al quale si costruisce il proprio ambiente su misura.

Il risultato visivo quando configurato: animazioni fluide delle finestre, blur dinamico degli sfondi, angoli arrotondati, gradienti di colore, tiling automatico (le finestre si dispongono da sole senza sovrapporsi), barra di stato completamente personalizzata. È il setup che si trova su [r/unixporn](https://www.reddit.com/r/unixporn) e che i CS student usano come firma identitaria.

**Debian 13 "Trixie" (Testing)** invece di Debian 12 Stable: Hyprland richiede librerie di sistema più recenti di quelle presenti in Debian 12. Trixie è il ramo "testing" di Debian — più aggiornato di Stable ma molto più stabile di Arch o Sid. Per un laptop universitario è la scelta corretta: pacchetti recenti, base Debian garantita, compatibilità totale con gli strumenti del laboratorio Ercolani.

**Schema cromatico: Catppuccin Mocha** — la palette dark più diffusa nella community Linux al momento, con tonalità lavanda/rosé su sfondo scuro profondo. Coerente su tutti gli strumenti (terminale, editor, browser, status bar, notifiche).

---

## Indice

- [FASE 1 — Installazione Debian 13 Trixie (base, senza desktop)](#fase-1)
- [FASE 2 — Configurazione sistema di base](#fase-2)
- [FASE 3 — Installazione Hyprland e ecosistema](#fase-3)
- [FASE 4 — Theming Catppuccin e configurazione visiva](#fase-4)
- [FASE 5 — Strumenti universitari](#fase-5)
- [FASE 6 — Dotfiles e versionamento della configurazione](#fase-6)
- [Riepilogo stack completo](#riepilogo)

---

<a name="fase-1"></a>
## FASE 1 — Installazione Debian 13 Trixie (base)

### 1.1 — Download ISO Debian Trixie

Debian Testing non ha release ISO stabili periodiche come Stable. Usare le immagini "weekly build":

**Download ISO Trixie (netinstall — richiede internet):**
> https://cdimage.debian.org/cdimage/trixie_di_rc1/amd64/iso-cd/

Scaricare `debian-trixie-DI-rc1-amd64-netinst.iso` (circa 700 MB).

**Alternativa: immagine live con installer grafico:**
> https://cdimage.debian.org/cdimage/weekly-live-builds/amd64/iso-hybrid/

Cercare `debian-live-trixie-*-gnome.iso` — include GNOME per una sessione live di prova prima di installare.

### 1.2 — Creazione chiavetta USB

Identico a quanto descritto nella guida precedente: Rufus, modalità ISO, schema GPT, UEFI.

### 1.3 — Installer: scegliere "No desktop environment"

Durante l'installazione, alla schermata **"Software selection"**, **deselezionare tutto** tranne:

- ✅ **SSH server**
- ✅ **Standard system utilities**

Non installare nessun desktop environment dall'installer. Hyprland verrà installato manualmente nella Fase 3. Questo garantisce un sistema base pulito, senza componenti GNOME o KDE che rimarrebbero come dipendenze orfane.

Per il resto, seguire la guida precedente (Fase 3.1 → 3.8) identicamente.

---

<a name="fase-2"></a>
## FASE 2 — Configurazione Sistema di Base

Al primo avvio si presenterà una shell testuale (nessuna GUI). È normale.

### 2.1 — Aggiornare il sistema

```bash
su -
apt update && apt full-upgrade -y
```

### 2.2 — Installare sudo e configurare l'utente

```bash
apt install sudo -y
usermod -aG sudo lorenzo
# Logout e login per applicare
exit
```

### 2.3 — Repository non-free e contrib

```bash
sudo nano /etc/apt/sources.list
```

Sostituire il contenuto con:

```
deb http://deb.debian.org/debian trixie main contrib non-free non-free-firmware
deb http://deb.debian.org/debian trixie-updates main contrib non-free non-free-firmware
deb http://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware
```

```bash
sudo apt update
sudo apt install firmware-linux firmware-linux-nonfree -y
```

### 2.4 — Abilitare i sorgenti per compilazione (necessario per Hyprland)

```bash
sudo nano /etc/apt/sources.list
```

Aggiungere le righe `deb-src` corrispondenti:

```
deb-src http://deb.debian.org/debian trixie main contrib non-free non-free-firmware
```

```bash
sudo apt update
```

### 2.5 — Strumenti base necessari

```bash
sudo apt install -y \
  git curl wget \
  build-essential cmake meson ninja-build \
  pkg-config \
  python3 python3-pip \
  pipewire pipewire-pulse wireplumber \
  xdg-user-dirs
```

### 2.6 — Installare un display manager (schermata di login grafica)

**SDDM** è il display manager più compatibile con Hyprland:

```bash
sudo apt install sddm -y
sudo systemctl enable sddm
```

---

<a name="fase-3"></a>
## FASE 3 — Installazione Hyprland e Ecosistema

### Metodo A — Installer automatizzato JaKooLit (RACCOMANDATO per il primo setup)

Questo script automatizza l'intero processo: compila Hyprland, installa tutti i componenti dell'ecosistema e applica una configurazione di default funzionante e già esteticamente rifinita.

```bash
# Clonare il repo
git clone https://github.com/JaKooLit/Debian-Hyprland.git ~/Debian-Hyprland
cd ~/Debian-Hyprland

# Rendere eseguibile e avviare
chmod +x install.sh
./install.sh
```

Lo script è interattivo: chiederà cosa installare (rispondere Yes a tutto nella prima run). Il processo richiede **30–60 minuti** perché compila Hyprland dal sorgente.

**Link repository:**
> https://github.com/JaKooLit/Debian-Hyprland

### Metodo B — Installazione manuale dei pacchetti (Trixie ha Hyprland nei repo ufficiali)

Debian Trixie include Hyprland nei propri repository ufficiali. L'installazione manuale è più semplice ma la versione potrebbe essere leggermente più vecchia rispetto alla compilazione da sorgente:

```bash
sudo apt install -y \
  hyprland \
  xdg-desktop-portal-hyprland \
  hyprpaper \
  hyprlock \
  hypridle \
  waybar \
  rofi-wayland \
  dunst \
  kitty \
  swaylock \
  grim slurp \
  wl-clipboard \
  brightnessctl \
  playerctl \
  pamixer \
  thunar \
  network-manager-gnome \
  polkit-kde-agent-1
```

### 3.1 — Componenti dell'Ecosistema Hyprland

| Componente | Strumento scelto | Funzione |
|---|---|---|
| **Compositor** | Hyprland | Gestisce finestre, effetti, tiling |
| **Status bar** | Waybar | Barra superiore/inferiore custom |
| **Terminal** | Kitty | Terminale GPU-accelerato |
| **Shell** | Zsh + Starship | Shell + prompt interattivo |
| **App launcher** | Rofi (Wayland) | Lanciatore applicazioni |
| **Notifiche** | Dunst | Notifiche desktop |
| **Wallpaper** | Hyprpaper / swww | Gestione sfondi (swww per animazioni) |
| **Lock screen** | Hyprlock | Schermata blocco |
| **Screenshot** | Grim + Slurp | Screenshot area/schermo intero |
| **File manager** | Thunar (GUI) | Gestione file grafica |
| **File manager** | lf / ranger | Gestione file da terminale |
| **Browser** | Firefox | Browser principale |
| **Editor** | Neovim + LazyVim | Editor da terminale |
| **IDE** | VSCode (code-oss) | Per i corsi di programmazione |

### 3.2 — Installare Zsh e Starship

```bash
sudo apt install zsh -y
chsh -s $(which zsh)   # impostare Zsh come shell default

# Installare Starship (prompt interattivo con icone Git, linguaggio corrente, ecc.)
curl -sS https://starship.rs/install.sh | sh
echo 'eval "$(starship init zsh)"' >> ~/.zshrc
```

**Link Starship:**
> https://starship.rs

### 3.3 — Installare lf (file manager terminale)

```bash
# lf non è in apt, scaricare il binario precompilato
wget https://github.com/gokcehan/lf/releases/latest/download/lf-linux-amd64.tar.gz
tar -xzf lf-linux-amd64.tar.gz
sudo mv lf /usr/local/bin/
```

### 3.4 — Installare Neovim con LazyVim

```bash
# Versione recente di Neovim non è in apt Trixie, installare AppImage
wget https://github.com/neovim/neovim/releases/latest/download/nvim-linux-x86_64.appimage
chmod +x nvim-linux-x86_64.appimage
sudo mv nvim-linux-x86_64.appimage /usr/local/bin/nvim

# LazyVim (distribuzione Neovim preconfigurata con LSP, syntax highlight, ecc.)
git clone https://github.com/LazyVim/starter ~/.config/nvim
```

**Link LazyVim:**
> https://www.lazyvim.org

Al primo avvio di `nvim`, LazyVim installerà automaticamente tutti i plugin.

---

<a name="fase-4"></a>
## FASE 4 — Theming Catppuccin e Configurazione Visiva

### 4.1 — Struttura dei file di configurazione Hyprland

Tutti i file di configurazione di Hyprland si trovano in `~/.config/hypr/`. La struttura consigliata:

```
~/.config/hypr/
├── hyprland.conf          # configurazione principale
├── hyprpaper.conf         # wallpaper
├── hyprlock.conf          # lock screen
├── hypridle.conf          # timeout/sleep
└── modules/
    ├── keybinds.conf      # scorciatoie tastiera
    ├── animations.conf    # animazioni finestre
    ├── decoration.conf    # blur, angoli, ombre
    ├── monitors.conf      # configurazione monitor
    └── windowrules.conf   # regole per le finestre
```

### 4.2 — Configurazione di base Hyprland (`~/.config/hypr/hyprland.conf`)

```conf
# Importare i moduli
source = ~/.config/hypr/modules/monitors.conf
source = ~/.config/hypr/modules/keybinds.conf
source = ~/.config/hypr/modules/animations.conf
source = ~/.config/hypr/modules/decoration.conf

# Variabili
$terminal = kitty
$browser = firefox
$fileManager = thunar
$launcher = rofi -show drun

# Avvio automatico
exec-once = hyprpaper
exec-once = waybar
exec-once = dunst
exec-once = hypridle
exec-once = /usr/lib/polkit-kde-authentication-agent-1

# Impostazioni generali
general {
    gaps_in = 5
    gaps_out = 10
    border_size = 2
    col.active_border = rgba(cba6f7ff) rgba(89b4faff) 45deg  # Catppuccin lavender + blue
    col.inactive_border = rgba(595959aa)
    layout = dwindle
}

# Tiling layout
dwindle {
    pseudotile = true
    preserve_split = true
}
```

### 4.3 — Decorazioni e effetti (`~/.config/hypr/modules/decoration.conf`)

```conf
decoration {
    rounding = 10                    # angoli arrotondati

    blur {
        enabled = true
        size = 8
        passes = 3
        new_optimizations = true
        xray = false
        noise = 0.0117
        contrast = 0.8917
        brightness = 0.8172
        vibrancy = 0.1696
    }

    drop_shadow = true
    shadow_range = 20
    shadow_render_power = 3
    col.shadow = rgba(1a1a1aee)

    active_opacity = 1.0
    inactive_opacity = 0.95
    fullscreen_opacity = 1.0
}
```

### 4.4 — Animazioni (`~/.config/hypr/modules/animations.conf`)

```conf
animations {
    enabled = true

    bezier = myBezier, 0.05, 0.9, 0.1, 1.05
    bezier = overshot, 0.05, 0.9, 0.1, 1.1
    bezier = smoothOut, 0.5, 0, 0.99, 0.99
    bezier = smoothIn, 0.5, -0.5, 0.68, 1.5

    animation = windows, 1, 5, myBezier
    animation = windowsOut, 1, 4, smoothOut, popin 80%
    animation = windowsMove, 1, 4, smoothIn, slide
    animation = border, 1, 10, default
    animation = borderangle, 1, 8, default
    animation = fade, 1, 7, smoothOut
    animation = workspaces, 1, 6, overshot, slide
}
```

### 4.5 — Catppuccin per Kitty

```bash
mkdir -p ~/.config/kitty
curl -o ~/.config/kitty/catppuccin-mocha.conf \
  https://raw.githubusercontent.com/catppuccin/kitty/main/themes/mocha.conf
```

In `~/.config/kitty/kitty.conf`:
```conf
include catppuccin-mocha.conf

font_family      JetBrainsMono Nerd Font
font_size        12.0
bold_font        auto
italic_font      auto

window_padding_width 10
background_opacity 0.92
```

### 4.6 — Nerd Fonts (necessari per icone nella shell e status bar)

```bash
# Scaricare JetBrainsMono Nerd Font (la più usata per coding)
wget https://github.com/ryanoasis/nerd-fonts/releases/latest/download/JetBrainsMono.zip
mkdir -p ~/.local/share/fonts
unzip JetBrainsMono.zip -d ~/.local/share/fonts/JetBrainsMono
fc-cache -fv
```

**Sito ufficiale Nerd Fonts:**
> https://www.nerdfonts.com

### 4.7 — Catppuccin per Waybar

Il repository ufficiale Catppuccin fornisce temi CSS per Waybar:

```bash
git clone https://github.com/catppuccin/waybar.git /tmp/catppuccin-waybar
mkdir -p ~/.config/waybar
cp /tmp/catppuccin-waybar/themes/mocha.css ~/.config/waybar/catppuccin.css
```

In `~/.config/waybar/style.css`, aggiungere all'inizio:
```css
@import "catppuccin.css";
```

**Repository Catppuccin ufficiale (tutti gli strumenti):**
> https://github.com/catppuccin/catppuccin

### 4.8 — Sfondo e Lock Screen

**Hyprpaper** (`~/.config/hypr/hyprpaper.conf`):
```conf
preload = ~/Pictures/wallpaper.jpg
wallpaper = ,~/Pictures/wallpaper.jpg
```

Per sfondi Catppuccin-compatibili:
> https://github.com/zhichaoh/catppuccin-wallpapers

**Hyprlock** (`~/.config/hypr/hyprlock.conf`):
```conf
background {
    monitor =
    path = ~/Pictures/wallpaper.jpg
    blur_passes = 3
    blur_size = 7
    noise = 0.0117
    contrast = 0.8917
    brightness = 0.8172
    vibrancy = 0.1696
}

input-field {
    size = 300, 50
    outline_thickness = 3
    outer_color = rgb(cba6f7)
    inner_color = rgb(1e1e2e)
    font_color = rgb(cdd6f4)
    fade_on_empty = false
    placeholder_text = <span foreground="##cdd6f4">Inserire password...</span>
    rounding = 15
}
```

---

<a name="fase-5"></a>
## FASE 5 — Strumenti Universitari

### 5.1 — VirtualBox su Debian Trixie

```bash
# Repository Oracle per Trixie
wget -O- https://www.virtualbox.org/download/oracle_vbox_2016.asc | \
  sudo gpg --dearmor -o /usr/share/keyrings/oracle-virtualbox.gpg

echo "deb [arch=amd64 signed-by=/usr/share/keyrings/oracle-virtualbox.gpg] \
  https://download.virtualbox.org/virtualbox/debian trixie contrib" | \
  sudo tee /etc/apt/sources.list.d/oracle-virtualbox.list

# Nota: se trixie non è disponibile come repo Oracle, usare bookworm (compatibile)
# Modificare "trixie" con "bookworm" nella riga sopra se necessario

sudo apt update && sudo apt install virtualbox-7.1 -y
sudo usermod -aG vboxusers $USER
```

### 5.2 — Vagrant, KVM, strumenti di sviluppo

Identici alla guida precedente (Fase 5.3, 5.4, 5.1). I comandi apt funzionano identicamente su Trixie.

### 5.3 — VSCode (code-oss) con tema Catppuccin

```bash
# Installare code-oss (build open source di VSCode)
sudo apt install code -y

# Oppure scaricando il .deb ufficiale Microsoft
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | \
  sudo gpg --dearmor -o /usr/share/keyrings/microsoft.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/microsoft.gpg] \
  https://packages.microsoft.com/repos/code stable main" | \
  sudo tee /etc/apt/sources.list.d/vscode.list
sudo apt update && sudo apt install code -y
```

Una volta installato, cercare l'estensione **"Catppuccin for VSCode"** nel marketplace.

---

<a name="fase-6"></a>
## FASE 6 — Dotfiles: Versionare la Propria Configurazione

Un aspetto fondamentale del workflow da CS student è mantenere i propri file di configurazione ("dotfiles") sotto version control con Git. Permette di ripristinare il proprio ambiente esatto su qualsiasi macchina in pochi minuti.

### Schema consigliato con bare repository

```bash
# Inizializzare il bare repo dei dotfiles
git init --bare $HOME/.dotfiles

# Creare alias (aggiungere a ~/.zshrc)
echo 'alias dotfiles="/usr/bin/git --git-dir=$HOME/.dotfiles/ --work-tree=$HOME"' >> ~/.zshrc
source ~/.zshrc

# Non mostrare i file non tracciati (evita rumore)
dotfiles config --local status.showUntrackedFiles no

# Aggiungere file di configurazione
dotfiles add ~/.config/hypr/hyprland.conf
dotfiles add ~/.config/waybar/config
dotfiles add ~/.config/kitty/kitty.conf
dotfiles add ~/.zshrc
dotfiles commit -m "initial dotfiles setup"

# Pubblicare su GitHub (account personale)
dotfiles remote add origin https://github.com/tuonome/dotfiles.git
dotfiles push -u origin main
```

In futuro, su qualsiasi altra macchina Linux, ripristinare l'intero ambiente con:
```bash
git clone --bare https://github.com/tuonome/dotfiles.git $HOME/.dotfiles
alias dotfiles="/usr/bin/git --git-dir=$HOME/.dotfiles/ --work-tree=$HOME"
dotfiles checkout
```

---

<a name="riepilogo"></a>
## Riepilogo Stack Completo

| Layer | Strumento | Note |
|---|---|---|
| **OS base** | Debian 13 Trixie (Testing) | Più aggiornato di Stable, Hyprland-compatible |
| **Display server** | Wayland | Nativo, nessun X11 |
| **Compositor** | Hyprland | Tiling, animazioni, effetti Wayland |
| **Display manager** | SDDM | Login grafico |
| **Status bar** | Waybar | Custom, moduli configurabili |
| **Terminal** | Kitty | GPU-accelerato, tabs, splits |
| **Shell** | Zsh + Starship | Prompt con icone, info Git, linguaggio |
| **Editor TUI** | Neovim + LazyVim | LSP, syntax, completamento |
| **IDE grafico** | VSCode | Per i laboratori UniBO |
| **Launcher** | Rofi (Wayland) | Ricerca applicazioni |
| **Notifiche** | Dunst | Notifiche desktop |
| **Wallpaper** | Hyprpaper | Gestione sfondi |
| **Lock screen** | Hyprlock | Con blur e Catppuccin |
| **Screenshot** | Grim + Slurp | Area e fullscreen |
| **File manager GUI** | Thunar | Per operazioni visive |
| **File manager TUI** | lf | Gestione da terminale |
| **Schema colori** | Catppuccin Mocha | Unifica terminale, editor, bar, browser |
| **Font** | JetBrainsMono Nerd Font | Icone + monospace coding |
| **Hypervisor** | VirtualBox + KVM | Corsi UniBO + performance |
| **Provisioning VM** | Vagrant | Amministrazione di Sistemi T |
| **Audio** | PipeWire + WirePlumber | Moderno, bassa latenza |
| **Dotfiles** | Git bare repo | Config versionata e portabile |

## Link di Riferimento

| Risorsa | URL |
|---|---|
| Debian Trixie ISO | https://cdimage.debian.org/cdimage/weekly-live-builds/amd64/iso-hybrid/ |
| Hyprland Wiki ufficiale | https://wiki.hypr.land |
| JaKooLit Debian-Hyprland | https://github.com/JaKooLit/Debian-Hyprland |
| Catppuccin (tutti i temi) | https://github.com/catppuccin/catppuccin |
| Catppuccin Wallpapers | https://github.com/zhichaoh/catppuccin-wallpapers |
| Nerd Fonts | https://www.nerdfonts.com |
| Starship prompt | https://starship.rs |
| LazyVim | https://www.lazyvim.org |
| r/unixporn (ispirazione) | https://www.reddit.com/r/unixporn |
| r/hyprland | https://www.reddit.com/r/hyprland |

---

## Nota Finale — Curva di Apprendimento

Hyprland non è un desktop che si usa passivamente: si configura, si rompe, si capisce come funziona e si aggiusta. Ogni riga del file `.conf` insegna qualcosa sul funzionamento del sistema. Questa è la ragione per cui è la scelta giusta per un corso di Ingegneria Informatica: il processo di configurazione è esso stesso studio.

Il punto di partenza consigliato è l'installer JaKooLit che fornisce una configurazione funzionante; da lì si procede per modifiche incrementali, leggendo la wiki di Hyprland per ogni parametro che si vuole cambiare.

---

*Documento prodotto sulla base dell'analisi dell'ecosistema Hyprland/Debian — Aprile 2026*
