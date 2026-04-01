# Guida Completa — Reinstallazione Laptop per UniBO
**Lorenzo Bernardini — Debian 12 "Bookworm" su hardware i7 / 16 GB RAM**

> **Premessa**: questa guida parte dall'assunzione che i dati personali presenti sul disco non debbano essere conservati. L'unica cosa da salvare prima di procedere è la licenza Windows. Tutto il resto verrà sovrascritto definitivamente.

---

## INDICE RAPIDO

- [FASE 0 — Salvataggio licenza Windows (5 minuti)](#fase-0)
- [FASE 1 — Download dei file necessari (30–60 minuti)](#fase-1)
- [FASE 2 — Creazione chiavetta USB avviabile (10 minuti)](#fase-2)
- [FASE 3 — Installazione Debian (30–45 minuti)](#fase-3)
- [FASE 4 — Configurazione post-installazione (20–30 minuti)](#fase-4)
- [FASE 5 — Installazione strumenti universitari (20 minuti)](#fase-5)
- [FASE 6 — Setup macchine virtuali](#fase-6)

---

<a name="fase-0"></a>
## FASE 0 — Salvataggio Licenza Windows (da eseguire PRIMA di qualsiasi altra operazione)

### Passo 0.1 — Estrarre il Product Key

Aprire **PowerShell come Amministratore** (tasto destro sul menu Start → "Windows PowerShell (amministratore)") e digitare:

```powershell
(Get-WmiObject -query 'select * from SoftwareLicensingService').OA3xOriginalProductKey
```

Apparirà una stringa nel formato `XXXXX-XXXXX-XXXXX-XXXXX-XXXXX`. **Annotarla su carta o su un dispositivo esterno** (telefono, altro PC, account cloud personale).

### Passo 0.2 — Verificare il tipo di licenza

```powershell
(Get-WmiObject -query 'select * from SoftwareLicensingService').OA3xOriginalProductKeyDescription
```

Se il risultato contiene la parola `OEM`, la licenza è embedded nel firmware UEFI del portatile: anche senza il key annotato, una futura reinstallazione di Windows sullo stesso hardware si attiverà automaticamente. Il key annotato sopra è comunque un'assicurazione ulteriore.

### Passo 0.3 — Annotare il modello esatto del laptop

Da PowerShell:
```powershell
Get-WmiObject -Class Win32_ComputerSystem | Select-Object Manufacturer, Model
```

Questo servirà per individuare i driver corretti in caso di necessità futura.

---

<a name="fase-1"></a>
## FASE 1 — Download dei File Necessari

Tutti i download seguenti vanno effettuati **su Windows, prima di formattare**, e salvati sulla chiavetta USB o su un disco esterno.

### 1.1 — Immagine ISO di Debian 12 "Bookworm"

Scaricare l'immagine **netinstall** (la versione minimale che scarica i pacchetti durante l'installazione, richiede connessione internet) oppure la **DVD ISO completa** (non richiede internet durante l'installazione, consigliata).

**Download DVD ISO completa (circa 3.7 GB) — RACCOMANDATO:**
```
https://cdimage.debian.org/debian-cd/current/amd64/iso-dvd/
```
Scaricare il file `debian-12.X.X-amd64-DVD-1.iso` (la versione più recente disponibile).

**Pagina ufficiale con tutti i link:**
> https://www.debian.org/distrib/

### 1.2 — Rufus (strumento per creare la chiavetta avviabile)

Rufus è lo strumento più semplice e affidabile per creare una chiavetta USB bootabile da Windows.

**Download diretto:**
> https://rufus.ie/it/

Scaricare `rufus-X.XX.exe` (versione standard, non portable). Non richiede installazione.

### 1.3 — (Opzionale ma consigliato) Ventoy

Alternativa a Rufus: permette di mettere più ISO sulla stessa chiavetta USB e scegliere quale avviare al boot. Utile se in futuro si vorrà tenere anche una Kali Linux live sulla chiavetta.

**Download:**
> https://www.ventoy.net/en/download.html

### Riepilogo file da avere pronti

| File | Dimensione approssimativa | Dove salvare |
|---|---|---|
| `debian-12.x.x-amd64-DVD-1.iso` | ~3.7 GB | Chiavetta USB o disco esterno |
| `rufus-x.xx.exe` | ~1.5 MB | Chiavetta USB o desktop |
| Product Key Windows (annotato) | — | Carta / dispositivo separato |

---

<a name="fase-2"></a>
## FASE 2 — Creazione Chiavetta USB Avviabile

**Materiale necessario**: chiavetta USB da almeno **8 GB** (il suo contenuto verrà completamente cancellato).

### Passo 2.1 — Aprire Rufus

Aprire `rufus-x.xx.exe` (non richiede installazione). Se Windows chiede i permessi di amministratore, concederli.

### Passo 2.2 — Configurazione in Rufus

1. **Device** (in alto): selezionare la propria chiavetta USB dall'elenco a discesa. Verificare che sia quella giusta controllando la dimensione indicata.

2. **Boot selection**: cliccare il pulsante **"SELECT"** e navigare fino al file `.iso` di Debian scaricato.

3. **Partition scheme**: selezionare **GPT** (necessario per i sistemi moderni con UEFI).

4. **Target system**: verrà impostato automaticamente su **UEFI (non CSM)** dopo aver scelto GPT. Lasciare così.

5. **File system**: lasciare **FAT32** (impostazione predefinita).

6. **Volume label**: può essere lasciato come suggerito automaticamente da Rufus.

7. Cliccare **START**.

8. Rufus potrebbe chiedere se scrivere in modalità **ISO** o **DD**: scegliere **modalità ISO** (consigliata per Debian).

9. Confermare l'avviso che tutti i dati sulla chiavetta verranno cancellati → **OK**.

Il processo richiede circa 5–10 minuti. Al termine, la chiavetta è pronta.

---

<a name="fase-3"></a>
## FASE 3 — Installazione di Debian 12

### Passo 3.1 — Accedere al BIOS/UEFI

Spegnere completamente il laptop. Inserire la chiavetta USB. Riaccendere il laptop e **premere immediatamente il tasto corretto** per accedere al menu di boot o al BIOS.

Il tasto varia a seconda del produttore del portatile:

| Marca comune | Tasto BIOS | Tasto Boot Menu |
|---|---|---|
| Lenovo | F1 o F2 | F12 |
| HP | F10 o Esc | F9 o Esc |
| Dell | F2 | F12 |
| ASUS | F2 o Del | F8 o Esc |
| Acer | F2 o Del | F12 |
| MSI | Del | F11 |

> Se non si conosce la marca/modello, provare F12, F11, F10, F2, Del durante l'avvio. L'accesso va tentato nelle primissime frazioni di secondo dopo l'accensione.

### Passo 3.2 — Impostazioni BIOS da verificare

Una volta nel BIOS/UEFI:

1. **Secure Boot → Disabled** (necessario per avviare Debian; può essere riabilitato dopo l'installazione se necessario, ma non è indispensabile).
2. **Boot order**: spostare "USB Drive" o "Removable Device" in prima posizione.
3. Salvare le modifiche (**Save & Exit**, solitamente F10).

### Passo 3.3 — Avvio del Installer Debian

Il laptop si avvierà dalla chiavetta e mostrerà un menu testuale (GRUB di Debian). Selezionare:

```
Graphical install
```

(con le frecce direzionali + Invio)

### Passo 3.4 — Percorso guidato dell'Installer

Seguire le schermate nell'ordine:

**a) Lingua**: selezionare **Italian** (Italiano).

**b) Posizione geografica**: selezionare **Italia**.

**c) Layout tastiera**: selezionare **Italiano** (o la variante specifica del proprio laptop).

**d) Configurazione di rete**:
- L'installer cercherà automaticamente una connessione ethernet o Wi-Fi.
- Se si usa Wi-Fi, selezionare la propria rete e inserire la password.
- Se non c'è connessione disponibile durante l'installazione (usando la DVD ISO), proseguire lo stesso: internet non è obbligatorio con la DVD ISO.

**e) Nome del computer (hostname)**: inserire un nome a piacere (es. `lorenzo-laptop`). Non contiene spazi.

**f) Nome dominio**: lasciare vuoto.

**g) Password di root**: inserire una password robusta per l'account amministratore di sistema. Annotarla. Questa è la password di `root` (il superutente).

**h) Nuovo utente**:
- Nome completo: es. `Lorenzo`
- Nome account: es. `lorenzo` (tutto minuscolo, senza spazi)
- Password: inserire la password per l'uso quotidiano.

### Passo 3.5 — Partizionamento del Disco (CRITICO)

Questa è la schermata più delicata. Selezionare:

```
Metodo di partizionamento: Guidato - usa l'intero disco
```

Poi selezionare il disco del laptop (sarà l'unico disco elencato, es. `sda` o `nvme0n1`).

Nella schermata successiva, scegliere:

```
Tutti i file in una partizione (consigliato per i nuovi utenti)
```

Verrà mostrato un riepilogo delle partizioni che verranno create. Cliccare **"Finish partitioning and write changes to disk"** e poi confermare con **"Yes"** quando chiede se si vogliono scrivere le modifiche.

> **Attenzione**: da questo momento il disco verrà formattato e tutti i dati precedenti cancellati permanentemente. Se si è sicuri di aver salvato la licenza Windows al passo 0, procedere.

### Passo 3.6 — Selezione Software

L'installer chiederà quali componenti installare. Mantenere selezionati (con la barra spaziatrice):

- ✅ **Ambiente desktop Debian** → poi scegliere **GNOME** (o XFCE se si preferisce la leggerezza)
- ✅ **Strumenti di sistema standard**
- ✅ **Server SSH** (consigliato: serve per l'accesso remoto al lab Ercolani)

Deselezionare tutto il resto (server di stampa, ecc.).

### Passo 3.7 — Installazione del Boot Loader GRUB

Quando chiesto, confermare l'installazione di GRUB sul disco principale. Selezionare il disco del laptop dall'elenco (es. `/dev/sda` o `/dev/nvme0n1`).

### Passo 3.8 — Completamento e Primo Avvio

Al termine dell'installazione, l'installer chiederà di rimuovere il supporto di installazione. Estrarre la chiavetta USB e cliccare **Continua**. Il sistema si riavvierà in Debian.

Al primo avvio comparirà la schermata di login grafica. Usare il nome utente e la password impostati al passo 3.4h.

---

<a name="fase-4"></a>
## FASE 4 — Configurazione Post-Installazione

Aprire il **Terminale** (su GNOME: tasto Super/Windows → cercare "Terminale").

### Passo 4.1 — Aggiornare l'intero sistema

```bash
su -
apt update && apt upgrade -y
```

> `su -` passa all'utente root. Inserire la password di root impostata durante l'installazione.

### Passo 4.2 — Installare sudo e aggiungere l'utente

```bash
apt install sudo -y
usermod -aG sudo lorenzo
```

(Sostituire `lorenzo` con il proprio nome utente se diverso.)

Effettuare il logout e il login per rendere effettiva l'appartenenza al gruppo sudo. D'ora in poi si può usare `sudo` invece di `su -`.

### Passo 4.3 — Installare i firmware non-free (driver hardware)

Molti portatili necessitano di firmware proprietari per Wi-Fi, Bluetooth e grafica. Aggiungere i repository non-free:

```bash
sudo nano /etc/apt/sources.list
```

Modificare la riga principale aggiungendo `non-free non-free-firmware` alla fine:

```
deb http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb http://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
deb http://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
```

Salvare (Ctrl+O, Invio, Ctrl+X) e aggiornare:

```bash
sudo apt update && sudo apt install firmware-linux firmware-linux-nonfree -y
sudo reboot
```

> Se dopo il riavvio il Wi-Fi non funziona, cercare il modello della scheda di rete con `lspci | grep -i wireless` e cercare il pacchetto firmware specifico (es. `firmware-iwlwifi` per schede Intel, `firmware-realtek` per Realtek).

### Passo 4.4 — Ottimizzazioni Performance

```bash
# Governor CPU (massima performance)
sudo apt install cpufrequtils -y
echo 'GOVERNOR="performance"' | sudo tee /etc/default/cpufrequtils
sudo systemctl restart cpufrequtils

# zram (swap compresso in RAM, evita accessi su disco)
sudo apt install zram-tools -y
sudo nano /etc/default/zramswap
# Impostare: ALGO=lz4 e PERCENT=25

# Ridurre la swappiness
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.d/99-performance.conf
sudo sysctl -p /etc/sysctl.d/99-performance.conf

# TLP per gestione termica e batteria
sudo apt install tlp tlp-rdw -y
sudo systemctl enable tlp
sudo tlp start
```

---

<a name="fase-5"></a>
## FASE 5 — Installazione Strumenti Universitari

### Passo 5.1 — Strumenti di sviluppo base

```bash
sudo apt install -y \
  build-essential gcc g++ gdb make cmake \
  git curl wget \
  python3 python3-pip python3-venv \
  vim neovim \
  openssh-client \
  nmap wireshark tcpdump netcat-openbsd \
  htop tree tmux
```

### Passo 5.2 — Oracle VirtualBox

VirtualBox non è presente nei repository Debian standard. Installarlo dal repository ufficiale Oracle:

**Aggiungere il repository Oracle:**
```bash
# Scaricare la chiave GPG ufficiale
wget -O- https://www.virtualbox.org/download/oracle_vbox_2016.asc | sudo gpg --dearmor -o /usr/share/keyrings/oracle-virtualbox.gpg

# Aggiungere il repository
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/oracle-virtualbox.gpg] https://download.virtualbox.org/virtualbox/debian bookworm contrib" | sudo tee /etc/apt/sources.list.d/oracle-virtualbox.list

# Installare
sudo apt update
sudo apt install virtualbox-7.1 -y
```

**Aggiungere l'utente al gruppo vboxusers:**
```bash
sudo usermod -aG vboxusers lorenzo
```

**Installare il VirtualBox Extension Pack** (necessario per USB 3.0, RDP, ecc.):
```bash
# Scaricare dal sito ufficiale la versione corrispondente a VirtualBox installato
# https://www.virtualbox.org/wiki/Downloads
# poi installare con:
sudo VBoxManage extpack install Oracle_VirtualBox_Extension_Pack-7.1.X.vbox-extpack
```

**Link download diretto pagina ufficiale:**
> https://www.virtualbox.org/wiki/Downloads

### Passo 5.3 — Vagrant

```bash
# Aggiungere il repository HashiCorp
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install vagrant -y
```

Verificare l'installazione:
```bash
vagrant --version
```

**Link pagina ufficiale:**
> https://developer.hashicorp.com/vagrant/install

### Passo 5.4 — KVM/QEMU (hypervisor ad alte prestazioni, facoltativo)

```bash
sudo apt install -y qemu-kvm libvirt-daemon-system virt-manager bridge-utils
sudo adduser $USER libvirt
sudo adduser $USER kvm
sudo systemctl enable --now libvirtd
```

Dopo il riavvio, avviare **virt-manager** dal menu applicazioni per gestire le VM KVM con interfaccia grafica.

---

<a name="fase-6"></a>
## FASE 6 — Setup Macchine Virtuali per i Corsi

### 6.1 — Kali Linux (Laboratorio di Sicurezza Informatica T)

Scaricare l'immagine VirtualBox dal sito ufficiale Kali:

**Download Kali Linux per VirtualBox:**
> https://www.kali.org/get-kali/#kali-virtual-machines

Selezionare: **VirtualBox** → **64-bit** (file `.7z`, circa 3.2 GB compressi).

Estrarre il file `.7z`:
```bash
sudo apt install p7zip-full -y
7z x kali-linux-*-virtualbox-amd64.7z
```

**Importare in VirtualBox:**
1. Aprire VirtualBox
2. Menu **File → Importa applicazione virtuale**
3. Selezionare il file `.ova` estratto
4. Cliccare **Importa**

> **Nota di sicurezza**: Kali Linux deve essere usata **esclusivamente dentro la VM**, mai come sistema host. L'isolamento garantito dalla virtualizzazione è parte integrante della pratica corretta di sicurezza informatica.

### 6.2 — VM per i corsi (fornite dai docenti)

Le immagini `.ova` distribuite dai docenti di Fondamenti di Informatica, Sistemi Operativi e Amministrazione di Sistemi si importano in modo identico:

```
VirtualBox → File → Importa applicazione virtuale → selezionare il file .ova
```

### 6.3 — Configurazione consigliata per le VM

Per ogni VM, nelle impostazioni VirtualBox, impostare:

| Parametro | Valore consigliato |
|---|---|
| RAM assegnata | 2–4 GB per VM leggere (Debian); 4–6 GB per Kali |
| CPU | 2–4 core virtuali |
| Accelerazione | VT-x/AMD-V abilitato, Nested Paging abilitato |
| Video | 128 MB |
| Rete | NAT (per internet) o Host-only (per isolamento) |
| Storage | Disco dinamico (cresce solo con i dati effettivi) |

---

## FASE 7 — Accesso Remoto al Laboratorio Ercolani (Opzionale)

Il laboratorio Ercolani è accessibile via SSH con le credenziali istituzionali UniBO. Richiedere l'accesso ai tecnici DISI come indicato sulla pagina:

> https://disi.unibo.it/en/department/technical-and-administrative-services/it-services/linux-account

Una volta ottenuto l'accesso:
```bash
ssh nome.cognome@<indirizzo-fornito-dal-DISI>
```

---

## RIEPILOGO LINK DI DOWNLOAD

| Software | Link | Note |
|---|---|---|
| **Debian 12 ISO** | https://www.debian.org/distrib/ | Scegliere DVD-1 amd64 |
| **Rufus** | https://rufus.ie/it/ | Per creare la chiavetta |
| **Ventoy** (alternativa) | https://www.ventoy.net/en/download.html | Multi-ISO |
| **VirtualBox** | https://www.virtualbox.org/wiki/Downloads | Versione Linux/Debian |
| **VBox Extension Pack** | https://www.virtualbox.org/wiki/Downloads | Stessa pagina di VBox |
| **Vagrant** | https://developer.hashicorp.com/vagrant/install | Tramite repo HashiCorp |
| **Kali Linux VM** | https://www.kali.org/get-kali/#kali-virtual-machines | Versione VirtualBox 64-bit |

---

## CHECKLIST FINALE

Prima di iniziare:
- [ ] Product Key Windows annotato e salvato
- [ ] Tipo licenza verificato (OEM/retail)
- [ ] ISO Debian scaricata e verificata
- [ ] Rufus scaricato
- [ ] Chiavetta USB da ≥8 GB disponibile

Durante l'installazione:
- [ ] Secure Boot disabilitato nel BIOS
- [ ] Boot da USB confermato
- [ ] Partizionamento: intero disco / partizione singola
- [ ] Desktop GNOME selezionato
- [ ] Server SSH selezionato

Post-installazione:
- [ ] Sistema aggiornato (`apt update && apt upgrade`)
- [ ] Repository non-free aggiunti (firmware hardware)
- [ ] VirtualBox installato e utente aggiunto a `vboxusers`
- [ ] Vagrant installato
- [ ] Kali Linux importata in VirtualBox
- [ ] Ottimizzazioni performance applicate

---

*Guida prodotta sulla base dell'analisi dell'ambiente DISI UniBO — Aprile 2026*
