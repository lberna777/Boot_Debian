# Ripristinare Windows in dual boot accanto a Debian

**Situazione di partenza**: il laptop è stato reinstallato seguendo `tutorial v3.md` — Debian occupa l'intero disco, partizionamento guidato a partizione singola (nessun LVM, nessuna cifratura LUKS), Secure Boot disabilitato nel BIOS. La licenza Windows originale (edizione OEM preinstallata) non è stata persa: vive nel firmware UEFI, non sul disco.

**Obiettivo**: restringere la partizione Debian, installare Windows nello spazio liberato, e ripristinare GRUB come boot manager con entrambi i sistemi selezionabili all'avvio.

**Tempo richiesto**: 2–4 ore, di cui circa 1 ora di attesa passiva (download ISO, installazione Windows).

**Rischio principale**: il ridimensionamento della partizione Debian è l'unico passo che può causare perdita di dati. Se i dati sul disco sono sacrificabili il backup completo si può saltare, ma il `salvataggio minimo` e il controllo `e2fsck` della FASE 1 vanno fatti comunque: costano cinque minuti e coprono la causa di fallimento più frequente.

---

## INDICE

- [FASE 0 — Verifiche preliminari (30 minuti)](#fase-0)
- [FASE 1 — Backup](#fase-1)
- [FASE 2 — Recupero della licenza dal firmware](#fase-2)
- [FASE 3 — Download ISO Windows e chiavetta avviabile](#fase-3)
- [FASE 4 — Ridimensionamento della partizione Debian](#fase-4)
- [FASE 5 — Installazione di Windows](#fase-5)
- [FASE 6 — Ripristino di GRUB](#fase-6)
- [FASE 7 — Configurazione post-installazione](#fase-7)
- [Risoluzione problemi](#problemi)

---

<a name="fase-0"></a>
## FASE 0 — Verifiche preliminari

Prima di decidere qualsiasi cosa servono i numeri reali del disco. Aprire un terminale su Debian ed eseguire:

```bash
# Layout completo del disco: partizioni, filesystem, dimensioni, punti di mount
lsblk -o NAME,SIZE,FSTYPE,FSUSED,FSAVAIL,LABEL,PARTLABEL,MOUNTPOINT

# Spazio effettivamente occupato dal sistema
df -h /

# Dimensione e occupazione della partizione EFI (ESP)
df -h /boot/efi

# Verifica che il sistema sia in modalità UEFI (non BIOS legacy)
[ -d /sys/firmware/efi ] && echo "UEFI OK" || echo "ATTENZIONE: BIOS legacy"

# Presenza del TPM (richiesto da Windows 11)
ls /sys/class/tpm/ && cat /sys/class/tpm/tpm0/tpm_version_major 2>/dev/null
```

### Cosa verificare negli output

| Verifica | Requisito | Se non è soddisfatto |
|---|---|---|
| Modalità firmware | Deve stampare `UEFI OK` | Con BIOS legacy la procedura è diversa e più fragile: fermarsi e rivalutare |
| Spazio libero su `/` | Almeno **130 GB liberi** per stare comodi, **80 GB** come minimo assoluto | Liberare spazio (vedi sotto) o rinunciare al dual boot su questo disco |
| Dimensione ESP (`/boot/efi`) | Almeno **100 MB liberi** dentro l'ESP | Vedere [Risoluzione problemi](#problemi) |
| TPM | `tpm_version_major` = `2` | Windows 11 richiede TPM 2.0; se assente vedere [Risoluzione problemi](#problemi) |

### Layout reale di questo laptop (rilevato)

```
NAME          SIZE FSTYPE FSUSED FSAVAIL MOUNTPOINT
nvme0n1     476,9G                                    ← SSD NVMe da 512 GB
├─nvme0n1p1   976M vfat     4,5M  969,5M /boot/efi    ← ESP, condivisa con Windows: NON TOCCARE
├─nvme0n1p2 460,2G ext4   224,6G  204,3G /            ← root Debian: da restringere
└─nvme0n1p3  15,7G swap                  [SWAP]       ← swap, in fondo al disco: NON TOCCARE
```

Nel resto della guida i device sono quelli reali: **`/dev/nvme0n1p1`** per l'ESP, **`/dev/nvme0n1p2`** per la root Debian.

Due verifiche della FASE 0 sono già superate:

- **ESP**: 976 MB con soli 4,5 MB occupati. Spazio più che sufficiente per il boot loader di Windows (~50 MB), quindi le due ESP separate e i problemi descritti in [Risoluzione problemi](#problemi) non si presenteranno.
- **Spazio libero**: 204,3 GiB su root. Sufficiente per una partizione Windows dimensionata bene.

Resta da verificare in autonomia il **TPM** (comandi qui sopra) e l'attivazione di **PTT** nel BIOS (FASE 3.3).

### Quanto spazio serve davvero a Windows

| Voce | Spazio |
|---|---|
| Windows 11 (requisito minimo Microsoft) | 64 GB |
| Windows 11 realmente installato + aggiornamenti | ~40 GB |
| Partizioni di servizio (MSR + WinRE) | ~0,7 GB |
| File di paging + ibernazione (con 16 GB di RAM) | ~20 GB |
| Applicazioni | 20–60 GB |
| **Totale consigliato** | **120–150 GB** |

Assegnare 64 GB "perché è il minimo" porta a una partizione piena entro pochi mesi, e ingrandirla dopo è molto più scomodo che dimensionarla bene adesso.

### Se lo spazio non basta

```bash
# Cosa occupa spazio nella home
du -h --max-depth=1 ~ | sort -rh | head -20

# Pulizia cache pacchetti e dipendenze orfane
sudo apt autoremove --purge
sudo apt clean

# Log di journald ridotti a 200 MB
sudo journalctl --vacuum-size=200M
```

Se anche così non si arriva a 80 GB liberi, l'alternativa concreta è un **secondo disco**: molti laptop i7 con 16 GB hanno un secondo slot M.2 o SATA libero. Un SSD dedicato a Windows elimina completamente il rischio del ridimensionamento e i due sistemi restano indipendenti. Verificare gli slot disponibili con `sudo dmidecode -t slot` e il manuale di servizio del modello.

---

<a name="fase-1"></a>
## FASE 1 — Backup

Il ridimensionamento di una partizione ext4 riesce senza problemi nella grande maggioranza dei casi, ma un'interruzione di corrente a metà lavoro rende il filesystem irrecuperabile.

### Se i dati sul disco sono sacrificabili

È la premessa di `tutorial v3.md`, e resta valida: se non c'è nulla da conservare, il backup completo da 225 GiB non ha senso. Quello che serve capire è **quanto costa il caso peggiore**, non quanto è probabile.

Se il resize fallisce si perde l'installazione Debian — non il disco, non Windows (non è ancora installato), non l'hardware. Il costo è rifare l'installazione seguendo `tutorial v3.md` o `tutorial v2.md`: 1–2 ore. Probabilità sull'ordine dell'1–2%.

Delle tre cause reali di fallimento, due si neutralizzano a costo zero e **non vanno saltate anche rinunciando al backup**:

| Causa | Mitigazione | Costo |
|---|---|---|
| Corruzione preesistente del filesystem | `e2fsck -f` prima del resize (FASE 4.2) | 5 minuti |
| Interruzione di corrente a metà operazione | Alimentatore collegato, non durante un temporale | zero |
| Guasto dell'SSD durante l'operazione | Non controllabile | — |

Il `salvataggio minimo` sotto va fatto in ogni caso: sono ~200 MB e cinque minuti, e copre le cose che *non* si rigenerano scaricandole di nuovo.

```bash
mkdir -p /media/$USER/CHIAVETTA/salvataggio-minimo
cd ~

# Chiavi, configurazioni, dotfile
tar czf /media/$USER/CHIAVETTA/salvataggio-minimo/essenziale.tar.gz \
  .ssh .gnupg .config/hypr .bashrc .zshrc .gitconfig 2>/dev/null

# Elenco pacchetti, per ricostruire il sistema rapidamente
dpkg --get-selections > /media/$USER/CHIAVETTA/salvataggio-minimo/pacchetti.txt

# Documenti e appunti: la parte davvero non rigenerabile
cp -r ~/Documenti /media/$USER/CHIAVETTA/salvataggio-minimo/ 2>/dev/null
sync
```

Le chiavi **SSH e GPG** sono la voce che fa più male perderle: non si riscaricano, vanno rigenerate e reinserite su GitHub e su ogni servizio dove erano registrate. Controllare anche di non avere lavoro non pushato:

```bash
# In ogni repository locale
git status && git log --oneline @{u}..HEAD
```

### L'alternativa: azzerare e reinstallare nell'ordine canonico

Se i dati sono sacrificabili si apre una strada che con dati da conservare sarebbe esclusa: **cancellare il disco, installare Windows per primo e Debian dopo**. È l'ordine per cui il dual boot è progettato — nessun resize, e l'installer Debian rileva Windows e configura GRUB da sé, per cui si salta l'intera FASE 6.

Il confronto è questo:

| | Resize (questa guida) | Azzerare e reinstallare |
|---|---|---|
| Tempo | ~45 min + FASE 6 | 3–4 ore |
| Debian da riconfigurare | Solo nell'1–2% dei casi | **Sempre** |
| Rischio di perdere l'installazione | 1–2% | 100% (è il piano) |
| Complessità | GParted + ripristino GRUB | Nessuna insidia |

Sui numeri conviene ancora il resize: si rischia un pomeriggio con probabilità dell'1–2% invece di spenderlo con certezza. L'azzeramento ha senso solo se l'installazione attuale sta comunque stretta e un sistema pulito è desiderabile di per sé — per esempio passando da GNOME all'assetto Hyprland di `tutorial v2.md`.

### Backup completo (se i dati contano)

Su disco esterno o chiavetta:

> **Su questo laptop la root occupa 224,6 GiB**: un backup completo richiede un disco esterno da almeno 256 GB. Prima di comprarne uno o di rinunciare, vale la pena guardare *cosa* sono quei 225 GiB — se buona parte sono immagini disco di VirtualBox o ISO riscaricabili, il backup che conta davvero si riduce a poche decine di GB:
>
> ```bash
> du -h --max-depth=2 ~ 2>/dev/null | sort -rh | head -25
> du -sh ~/VirtualBox\ VMs ~/Downloads 2>/dev/null
> ```
>
> Le VM si possono esportare in `.ova` compressi, o accettare di ricrearle. Le ISO si riscaricano. Codice, configurazioni, documenti e appunti universitari sono la parte non rigenerabile: quella va salvata comunque.

```bash
# Backup della home (esclude cache e file rigenerabili)
rsync -aAXv --info=progress2 \
  --exclude='.cache' --exclude='.local/share/Trash' \
  --exclude='.mozilla/firefox/*/cache2' \
  ~/ /media/$USER/DISCO_ESTERNO/backup-home/

# Elenco dei pacchetti installati, per ricostruire il sistema rapidamente
dpkg --get-selections > /media/$USER/DISCO_ESTERNO/pacchetti-installati.txt

# Backup dei file di configurazione di sistema modificati
sudo tar czf /media/$USER/DISCO_ESTERNO/etc-backup.tar.gz /etc
```

Salvare inoltre, **su un supporto separato dal laptop** (telefono, cloud, foglio di carta):

- il product key recuperato nella FASE 2;
- l'output di `lsblk -f` (contiene gli UUID delle partizioni, utili se GRUB va ricostruito a mano);
- l'output di `cat /etc/fstab`.

```bash
lsblk -f > ~/layout-disco-pre-windows.txt
cat /etc/fstab >> ~/layout-disco-pre-windows.txt
sudo efibootmgr -v >> ~/layout-disco-pre-windows.txt
```

Copiare `layout-disco-pre-windows.txt` fuori dal laptop.

---

<a name="fase-2"></a>
## FASE 2 — Recupero della licenza dal firmware

La licenza OEM di un laptop preinstallato è scritta in una tabella ACPI del firmware chiamata **MSDM**. Non è stata cancellata dalla formattazione, ed è leggibile direttamente da Linux:

```bash
sudo strings /sys/firmware/acpi/tables/MSDM | tail -1
```

L'output è una stringa nel formato `XXXXX-XXXXX-XXXXX-XXXXX-XXXXX`: è il product key originale del laptop. Se il file `MSDM` non esiste:

```bash
ls /sys/firmware/acpi/tables/ | grep -i msdm
```

Se non compare nulla, il laptop non ha una licenza embedded nel firmware — in quel caso serve il key annotato a suo tempo, oppure una licenza nuova.

### Sull'edizione: Enterprise, Pro o Education

Una precisazione importante, perché cambia quale ISO scaricare. Le licenze OEM preinstallate sui laptop consumer sono praticamente sempre **Home** o **Pro**, mai Enterprise: Enterprise si distribuisce solo tramite contratti di volume licensing aziendali. Se il ricordo è di un "Windows Enterprise", le possibilità sono due: era in realtà **Pro**, oppure era una licenza **Education/Enterprise fornita dall'università** (UniBO distribuisce licenze Windows Education agli studenti di Ingegneria/Informatica tramite il portale Azure for Education).

Come risolvere il dubbio senza indovinare: **il programma di installazione di Windows legge da sé la tabella MSDM**. Se nel firmware c'è un key, setup salta la richiesta del product key e installa automaticamente l'edizione corrispondente. Quindi:

1. scaricare l'ISO multi-edizione ufficiale di Windows 11 (contiene Home, Pro, Education);
2. lasciare che setup scelga l'edizione dal firmware;
3. se serve invece la licenza universitaria, recuperare il key dal portale Azure for Education di UniBO **prima** di iniziare, e inserirlo a installazione conclusa da *Impostazioni → Sistema → Attivazione*.

L'edizione Enterprise vera e propria non è scaricabile pubblicamente se non come versione di valutazione a 90 giorni: se un giorno servisse davvero, si passa da Pro a Enterprise con un cambio di key, senza reinstallare.

---

<a name="fase-3"></a>
## FASE 3 — Download ISO Windows e chiavetta avviabile

### 3.1 — Scaricare l'ISO

Da Linux, la pagina ufficiale Microsoft offre il download diretto dell'ISO (il selettore compare solo per i visitatori non-Windows, quindi da Debian funziona senza trucchi):

> https://www.microsoft.com/software-download/windows11

Selezionare *"Download Windows 11 Disk Image (ISO) for x64 devices"*, scegliere la lingua **Italiano** e scaricare. Il file è di circa **5,5–6 GB**.

Verificare l'integrità del download confrontando lo SHA-256 con quello pubblicato sulla pagina Microsoft:

```bash
sha256sum ~/Downloads/Win11_*.iso
```

### 3.2 — Preparare la chiavetta con Ventoy

Rufus non esiste su Linux, e `dd` **non funziona** con le ISO recenti di Windows: il file `install.wim` supera i 4 GB e non entra in una FAT32, per cui la chiavetta risulta non avviabile in UEFI. Lo strumento corretto da Linux è **Ventoy**, che oltre a gestire il problema permette di tenere più ISO sulla stessa chiavetta (comodo: la ISO di Debian serve nella FASE 4 e nella FASE 6).

Serve una chiavetta da **almeno 16 GB**, il cui contenuto verrà cancellato.

```bash
cd ~/Downloads

# Scaricare l'ultima release Ventoy per Linux da https://github.com/ventoy/Ventoy/releases
tar xzf ventoy-*-linux.tar.gz
cd ventoy-*/

# Identificare la chiavetta: verificarne la dimensione, NON sbagliare device
lsblk -o NAME,SIZE,MODEL,TRAN

# Installare Ventoy sulla chiavetta (sostituire sdX con il device corretto,
# es. /dev/sdb — NON una partizione come /dev/sdb1)
sudo ./Ventoy2Disk.sh -i /dev/sdX
```

> **Attenzione**: `Ventoy2Disk.sh -i` cancella l'intero device indicato. Controllare due volte l'output di `lsblk` prima di premere Invio: il disco interno del laptop non deve mai essere il bersaglio.

A installazione completata la chiavetta espone una partizione dati chiamata `Ventoy`. Copiarci dentro le ISO, semplicemente come file:

```bash
# Montare la partizione dati (di solito compare automaticamente)
cp ~/Downloads/Win11_*.iso /media/$USER/Ventoy/

# Copiare anche una ISO live di Debian: serve per GParted (FASE 4) e per
# riparare GRUB (FASE 6). Se non se ne ha una:
#   https://cdimage.debian.org/debian-cd/current-live/amd64/iso-hybrid/
cp ~/Downloads/debian-live-*-gnome.iso /media/$USER/Ventoy/

sync   # attendere il completamento della scrittura prima di scollegare
```

### 3.3 — Impostazioni BIOS da verificare

Riavviare ed entrare nel BIOS/UEFI (tasto secondo il produttore: Lenovo F1/F2, HP F10, Dell F2, ASUS F2/Del, Acer F2, MSI Del).

| Impostazione | Valore | Motivo |
|---|---|---|
| **TPM / PTT / fTPM** | **Enabled** | Requisito di Windows 11. Su Intel si chiama *PTT (Platform Trust Technology)*, su AMD *fTPM* |
| **Secure Boot** | Lasciare **Disabled** per ora | Setup di Windows richiede solo che la macchina sia *Secure Boot capable*, non che sia attivo. Si può riabilitare in FASE 7 |
| **SATA mode / VMD / RST** | **AHCI**, non RAID/RST | In modalità RST Windows non vede l'SSD senza driver aggiuntivi. Se Debian è già installato è quasi certamente già AHCI: **non cambiare questa impostazione**, cambiarla ora renderebbe Debian non avviabile |
| **Fast Boot** (firmware) | Disabled | Rende accessibile il menu di boot |

Salvare con **F10**.

---

<a name="fase-4"></a>
## FASE 4 — Ridimensionamento della partizione Debian

Una partizione root non si può restringere mentre è montata: serve avviare da un sistema live.

### 4.1 — Avviare GParted da live USB

1. riavviare con la chiavetta Ventoy inserita, premere il tasto del boot menu (F12 nella maggior parte dei casi) e selezionare la chiavetta;
2. dal menu Ventoy scegliere la **ISO live di Debian**;
3. a desktop caricato, aprire un terminale e lanciare GParted:

```bash
sudo apt update && sudo apt install -y gparted   # se non già presente
sudo gparted
```

### 4.2 — Controllo del filesystem

Prima di ridimensionare, verificare la salute del filesystem. In GParted: selezionare la partizione root (la grande ext4), menu **Partition → Check**. Oppure da terminale, a partizione **non montata**:

```bash
sudo umount /dev/nvme0n1p2 2>/dev/null
sudo e2fsck -f /dev/nvme0n1p2
```

Se `e2fsck` segnala errori, farli correggere e rieseguirlo finché è pulito. **Non ridimensionare un filesystem con errori.**

### 4.3 — Restringere la partizione

Dimensioni calcolate sul layout reale: root `/dev/nvme0n1p2` = 460,2 GiB = **471.244 MiB**, di cui 224,6 GiB occupati.

| Spazio a Windows | `Free space following (MiB)` | Root risultante | Libero su Debian dopo |
|---|---|---|---|
| 120 GiB | `122880` | 340,2 GiB | ~115 GiB |
| **150 GiB — consigliato** | **`153600`** | **310,2 GiB** | **~85 GiB** |
| 200 GiB | `204800` | 260,2 GiB | ~35 GiB — troppo stretto |

150 GiB è il punto di equilibrio: Windows sta comodo con applicazioni, aggiornamenti e file di paging, e a Debian restano ~85 GiB liberi, abbastanza per VM e pacchetti. A 200 GiB il lato Linux scenderebbe a ~35 GiB liberi, che con VirtualBox in uso si esaurirebbero in fretta.

In GParted:

1. selezionare **`/dev/nvme0n1p2`** (la ext4 da 460,2 GiB);
2. **Partition → Resize/Move**;
3. compilare **`Free space following (MiB)` = `153600`** e premere Tab: GParted ricalcola *New size* da sé. Conviene agire su questo campo invece che su *New size*, perché è la grandezza che conta e si evita un errore di sottrazione;
4. **verificare che `Free space preceding (MiB)` resti a `0`**. Se diventa diverso da zero, GParted sposterebbe l'inizio della partizione: operazione lentissima e che rende il sistema non avviabile. Rimetterlo a 0;
5. cliccare **Resize/Move**, poi il segno di spunta ✓ per applicare;
6. **non interrompere l'operazione**: con 224,6 GiB di dati da verificare e reindirizzare, dura verosimilmente **30–60 minuti**. Il laptop deve essere collegato alla corrente.

Al termine si vedrà lo spazio non allocato. **Lasciarlo non allocato**: sarà il programma di installazione di Windows a crearci le sue partizioni.

> **Sulla partizione di swap**: il partizionamento guidato di Debian mette la swap in fondo al disco, quindi lo spazio libero risulterà "in mezzo" tra root e swap. Questo va benissimo, Windows si installa senza problemi in un buco intermedio. Non spostare né eliminare la swap: si eviterebbe di dover aggiornare `/etc/fstab` e si eliminerebbe un rischio inutile.

### 4.4 — Verificare che Debian si avvii ancora

Riavviare **senza la chiavetta** e accertarsi che Debian parta normalmente e che `df -h /` mostri la nuova dimensione. Non procedere all'installazione di Windows prima di questa verifica: se qualcosa è andato storto, questo è il momento buono per accorgersene, con il backup ancora fresco e nessun'altra variabile in gioco.

---

<a name="fase-5"></a>
## FASE 5 — Installazione di Windows

### 5.1 — Avviare il setup

Riavviare dalla chiavetta Ventoy e selezionare la ISO di Windows 11. Al menu Ventoy scegliere **"Boot in normal mode"**.

Seguire le schermate iniziali: lingua **Italiano**, layout tastiera **Italiano**, poi **Installa ora**.

Alla richiesta del product key: se il firmware contiene la tabella MSDM, la schermata **non comparirà affatto** e l'edizione verrà scelta automaticamente. Se compare, scegliere **"Non ho un codice Product Key"** e selezionare l'edizione corrispondente alla licenza (Pro nella maggior parte dei casi): si attiva dopo, a sistema installato.

### 5.2 — Schermata di partizionamento (il passo critico)

Scegliere **"Personalizzata: installa solo Windows (avanzato)"** — mai *"Aggiornamento"*.

Comparirà l'elenco delle partizioni del disco. Un layout tipico:

Quello che comparirà su questo disco, dopo il ridimensionamento della FASE 4:

| Voce in elenco | Dimensione | Cos'è |
|---|---|---|
| Partizione 1: Sistema | **976 MB** | ESP — contiene GRUB, **NON TOCCARE** |
| Partizione 2: Primaria | **~310 GB** | Debian root ext4, **NON TOCCARE** |
| **Spazio non allocato** | **~150 GB** | ← **selezionare questo** |
| Partizione 3: Primaria | **~15,7 GB** | swap Linux, **NON TOCCARE** |

Windows non sa leggere ext4 né swap, quindi le partizioni 2 e 3 potrebbero comparire senza etichetta o come "Primaria" senza altre indicazioni: riconoscerle dalla dimensione. La sola voce da selezionare è quella da ~150 GB marcata *Spazio non allocato*.

> **Il punto in cui si distrugge tutto**: selezionare *"Spazio non allocato"* e cliccare **Nuovo**. Non cliccare mai **Formatta** o **Elimina** su nessuna partizione esistente, e in particolare non sulla Partizione 1: è la ESP condivisa, e formattarla cancella il boot loader di Debian. Se la schermata mostra un'unica voce "Spazio non allocato" pari all'intero disco, il ridimensionamento non è andato a buon fine — **annullare l'installazione** e tornare alla FASE 4.

Selezionato lo spazio non allocato, **Nuovo → Applica**. Windows creerà automaticamente la partizione MSR (16 MB) e quella di recupero WinRE (~600 MB) accanto alla partizione principale, e riutilizzerà la ESP esistente invece di crearne una nuova. Selezionare la partizione principale appena creata e cliccare **Avanti**.

### 5.3 — Completamento

L'installazione copia i file e riavvia più volte. Da questo momento **il laptop si avvierà direttamente in Windows**: è normale e atteso — il setup di Windows si mette in cima all'ordine di boot UEFI, ignorando GRUB. Debian è intatto, semplicemente non ancora raggiungibile dal menu. Si sistema nella FASE 6.

Completare la configurazione iniziale di Windows (account, privacy, rete). Due accortezze:

- **account locale invece di account Microsoft**: se si vuole evitare l'account Microsoft, alla schermata di rete scollegare il Wi-Fi, oppure premere `Shift+F10` e digitare `oobe\bypassnro` (riavvia l'OOBE con l'opzione offline);
- **BitLocker**: Windows 11 può attivare la cifratura automatica del disco al primo accesso con account Microsoft. In un dual boot è una complicazione seria — disattivarla appena possibile da *Impostazioni → Privacy e sicurezza → Crittografia dispositivo* → **Disattivato**.

### 5.4 — Attivazione

*Impostazioni → Sistema → Attivazione*. Con licenza OEM nel firmware, lo stato dovrebbe già essere **"Windows è attivato con una licenza digitale collegata all'hardware"**. Se chiede il key, inserire quello recuperato nella FASE 2 (o quello universitario).

---

<a name="fase-6"></a>
## FASE 6 — Ripristino di GRUB

Windows ha reso il proprio boot manager predefinito. Occorre rimettere GRUB davanti e fargli rilevare Windows. Due strade: quella rapida dal BIOS, e quella corretta via chroot.

### 6.1 — Tentativo rapido: ordine di boot dal BIOS

Entrare nel BIOS/UEFI e cercare la sezione **Boot Order** / **UEFI Boot Priority**. Se compaiono due voci — `debian` e `Windows Boot Manager` — spostare **`debian` in prima posizione**, salvare e riavviare. Se GRUB riappare, restano solo i passi 6.3 e 6.4.

### 6.2 — Ripristino via chroot da live USB

Se il BIOS non elenca `debian`, o GRUB non compare comunque, avviare la **ISO live di Debian** da Ventoy e aprire un terminale.

```bash
# 1. Identificare le partizioni: la root ext4 grande e la ESP (fat32, ~512 MB)
sudo lsblk -f

# 2. Montare la root Debian (sostituire con la partizione reale)
sudo mount /dev/nvme0n1p2 /mnt

# 3. Montare la ESP nella posizione attesa dal sistema
sudo mount /dev/nvme0n1p1 /mnt/boot/efi

# 4. Rendere disponibili i filesystem virtuali dentro il chroot
for d in /dev /dev/pts /proc /sys /run; do sudo mount --bind $d /mnt$d; done

# 5. Entrare nel sistema installato
sudo chroot /mnt /bin/bash
```

Dentro il chroot:

```bash
# Installare os-prober: è ciò che permette a GRUB di trovare Windows
apt update && apt install -y os-prober

# Verificare che il rilevamento degli altri sistemi non sia disabilitato
grep OS_PROBER /etc/default/grub
```

Se compare `GRUB_DISABLE_OS_PROBER=true`, cambiarlo in `false` (nelle versioni recenti di Debian il rilevamento è disattivato per impostazione predefinita, quindi questa riga va aggiunta se assente):

```bash
sed -i 's/^GRUB_DISABLE_OS_PROBER=true/GRUB_DISABLE_OS_PROBER=false/' /etc/default/grub
grep -q GRUB_DISABLE_OS_PROBER /etc/default/grub || \
  echo 'GRUB_DISABLE_OS_PROBER=false' >> /etc/default/grub
```

Poi reinstallare GRUB e rigenerare il menu:

```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi \
  --bootloader-id=debian --recheck

update-grub
```

L'output di `update-grub` deve contenere una riga tipo:

```
Found Windows Boot Manager on /dev/nvme0n1p1@/EFI/Microsoft/Boot/bootmgfw.efi
```

Se quella riga non appare, Windows non verrà elencato nel menu: vedere [Risoluzione problemi](#problemi).

Uscire e smontare tutto:

```bash
exit
sudo umount -R /mnt
sudo reboot
```

### 6.3 — Verifica dell'ordine di boot UEFI

A Debian riavviato:

```bash
sudo efibootmgr -v
```

`BootOrder` deve elencare per primo il numero corrispondente a `debian`. Se non è così:

```bash
# Esempio: se debian è Boot0002 e Windows Boot Manager è Boot0000
sudo efibootmgr -o 0002,0000
```

### 6.4 — Test finale

Riavviare e verificare che il menu GRUB mostri entrambe le voci, e che **entrambi** i sistemi si avvino correttamente. Provarli uno alla volta prima di considerare il lavoro finito.

---

<a name="fase-7"></a>
## FASE 7 — Configurazione post-installazione

Tre problemi si presentano in ogni dual boot Windows/Linux. Vale la pena sistemarli subito, prima di dimenticare che esistono.

### 7.1 — Disattivare Fast Startup di Windows

Con *Avvio rapido* attivo, Windows non si spegne davvero: iberna il kernel e lascia il filesystem NTFS in uno stato "sporco". Se da Debian si monta quella partizione, i dati possono corrompersi.

Su Windows, **PowerShell come amministratore**:

```powershell
powercfg /hibernate off
```

Questo disattiva sia l'ibernazione sia il Fast Startup. In alternativa dall'interfaccia: *Pannello di controllo → Opzioni risparmio energia → Specifica comportamento pulsanti di alimentazione → Modifica impostazioni attualmente non disponibili* → deselezionare **Attiva avvio rapido**.

### 7.2 — Risolvere lo sfasamento dell'orologio

Linux tiene l'orologio hardware in UTC, Windows in ora locale: alternandoli, l'ora risulta sbagliata di due ore. La soluzione corretta è dire a Windows di usare UTC. Da **PowerShell come amministratore**:

```powershell
reg add "HKLM\SYSTEM\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /t REG_DWORD /d 1 /f
```

Su Debian, verificare di essere in UTC:

```bash
timedatectl set-local-rtc 0
timedatectl   # "RTC in local TZ: no"
```

### 7.3 — Accedere alla partizione Windows da Debian

Con Fast Startup disattivato, la partizione NTFS si monta senza rischi:

```bash
# Identificare la partizione Windows (la NTFS grande)
lsblk -f | grep ntfs

sudo mkdir -p /mnt/windows
sudo mount -t ntfs3 /dev/nvme0n1p4 /mnt/windows   # partizione reale
```

Per un mount automatico all'avvio, aggiungere a `/etc/fstab` usando l'UUID (mai il nome del device, che può cambiare):

```
UUID=XXXXXXXXXXXXXXXX  /mnt/windows  ntfs3  defaults,noauto,user,uid=1000,gid=1000  0 0
```

`noauto` è deliberato: la partizione si monta su richiesta e un eventuale problema su NTFS non blocca l'avvio di Debian.

### 7.4 — Riabilitare Secure Boot (opzionale)

Debian supporta Secure Boot tramite shim firmato, quindi ora entrambi i sistemi possono convivere con Secure Boot attivo. Riabilitarlo dal BIOS **solo dopo** aver verificato che entrambi si avviino, e sapendo che con Secure Boot attivo i moduli kernel non firmati (per esempio quelli di VirtualBox) richiedono la firma MOK o non si caricheranno. Se sul laptop si usa VirtualBox come da `tutorial v3.md`, conviene lasciare Secure Boot disattivato.

### 7.5 — Ordine di boot dopo gli aggiornamenti di Windows

Certi aggiornamenti importanti di Windows si rimettono in cima all'ordine di boot UEFI. Non è un danno: si corregge in trenta secondi da Debian con `sudo efibootmgr -o <ordine>` (vedi 6.3), o dal BIOS. Vale la pena annotare l'output di `efibootmgr -v` da qualche parte, per non dover ricostruire i numeri ogni volta.

---

<a name="problemi"></a>
## Risoluzione problemi

### GRUB non trova Windows (`os-prober` non lo rileva)

Verificare che il boot loader di Windows sia effettivamente nella ESP:

```bash
sudo mount /dev/nvme0n1p1 /mnt   # la ESP
sudo ls -R /mnt/EFI/
```

Deve esistere `/mnt/EFI/Microsoft/Boot/bootmgfw.efi`. Se c'è ma `update-grub` lo ignora, aggiungere una voce manuale in `/etc/grub.d/40_custom`:

```bash
sudo tee -a /etc/grub.d/40_custom >/dev/null <<'EOF'
menuentry "Windows Boot Manager" --class windows --class os {
    insmod part_gpt
    insmod fat
    insmod chain
    search --fs-uuid --no-floppy --set=root XXXX-XXXX
    chainloader /EFI/Microsoft/Boot/bootmgfw.efi
}
EOF

sudo update-grub
```

Sostituire `XXXX-XXXX` con l'UUID della ESP, ottenibile con `sudo blkid /dev/nvme0n1p1` (campo `UUID`, formato corto tipo `A1B2-C3D4`).

Se `bootmgfw.efi` **non** esiste, Windows ha creato una propria ESP separata: cercarla con `lsblk -f | grep vfat` e usare quella nella voce manuale.

### La ESP è troppo piccola per Windows

Se `df -h /boot/efi` mostra meno di 100 MB liberi, il setup di Windows potrebbe fallire nel copiarci il proprio boot loader. Opzioni, in ordine di preferenza:

1. **liberare spazio nella ESP**: rimuovere i kernel vecchi con `sudo apt autoremove --purge`, che ripulisce anche gli initramfs;
2. **lasciare che Windows crei la propria ESP**: se nello spazio non allocato non trova una ESP utilizzabile, ne crea una nuova. Funziona, con due ESP sul disco, ma GRUB deve puntare a quella giusta (vedi sopra);
3. **ingrandire la ESP** con GParted: richiede di spostare l'inizio della partizione root, operazione lenta e con rischio reale di rendere il sistema non avviabile. Da considerare solo con un backup completo e verificato.

### Windows dice "Questo PC non può eseguire Windows 11"

Verificare TPM e Secure Boot capability nel BIOS (FASE 3.3). Se il TPM è assente o è solo 1.2, si può proseguire aggirando il controllo: alla schermata dell'errore premere `Shift+F10` per aprire il prompt, poi:

```
regedit
```

In `HKEY_LOCAL_MACHINE\SYSTEM\Setup` creare la chiave `LabConfig` e dentro due valori DWORD a `1`: `BypassTPMCheck` e `BypassSecureBootCheck`. Chiudere regedit e tornare indietro di una schermata. Va detto chiaramente: su un sistema così Windows 11 si installa ma Microsoft non ne garantisce gli aggiornamenti nel tempo. Su hardware senza TPM 2.0 la scelta più solida resta **Windows 10** (supporto terminato a ottobre 2025) o una macchina virtuale.

### Debian non si avvia più dopo l'installazione di Windows

Nella quasi totalità dei casi il sistema è intatto e il problema è solo l'ordine di boot: rifare la FASE 6. Se GRUB parte ma cade in `grub rescue>`, la partizione root è cambiata di numero — individuarla con `ls (hd0,gpt2)/` e reinstallare GRUB via chroot da live USB (FASE 6.2).

### Prima di tutto questo: le applicazioni girano davvero solo su Windows?

Un dual boot costa una partizione, un riavvio ogni volta che si cambia sistema, e la manutenzione della FASE 7. Se le applicazioni in questione sono poche, vale trenta minuti di verifica prima di impegnarsi:

- **https://appdb.winehq.org** — database di compatibilità Wine, cercare il nome dell'applicazione;
- **Bottles** (`flatpak install flathub com.usebottles.bottles`) — gestisce prefissi Wine con configurazioni preconfezionate, decisamente più semplice di Wine a mano;
- **VirtualBox** — già presente nel setup secondo `tutorial v3.md`: una VM Windows basta per software gestionali, client bancari, Office. Non basta per software che richiede GPU o accesso hardware diretto (CAD, giochi, programmatori di firmware).

Se le applicazioni richiedono GPU o hardware dedicato, il dual boot è la risposta giusta e questa guida è la strada. Se sono applicazioni da ufficio, la VM costa molto meno in manutenzione.

---

## Checklist finale

- [ ] `salvataggio-minimo` fatto: chiavi SSH/GPG, dotfile, documenti, elenco pacchetti
- [ ] Nessun commit locale non pushato (`git log --oneline @{u}..HEAD` in ogni repo)
- [ ] Backup completo della home, **se** i dati contano — e verificato aprendo qualche file
- [ ] `layout-disco-pre-windows.txt` copiato fuori dal laptop
- [ ] Product key recuperato dalla tabella MSDM e annotato altrove
- [ ] Almeno 130 GB liberi su `/`
- [ ] TPM 2.0 attivo nel BIOS
- [ ] Chiavetta Ventoy con ISO di Windows 11 **e** ISO live di Debian
- [ ] `e2fsck` sulla root eseguito e pulito
- [ ] Partizione ridimensionata, spazio libero **non allocato**, Debian riavviato correttamente
- [ ] Windows installato nello spazio non allocato (nessuna partizione esistente formattata)
- [ ] Windows attivato
- [ ] GRUB reinstallato, entrambe le voci presenti, entrambi i sistemi avviati e testati
- [ ] `powercfg /hibernate off` eseguito su Windows
- [ ] `RealTimeIsUniversal` impostato, orologio coerente tra i due sistemi
