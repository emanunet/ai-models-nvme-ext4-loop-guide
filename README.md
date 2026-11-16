# Gestione e Spostamento Modelli IA su Unità NVMe Esterna in Linux (Zorin OS)

Questa guida descrive come archiviare modelli di Intelligenza Artificiale (**Hugging Face**, **LM Studio**, **Unsloth**) su un disco esterno NVMe formattato in **exFAT** o **NTFS**, superando l'assenza del supporto ai collegamenti simbolici (*symlink*) tramite un **disco virtuale formattato in `ext4`** montato automaticamente all'avvio.

---

## Indice
1. [Perché usare un disco virtuale `ext4`](#1-perché-usare-un-disco-virtuale-ext4)
2. [Prerequisiti e segnaposto](#2-prerequisiti-e-segnaposto)
3. [Creazione e formattazione del disco virtuale](#3-creazione-e-formattazione-del-disco-virtuale)
4. [Protezione anti-scrittura e montaggio](#4-protezione-anti-scrittura-e-montaggio)
5. [Migrazione dei modelli e collegamenti simbolici](#5-migrazione-dei-modelli-e-collegamenti-simbolici)
   - [Cache di Hugging Face](#51-cache-di-hugging-face)
   - [LM Studio (AppImage / Standard)](#52-lm-studio-appimage--standard)
   - [LM Studio (Versione Flatpak)](#53-lm-studio-versione-flatpak)
   - [Unsloth Studio](#54-unsloth-studio)
6. [Montaggio automatico all'avvio (`/etc/fstab`)](#6-montaggio-automatico-allavvio-etcfstab)
7. [Comandi rapidi di manutenzione e controllo](#7-comandi-rapidi-di-manutenzione-e-controllo)

---

## 1. Perché usare un disco virtuale `ext4`

I dischi esterni sono spesso formattati in **exFAT** per garantire la compatibilità tra Windows, macOS e Linux. Tuttavia, librerie come `huggingface_hub` fanno largo uso di **collegamenti simbolici** nella cartella `blobs`, che non sono supportati da exFAT né da NTFS su Linux.

Creare un singolo file contenitore (`ai_models.img`) formattato internamente in **`ext4`** permette di:
- **Non ripartizionare** né rischiare i dati già presenti sull'unità esterna.
- **Supportare nativamente** permessi Linux e collegamenti simbolici.
- **Sfruttare la piena velocità** di interfacce NVMe USB4 / Thunderbolt (40 Gbps).

---

## 2. Prerequisiti e segnaposto

Prima di eseguire i comandi, verifica che l'unità esterna sia montata nel sistema e sostituisci i seguenti segnaposto con i tuoi valori reali:

| Segnaposto | Descrizione | Esempio |
| :--- | :--- | :--- |
| `<UTENTE>` | Il tuo nome utente Linux (`whoami`) | `mario` |
| `<NOME_DISCO>` | Il nome con cui l'unità esterna è montata in `/media/<UTENTE>/` | `NVMe_Esterno` |
| `<DIMENSIONI_MB>` | Capacità del disco virtuale in Megabyte (`1024 * GB`) | `512000` (per 500 GB) |
| `<UUID_DISCO>` | Identificativo univoco della partizione esterna (`lsblk -f`) | `672B-A9FE` |

> **Nota:** Evita di aprire o cliccare sul file `ai_models.img` dal file manager o dall'app *Dischi*, poiché il sistema desktop potrebbe creare un dispositivo di loop duplicato in sola lettura. Accedi sempre e solo tramite il punto di montaggio `/mnt/ai_models`.

---

## 3. Creazione e formattazione del disco virtuale

1. **Crea il file contenitore sull'unità esterna** (su `exFAT` il comando `fallocate` non è supportato, pertanto si utilizza `dd`):
   ```bash
   dd if=/dev/zero of=/media/<UTENTE>/<NOME_DISCO>/ai_models.img bs=1M count=<DIMENSIONI_MB> status=progress
   ```

2. **Formatta il file immagine in `ext4`** (conferma con `y` quando richiesto):
   ```bash
   mkfs.ext4 /media/<UTENTE>/<NOME_DISCO>/ai_models.img
   ```

---

## 4. Protezione anti-scrittura e montaggio

Per evitare che i programmi IA scrivano accidentalmente sul disco interno del PC qualora dimenticassi di collegare l'unità esterna, impostiamo i permessi `000` (nessun accesso) sulla cartella sottostante `/mnt/ai_models` prima di montare il disco virtuale:

```bash
sudo mkdir -p /mnt/ai_models
sudo chmod 000 /mnt/ai_models
sudo mount -o loop /media/<UTENTE>/<NOME_DISCO>/ai_models.img /mnt/ai_models
sudo chown -R <UTENTE>:<UTENTE> /mnt/ai_models
```

Verifica che il volume sia montato correttamente e mostri la capacità prevista:

```bash
df -h /mnt/ai_models
```

---

## 5. Migrazione dei modelli e collegamenti simbolici

### 5.1 Cache di Hugging Face

Copia la cache esistente mantenendo intatti i collegamenti simbolici (`-rp`), verifica la dimensione copiata e sostituisci la cartella locale con un collegamento simbolico:

```bash
cp -rp ~/.cache/huggingface /mnt/ai_models/huggingface
du -sh /mnt/ai_models/huggingface
rm -rf ~/.cache/huggingface
ln -s /mnt/ai_models/huggingface ~/.cache/huggingface
```

### 5.2 LM Studio (AppImage / Standard)

Sposta i modelli `.gguf` nel disco virtuale e collega la directory predefinita di LM Studio:

```bash
mkdir -p /mnt/ai_models/lm-studio
mv ~/.lmstudio/models/* /mnt/ai_models/lm-studio/
rmdir ~/.lmstudio/models
ln -s /mnt/ai_models/lm-studio ~/.lmstudio/models
```

### 5.3 LM Studio (Versione Flatpak)

Se utilizzi LM Studio in formato Flatpak, sposta gli eventuali modelli presenti nella cartella sandbox, crea il collegamento simbolico e concedi all'app il permesso di accesso a `/mnt/ai_models`:

```bash
cp -rn ~/.var/app/ai.lmstudio.lm-studio/.lmstudio/models/* /mnt/ai_models/lm-studio/
rm -rf ~/.var/app/ai.lmstudio.lm-studio/.lmstudio/models
ln -s /mnt/ai_models/lm-studio ~/.var/app/ai.lmstudio.lm-studio/.lmstudio/models
flatpak override --user --filesystem=/mnt/ai_models ai.lmstudio.lm-studio
```

### 5.4 Unsloth Studio

Nell'interfaccia web di Unsloth Studio:
1. Apri il menu di selezione dei modelli e individua la sezione **CUSTOM FOLDERS**.
2. Rimuovi eventuali vecchi percorsi locali o duplicati cliccando sulla **`X`**.
3. Clicca su **`+`** e aggiungi il percorso reale `/mnt/ai_models/lm-studio` (e, se necessario, `/mnt/ai_models/lm-studio/lmstudio-community`).

---

## 6. Montaggio automatico all'avvio (`/etc/fstab`)

1. **Recupera l'UUID della partizione esterna:**
   ```bash
   lsblk -f | grep <NOME_DISCO>
   ```

2. **Modifica il file `/etc/fstab`:**
   ```bash
   sudo nano /etc/fstab
   ```

3. **Aggiungi in fondo al file queste due righe** (sostituendo `<UUID_DISCO>`, `<UTENTE>` e `<NOME_DISCO>`):
   ```fstab
   UUID=<UUID_DISCO> /media/<UTENTE>/<NOME_DISCO> exfat defaults,uid=1000,gid=1000,nofail,x-systemd.device-timeout=5 0 0
   /media/<UTENTE>/<NOME_DISCO>/ai_models.img /mnt/ai_models ext4 loop,nofail,x-systemd.requires=/media/<UTENTE>/<NOME_DISCO> 0 0
   ```
   - `nofail`: impedisce il blocco del sistema all'avvio se l'NVMe esterno è scollegato.
   - `x-systemd.requires`: garantisce che il file `.img` venga montato solo dopo aver montato l'unità fisica esterna.

4. **Ricarica `systemd` e verifica la configurazione senza riavviare:**
   ```bash
   sudo systemctl daemon-reload
   sudo umount /mnt/ai_models
   sudo mount -a
   df -h /mnt/ai_models
   ```

---

## 7. Comandi rapidi di manutenzione e controllo

| Operazione | Comando |
| :--- | :--- |
| **Montaggio manuale (se collegato a PC già avviato)** | `sudo mount -a` |
| **Controllo spazio disco virtuale** | `df -h /mnt/ai_models` |
| **Dettaglio spazio per sottocartella** | `du -sh /mnt/ai_models/*` |
| **Elenco file superiori a 500 MB** | `find /mnt/ai_models -type f -size +500M -exec du -sh {} + \| sort -hr` |
| **Ricerca file duplicati esatti (SHA-256)** | `find /mnt/ai_models -type f -size +500M -exec sha256sum {} + \| sort \| uniq -w64 -D` |
