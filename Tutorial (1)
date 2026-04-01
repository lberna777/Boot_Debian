# Setup Laptop Dedicato — UniBO Ingegneria Informatica
**Lorenzo Bernardini — A.A. 2025/2026**

---

## 1. Analisi dell'Ambiente di Laboratorio UniBO

La ricerca condotta sulle risorse pubbliche del Dipartimento di Informatica — Scienza e Ingegneria (DISI) di UniBO ha prodotto i seguenti dati certi:

### 1.1 Laboratorio Ercolani (sede principale DISI, Bologna)
- **Sistema Operativo**: Debian GNU/Linux (distribuzione stabile)
- **Hardware dei workstation**: Intel Core i5-4590 @ 3.30 GHz, 8 GB RAM
- **Numero postazioni**: 52 PC fissi + 45 postazioni laptop
- **Accesso remoto**: disponibile 24/7 (escluso periodi d'esame)

### 1.2 Corsi e ambienti specifici rilevati

| Corso | Ambiente | Virtualizzazione | Note |
|---|---|---|---|
| **Fondamenti di Informatica T** | Debian GNU/Linux | Oracle VirtualBox | VM distribuita agli studenti |
| **Sistemi Operativi T** | OS161 (kernel didattico Unix-like) | VirtualBox | Kernel compilato e modificato dallo studente |
| **Laboratorio di Sicurezza Informatica T** | Kali Linux 64-bit | Oracle VirtualBox | Immagine `.ova` scaricabile (Kali 64-bit VBox, ~3.7 GB; versione light ~0.9 GB) |
| **Amministrazione di Sistemi T** | GNU/Linux multi-VM | VirtualBox + **Vagrant** | Architetture client/server/router simulate |

**Sintesi critica**: l'intero ecosistema didattico di DISI ruota attorno a **Debian GNU/Linux** come sistema host nei laboratori e **Oracle VirtualBox** come hypervisor standard per le esercitazioni. Il corso di sicurezza informatica aggiunge **Kali Linux** come macchina virtuale guest. Vagrant è lo strumento di provisioning usato in Amministrazione di Sistemi.

---

## 2. Licenza Windows Pro — Come Preservarla

Prima di qualsiasi operazione di formattazione, eseguire il seguente comando in PowerShell (come Amministratore) per estrarre e salvare il product key:

```powershell
(Get-WmiObject -query 'select * from SoftwareLicensingService').OA3xOriginalProductKey
```

**Nota importante**: la maggior parte dei portatili moderni con licenza OEM ha il product key embedded nel firmware UEFI/BIOS. In questo caso, una futura reinstallazione di Windows 11 Pro sullo stesso hardware lo rileverà e attiverà automaticamente. Il key estratto sopra va comunque annotato e conservato offline come backup.

---

## 3. Sistema Operativo Host Raccomandato

### Scelta: **Debian 12 "Bookworm" (stable)**

**Motivazione accademica**: è la distribuzione identica a quella installata nei laboratori Ercolani. Ogni comando eseguito, ogni configurazione applicata e ogni comportamento del sistema sarà direttamente trasferibile all'ambiente universitario senza adattamenti. Questo è il criterio dirimente rispetto ad alternative come Ubuntu o Fedora.

**Motivazione tecnica**: Debian stable è rinomato per l'altissima stabilità del sistema di base, l'assenza di overhead derivante da funzionalità enterprise (come in RHEL/Fedora), e la maturità degli strumenti di packaging (`apt`, `dpkg`).

### Alternativa accettabile: **Ubuntu 24.04 LTS "Noble Numbat"**
Ubuntu è un derivato Debian e garantisce compatibilità quasi totale con i comandi e i pacchetti del laboratorio. È preferibile a Debian solo nel caso in cui l'hardware del portatile (scheda Wi-Fi, GPU, touchpad) richieda driver più aggiornati non presenti nel kernel di Debian stable. In caso di dubbi, verificare la compatibilità hardware prima dell'installazione.

---

## 4. Ambiente Desktop

Per un i7 con 16 GB di RAM, il compromesso ottimale tra usabilità e performance è:

- **GNOME (Wayland)** — desktop raccomandato per Debian. Moderno, ben integrato, Wayland garantisce migliore gestione delle risorse grafiche rispetto a X11 su hardware recente. L'overhead su 16 GB è irrilevante.
- **XFCE** — alternativa se si vuole riservare ogni MB di RAM alle macchine virtuali. Più austero ma estremamente reattivo.

La scelta di XFCE è giustificata solo se si prevede di eseguire simultaneamente tre o più VM pesanti.

---

## 5. Configurazione della Virtualizzazione

### 5.1 Oracle VirtualBox (obbligatorio)
VirtualBox va installato sul sistema host come strumento primario, poiché è l'hypervisor usato in tutti i laboratori DISI. Le immagini `.ova` distribuite dai docenti saranno importabili senza modifiche.

```bash
sudo apt install virtualbox virtualbox-ext-pack
```

Le immagini da avere pronte:
- **Kali Linux 64-bit VBox** (~3.7 GB) — per il Laboratorio di Sicurezza Informatica
- **VM Debian** fornita dai docenti di Fondamenti di Informatica
- **VM per OS161** — da compilare nel corso di Sistemi Operativi

### 5.2 KVM/QEMU + virt-manager (facoltativo, consigliato)
Per carichi di lavoro ad alte prestazioni (compilazioni pesanti, esercitazioni di reti con più VM in parallelo), KVM è un hypervisor di tipo 1 che sfrutta le istruzioni hardware di virtualizzazione dell'i7 (Intel VT-x) in modo più efficiente di VirtualBox. I due strumenti coesistono senza conflitti.

```bash
sudo apt install qemu-kvm libvirt-daemon-system virt-manager
sudo adduser $USER libvirt
sudo adduser $USER kvm
```

### 5.3 Vagrant
Per il corso di Amministrazione di Sistemi T, Vagrant è strumento richiesto. Va installato separatamente dal repository ufficiale:

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vagrant
```

---

## 6. Ottimizzazione delle Prestazioni

### 6.1 Governor CPU
L'i7 di generazione recente supporta il governor `performance` per massimizzare il throughput computazionale:

```bash
sudo apt install cpufrequtils
echo 'GOVERNOR="performance"' | sudo tee /etc/default/cpufrequtils
sudo systemctl restart cpufrequtils
```

**Nota**: su portatile, impostare `performance` consuma più batteria. Alternativa bilanciata: `schedutil` (il governor predefinito moderno, auto-adattivo).

### 6.2 Gestione della Memoria — zram
Con 16 GB di RAM, lo swap su disco è raramente necessario. Si consiglia di sostituirlo con **zram** (swap compresso in RAM), che evita accessi al disco e mantiene le prestazioni in caso di utilizzo intenso di VM:

```bash
sudo apt install zram-tools
# Configurazione in /etc/default/zramswap
# ALGO=lz4 (algoritmo più veloce)
# PERCENT=25 (usa al massimo il 25% della RAM per lo swap compresso)
```

Abbassare la swappiness per ridurre il ricorso allo swap:
```bash
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.d/99-performance.conf
sudo sysctl -p /etc/sysctl.d/99-performance.conf
```

### 6.3 Filesystem
Usare **ext4** con l'opzione `noatime` per ridurre le scritture su disco:
In `/etc/fstab`, aggiungere `noatime` alle opzioni della partizione root:
```
UUID=... / ext4 defaults,noatime 0 1
```

**Alternativa**: **Btrfs** — offre compressione trasparente (`zstd`) e snapshot istantanei, utili per ripristinare l'ambiente dopo esperimenti andati male. Richiede una pianificazione dello schema di partizioni in fase di installazione.

### 6.4 TLP — Gestione Termica e Energetica
```bash
sudo apt install tlp tlp-rdw
sudo systemctl enable tlp
sudo tlp start
```
TLP ottimizza automaticamente il comportamento della CPU e dei dispositivi in base alla fonte di alimentazione.

### 6.5 Disabilitare Servizi Superflui
```bash
# Verificare i servizi attivi
systemctl list-units --type=service --state=running
# Esempi di servizi non necessari su una macchina da sviluppo:
sudo systemctl disable bluetooth  # se non usato
sudo systemctl disable cups       # se non si stampa
```

---

## 7. Strumenti Aggiuntivi Essenziali per i Corsi

```bash
# Compilazione e sviluppo (Fondamenti, Sistemi Operativi)
sudo apt install build-essential gcc g++ gdb make cmake git

# Strumenti di rete (Reti di Calcolatori, Sicurezza)
sudo apt install nmap wireshark tcpdump netcat-openbsd

# Ambienti Python e scripting
sudo apt install python3 python3-pip python3-venv

# Editor e ambienti di sviluppo
sudo apt install vim neovim emacs  # scegliere quello del professore di riferimento

# SSH e accesso remoto al lab Ercolani
sudo apt install openssh-client

# Strumenti per OS161 (Sistemi Operativi T)
sudo apt install bmake libmpc-dev
```

---

## 8. Schema di Partizioni Raccomandato

Per un disco da 256/512 GB con installazione Debian + spazio per VM:

| Partizione | Dimensione | Filesystem | Scopo |
|---|---|---|---|
| EFI | 512 MB | FAT32 | Boot UEFI |
| `/` (root) | 40–60 GB | ext4 / Btrfs | Sistema base |
| `/home` | 80–150 GB | ext4 / Btrfs | Dati utente, progetti |
| VM storage | 80–200 GB | ext4 | Immagini VirtualBox/KVM |
| swap | — | zram (in RAM) | Swap compresso |

**Nota**: posizionare le VM su una partizione dedicata facilita la gestione dello spazio e, in caso di utilizzo di Btrfs, permette di escluderle dalla compressione e dagli snapshot (le immagini di VM non beneficiano di compressione).

---

## 9. Accesso Remoto al Laboratorio Ercolani

Il laboratorio Ercolani è accessibile da remoto via SSH con le credenziali istituzionali UniBO (dopo autorizzazione dei tecnici DISI). Questo permette di lavorare sulle macchine del dipartimento direttamente dal portatile:

```bash
ssh nome.cognome@<indirizzo-lab-ercolani>
```

Verificare le istruzioni aggiornate sulla pagina [Account Linux — DISI](https://disi.unibo.it/en/department/technical-and-administrative-services/it-services/linux-account).

---

## 10. Riepilogo delle Scelte Architetturali

| Componente | Scelta | Motivazione |
|---|---|---|
| **OS Host** | Debian 12 Bookworm | Identico al lab Ercolani |
| **Desktop** | GNOME (Wayland) | Performance + usabilità su 16 GB |
| **Hypervisor primario** | Oracle VirtualBox | Compatibilità immagini didattiche |
| **Hypervisor secondario** | KVM/QEMU | Performance su carichi pesanti |
| **Provisioning VM** | Vagrant | Richiesto da Amministrazione di Sistemi |
| **VM Sicurezza** | Kali Linux 64-bit (VBox) | Ambiente isolato per penetration testing |
| **Swap** | zram (lz4, 25%) | Nessun accesso disco, prestazioni elevate |
| **Filesystem** | ext4 con noatime (o Btrfs) | Riduzione I/O, ottimizzazione velocità |
| **CPU Governor** | performance / schedutil | Massimo throughput computazionale |

---

*Documento generato sulla base dell'analisi delle risorse pubbliche DISI UniBO — Aprile 2026*
