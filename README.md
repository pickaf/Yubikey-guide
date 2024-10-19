Questa è una guida all'utilizzo [YubiKey](https://www.yubico.com/products/identifying-your-yubikey/) come [smart card](https://security.stackexchange.com/questions/38924/how-does-storing-gpg-ssh-private-keys-on-smart-cards-compare-to-plain-usb-drives) per crittografare, firmare e autenticare.

Le chiavi salvate nella Yubikey [non sono esportabili](https://web.archive.org/web/20201125172759/https://support.yubico.com/hc/en-us/articles/360016614880-Can-I-Duplicate-or-Back-Up-a-YubiKey-), a differenza delle credenziali basate su file system, pur rimanendo conveniente per l'uso quotidiano. YubiKey può essere configurato per richiedere un tocco fisico per le operazioni crittografiche, riducendo il rischio di compromissione delle credenziali.

- [Acquistare una YubiKey](#acquistare-yubikey)
- [Preparazione](#preparazione)
- [Installare il software](#installare-il-software)
- [Preparare GnuPG](#preparare-gnupg)
   * [Configurazione](#configurazione)
   * [Identita](#identita)
   * [Chiave](#chiave)
   * [Scadenza](#scadenza)
   * [Passphrase](#passphrase)
- [Creare una chiave di certificazione](#creare-una-chiave-di-certificazione)
- [Creare le sottochiavi](#creare-le-sottochiavi)
- [Verificare le chiavi](#verificare-le-chiavi)
- [Backup delle chavi](#backup-delle-chiavi)
- [Esporta la chiave pubblica](#esporta-la-chiave-pubblica)
- [Configura la YubiKey](#configura-la-yubikey)
   * [Cambia PIN](#cambia-pin)
   * [Set attributes](#set-attributes)
- [Trasferire le sottochiavi](#transferire-le-sottochiavi)
   * [Chiave di firma](#chiave-di-firma)
   * [Chiave di crittografia](#chiave-di-crittografia)
   * [Chiave di autentizazione](#chiave-di-autenticazione)
- [Verifica trasferimento](#verifica-trasferimento)
- [Setup finale](#setup-finale)
- [Usa la YubiKey](#usa-la-yubikey)
   * [Crittografia](#crittografia)
   * [Firma](#firma)
   * [Configura tocco](#configura-tocco)
   * [SSH](#ssh)
      + [Prerequisti](#prerequisiti)
      + [Cambio PIN e PUK](#cambio-pin-e-puk)
      + [Creazione del certificato](#creazione-del-certificato)
      + [Carica la chiave pubblica](#carica-la-chiave-pubblica)
      + [Prova ad accedere](#prova-ad-accedere)
      + [Note aggiuntive](#note-aggiuntive)
   * [GitHub](#github)
   * [Utilizzo di più YubiKey](#utilizzo-di-più-yubikey)
   * [Email](#email)
      + [Thunderbird](#thunderbird)
   * [Keyserver](#keyserver)
- [Aggiornamento delle chaivi](#aggiornamento-delle-chiavi)
   * [Migliorare l'entropia](#migliorare-l-entropia)
   * [Ruota le sottochaivi](#ruota-le-sottochiavi)
- [Reset YubiKey](#reset-yubikey)
- [Indurimento opzionale](#indurimento-opzionale)
   * [Migliorare l'entropia](#migliorare-l-entropia)
   * [Abilità KDF](#abilità-kdf)
   * [Network](#network)
- [Note](#note)
- [Risoluzione dei problemi](#risoluzione-dei-problemi)
- [Soluzioni alternative](#soluzion-alternative)
- [Risorse aggiuntive](#risorse-aggiuntive)

# Acquistare Yubikey

Le attuali [Yubikey](https://www.yubico.com/store/compare/) ad eccezione della serie di chiavi di sicurezza solo FIDO e delle YubiKey della serie Bio, sono compatibili con questa guida.

[Verifica la Yubikey](https://support.yubico.com/hc/en-us/articles/360013723419-How-to-Confirm-Your-Yubico-Device-is-Genuine) visitando [yubico.com/genuine](https://www.yubico.com/genuine/). Seleziona *Verify Device* per iniziare il processo. Tocca YubiKey quando richiesto e consenti al sito di vedere la marca e il modello del dispositivo quando richiesto. Questa attestazione del dispositivo può aiutare a mitigare un [supply chain attacks](https://media.defcon.org/DEF%20CON%2025/DEF%20CON%2025%20presentations/DEF%20CON%2025%20-%20r00killah-and-securelyfitz-Secure-Tokin-and-Doobiekeys.pdf).

Si consigliano inoltre diversi dispositivi di archiviazione portatili (come le schede microSD) per l'archiviazione di backup crittografati.

# Preparazione

Si consiglia un ambiente operativo dedicato e sicuro per generare chiavi crittografiche.

In questa guida useremo Debian Live perchè bilancia usabilità e sicurezza.

Scarica i file d' immagine e di firma più recenti: 

```console
curl -fLO "https://cdimage.debian.org/debian-cd/current-live/amd64/iso-hybrid/SHA512SUMS"

curl -fLO "https://cdimage.debian.org/debian-cd/current-live/amd64/iso-hybrid/SHA512SUMS.sign"

curl -fLO "https://cdimage.debian.org/debian-cd/current-live/amd64/iso-hybrid/$(awk '/xfce.iso$/ {print $2}' SHA512SUMS)"
```

Scarica la chiave pubblica di firma Debian:

```console
gpg --keyserver hkps://keyring.debian.org --recv DF9B9C49EAA9298432589D76DA87E80D6294BE9B
```

Se non è possibile ricevere la chiave pubblica, utilizzare un server di chiavi o un server DNS diverso:
```console
gpg --keyserver hkps://keyserver.ubuntu.com:443 --recv DF9B9C49EAA9298432589D76DA87E80D6294BE9B
```

Verifica la firma:

```console
gpg --verify SHA512SUMS.sign SHA512SUMS
```

`gpg: Good signature from "Debian CD signing key <debian-cd@lists.debian.org>"` deve apparire nell'output.

Verifica che l'hash crittografico del file immagine corrisponda a quello nel file firmato:

```console
grep $(sha512sum debian-live-*-amd64-xfce.iso) SHA512SUMS
```

Collegare un dispositivo di archiviazione portatile e identificare l'etichetta del disco.

```console
$ sudo dmesg | tail
usb-storage 3-2:1.0: USB Mass Storage device detected
sd 2:0:0:0: [sdc] Attached SCSI removable disk
```

Copia l'immagine di Debian nel dispositivo:

```console
sudo dd if=debian-live-*-amd64-xfce.iso of=/dev/sdc bs=4M status=progress ; sync
```

Spegnere, rimuovere i dischi rigidi interni e tutti i dispositivi non necessari, come la scheda wireless. 

# Installare il software

Caricare il sistema operativo e configurare la rete. 

**Importante:** Prima di collegare alla rete la macchina è auspicabile che tu legga la sezione [network](#network).

**Note:** Se lo schermo si blocca puoi sbloccarlo usando `user` come username e `live` come password.

Apri il terminale e installa i pacchetti software richiesti.

```console
sudo apt update

sudo apt -y upgrade

sudo apt -y install \
  wget gnupg2 gnupg-agent dirmngr \
  cryptsetup scdaemon pcscd \
  yubikey-personalization yubikey-manager
```

**Note:**  Per l'utilizzo potrebbe essere necessario installare una dipendenza aggiuntiva del pacchetto python [`ykman`](https://support.yubico.com/support/solutions/articles/15000012643-yubikey-manager-cli-ykman-user-guide) - `pip install yubikey-manager`

# Preparare GnuPG

Crea una directory temporanea che verrà cancellata al [riavvio](https://en.wikipedia.org/wiki/Tmpfs) e impostala come directory GnuPG:

```console
export GNUPGHOME=$(mktemp -d -t gnupg-$(date +%Y-%m-%d)-XXXXXXXXXX)
```

## Configurazione

Importa o crea una [configurazione rafforzata](https://github.com/drduh/config/blob/master/gpg.conf):

```console
cd $GNUPGHOME

wget https://raw.githubusercontent.com/drduh/config/master/gpg.conf
```

La rete può essere disabilitata per il resto della configurazione.

## Identita

Quando si crea un'identità con GnuPG, le opzioni predefinite richiedono un "Nome reale", un "Indirizzo email" e un "Commento" opzionale. 

A seconda di come intendi utilizzare GnuPG, imposta rispettivamente questi valori: 

```console
export IDENTITY="YubiKey User <yubikey@example>"
```

Oppure utilizza qualsiasi attributo che identificherà in modo univoco la chiave (questo potrebbe essere incompatibile con alcuni casi d'uso): 

```console
export IDENTITY="My Cool YubiKey - 2024"
```

## Chiave

Selezionare l'algoritmo e la dimensione della chiave desiderati. Questa guida consiglia RSA a 4096 bit. 

Imposta il valore:

```console
export KEY_TYPE=rsa4096
```

## Scadenza

Determinare la durata di validità della sottochiave desiderata. 

L'impostazione della scadenza di una sottochiave impone la gestione del ciclo di vita dell'identità e delle credenziali. Tuttavia, impostare una scadenza sulla chiave di certificazione è inutile, perché può essere utilizzata solo per estendersi. I certificati di revoca dovrebbero invece essere utilizzati per revocare un’identità. 

Questa guida consiglia una scadenza di due anni per le sottochiavi per bilanciare sicurezza e usabilità, tuttavia sono possibili durate più lunghe per ridurre la frequenza di manutenzione.

Quando le sottochiavi scadono, possono ancora essere utilizzate per decrittografare con GnuPG e autenticarsi con SSH, tuttavia non possono essere utilizzate per crittografare o firmare nuovi messaggi. 

Le sottochiavi devono essere rinnovate o ruotate utilizzando la chiave di certificazione.

Imposta la data di scadenza su due anni:

```console
export EXPIRATION=2y
```

Oppure imposta la data di scadenza su una data specifica per pianificare la manutenzione: 

```console
export EXPIRATION=2026-05-01
```

## Passphrase

Genera una passphrase per la chiave di certificazione. Verrà utilizzato raramente per gestire le sottochiavi e dovrebbe essere molto potente. Si consiglia che la passphrase sia composta solo da lettere maiuscole e numeri per una migliore leggibilità.

I seguenti comandi genereranno una passphrase forte ed eviteranno caratteri ambigui: 

```console
export CERTIFY_PASS=$(LC_ALL=C tr -dc 'A-Z1-9' < /dev/urandom | \
  tr -d "1IOS5U" | fold -w 30 | sed "-es/./ /"{1..26..5} | \
  cut -c2- | tr " " "-" | head -1) ; printf "\n$CERTIFY_PASS\n\n"
```

Scrivi la passphrase in un luogo sicuro, possibilmente separato dal dispositivo di archiviazione portatile utilizzato per il materiale della chiave, oppure memorizzala. 

Questo repository include il modello [`passphrase.html`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/passphrase.html) per facilitare la trascrizione delle credenziali. Salva il file raw, aprilo con un browser e stampalo. Utilizzare una penna o un pennarello indelebile per selezionare una lettera o un numero su ciascuna riga per ciascun carattere della passphrase. [`passphrase.csv`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/passphrase.csv) può essere stampato anche senza browser:

```console
lp -d Printer-Name passphrase.csv
```

# Creare una chiave di certificazione

La chiave primaria da generare è la chiave `Certify`, che è responsabile dell'emissione delle sottochiavi per le operazioni di crittografia, firma e autenticazione. 

La chiave di certificazione deve essere tenuta sempre offline e accessibile solo da un ambiente dedicato e sicuro per emettere o revocare sottochiavi. 

Non impostare una data di scadenza sulla chiave di certificazione.

Genera la chiave di certificazione:

```console
gpg --batch --passphrase "$CERTIFY_PASS" \
    --quick-generate-key "$IDENTITY" "$KEY_TYPE" cert never
```

Imposta e visualizza l'identificatore chiave e l'impronta digitale per utilizzarli in seguito: 

```console
export KEYID=$(gpg -k --with-colons "$IDENTITY" | awk -F: '/^pub:/ { print $5; exit }')

export KEYFP=$(gpg -k --with-colons "$IDENTITY" | awk -F: '/^fpr:/ { print $10; exit }')

printf "\nKey ID: %40s\nKey FP: %40s\n\n" "$KEYID" "$KEYFP"
```

# Creare le sottochiavi

Utilizzare il comando seguente per generare sottochiavi di firma, crittografia e autenticazione utilizzando il tipo di chiave, la passphrase e la scadenza precedentemente configurati: 

```console
for SUBKEY in sign encrypt auth ; do \
  gpg --batch --pinentry-mode=loopback --passphrase "$CERTIFY_PASS" \
      --quick-add-key "$KEYFP" "$KEY_TYPE" "$SUBKEY" "$EXPIRATION"
done
```

# Verificare le chiavi

Elenca le chiavi segrete disponibili:

```console
gpg -K
```

L'output visualizzerà le chiavi **[C]ertify, [S]ignature, [E]ncryption** e **[A]uthentication**:

```console
sec   rsa4096/0xF0F2CFEB04341FB5 2024-01-01 [C]
      Key fingerprint = 4E2C 1FA3 372C BA96 A06A  C34A F0F2 CFEB 0434 1FB5
uid                   [ultimate] YubiKey User <yubikey@example>
ssb   rsa4096/0xB3CD10E502E19637 2024-01-01 [S] [expires: 2026-05-01]
ssb   rsa4096/0x30CBE8C4B085B9F7 2024-01-01 [E] [expires: 2026-05-01]
ssb   rsa4096/0xAD9E24E1B8CB9600 2024-01-01 [A] [expires: 2026-05-01]
```

# Backup delle chiavi

Salvare una copia della chiave di certificazione, delle sottochiavi e della chiave pubblica: 

```console
gpg --output $GNUPGHOME/$KEYID-Certify.key \
    --batch --pinentry-mode=loopback --passphrase "$CERTIFY_PASS" \
    --armor --export-secret-keys $KEYID

gpg --output $GNUPGHOME/$KEYID-Subkeys.key \
    --batch --pinentry-mode=loopback --passphrase "$CERTIFY_PASS" \
    --armor --export-secret-subkeys $KEYID

gpg --output $GNUPGHOME/$KEYID-$(date +%F).asc \
    --armor --export $KEYID
```

Crea un **backup crittografato** su un dispositivo di archiviazione portatile da conservare offline in un luogo sicuro e durevole. 

Si consiglia di ripetere più volte il seguente processo su più dispositivi di archiviazione portatili, poiché è probabile che falliscano nel tempo. Come misura di backup aggiuntiva, [Paperkey](https://www.jabberwocky.com/software/paperkey/) può essere utilizzato per creare una copia fisica dei materiali chiave per una maggiore durata. 

**Suggerimento** Il file system [ext2](https://en.wikipedia.org/wiki/Ext2) senza crittografia può essere montato su Linux e OpenBSD. Utilizza invece il file system `FAT32` o `NTFS` per la compatibilità con macOS e Windows. 

**Linux**

In questo caso, collega un dispositivo di archiviazione portatile e controlla la sua etichetta `/dev/sdc`:

```console
$ sudo dmesg | tail
usb-storage 3-2:1.0: USB Mass Storage device detected
sd 2:0:0:0: [sdc] Attached SCSI removable disk

$ sudo fdisk -l /dev/sdc
Disk /dev/sdc: 14.9 GiB, 15931539456 bytes, 31116288 sectors
```

**Attezione** Confermare la destinazione ( `of`) prima di emettere il comando seguente - è distruttivo! Questa guida utilizza `/dev/sdc` in tutto, ma questo valore potrebbe essere diverso sul tuo sistema. 

Azzerare l'intestazione per prepararsi alla crittografia: 

```console
sudo dd if=/dev/zero of=/dev/sdc bs=4M count=1
```

Rimuovere e ricollegare il dispositivo di archiviazione.

Cancella e crea una nuova tabella delle partizioni:

```console
sudo fdisk /dev/sdc <<EOF
g
w
EOF
```

Crea una piccola partizione (si consigliano almeno 20 Mb per tenere conto della dimensione dell'intestazione LUKS) per archiviare materiali segreti: 


```console
sudo fdisk /dev/sdc <<EOF
n


+20M
w
EOF
```

Usa [LUKS](https://dys2p.com/en/2023-05-luks-security.html) per crittografare la nuova partizione.

Genera un'altra [Passphrase](#passphrase) (idealmente differente da quella usata pel la chiave di certificazione) univoca per proteggere il volume crittografato:

```console
export LUKS_PASS=$(LC_ALL=C tr -dc 'A-Z1-9' < /dev/urandom | \
  tr -d "1IOS5U" | fold -w 30 | sed "-es/./ /"{1..26..5} | \
  cut -c2- | tr " " "-" | head -1) ; printf "\n$LUKS_PASS\n\n"
```

Questa passphrase verrà utilizzata raramente per accedere alla chiave Certify e dovrebbe essere molto complessa.

Annotare la passphrase o memorizzarla. 

Formatta la partizione:

```console
echo $LUKS_PASS | sudo cryptsetup -q luksFormat /dev/sdc1
```

Monta la partizione:

```console
echo $LUKS_PASS | sudo cryptsetup -q luksOpen /dev/sdc1 gnupg-secrets
```

Crea un file system ext2:

```console
sudo mkfs.ext2 /dev/mapper/gnupg-secrets -L gnupg-$(date +%F)
```

Monta il filesystem e copia la directory di lavoro temporanea di GnuPG:

```console
sudo mkdir /mnt/encrypted-storage

sudo mount /dev/mapper/gnupg-secrets /mnt/encrypted-storage

sudo cp -av $GNUPGHOME /mnt/encrypted-storage/
```

Smonta e chiudi il volume crittografato:

```console
sudo umount /mnt/encrypted-storage

sudo cryptsetup luksClose gnupg-secrets
```

Ripeti la procedura per eventuali dispositivi di archiviazione aggiuntivi (se ne consigliano almeno due).

# Esporta la chiave pubblica

**Importante:** Senza la chiave pubblica, non sarà possibile utilizzare GnuPG per decrittografare o firmare i messaggi. Tuttavia, YubiKey può ancora essere utilizzata per l'autenticazione SSH qualora si usi GnuPG per gestire le chiavi SSH (in questa guida non saremo cosi pazzi da farlo).

Collega un altro dispositivo di archiviazione portatile o crea una nuova partizione su quella esistente. 

**Linux**

Usando lo stesso `/dev/sdc` dispositivo come nel passaggio precedente, creare una piccola partizione (si consigliano almeno 20 Mb) per archiviare i materiali: 

```console
sudo fdisk /dev/sdc <<EOF
n


+20M
w
EOF
```

Creare un file system e esportare la chiave pubblica:

```console
sudo mkfs.ext2 /dev/sdc2

sudo mkdir /mnt/public

sudo mount /dev/sdc2 /mnt/public

gpg --armor --export $KEYID | sudo tee /mnt/public/$KEYID-$(date +%F).asc

sudo chmod 0444 /mnt/public/*.asc
```

Smonta e rimuovi il dispositivo di archiviazione:

```console
sudo umount /mnt/public
```

# Configura la YubiKey

Connetti YubiKey e confermane lo stato: 

```console
gpg --card-status
```

Se è bloccata, [ripristinala](#reset-yubikey).

## Cambia PIN

L'[interfaccia PGP](https://developers.yubico.com/PGP/) ha i propri PIN separati da altri moduli come [PIV](https://developers.yubico.com/PIV/Introduction/YubiKey_and_PIV.html):

Name       | Default value | Capability
-----------|---------------|-------------------------------------------------------------
User PIN   | `123456`      | operazioni crittografiche (decifrare, firmare, autenticare)
Admin PIN  | `12345678`    | reimpostare il PIN, modificare il codice di reimpostazione, aggiungere chiavi e informazioni sul proprietario
Reset Code | None          | reset PIN ([maggiori info](https://forum.yubico.com/viewtopicd01c.html?p=9055#p9055))

Determinare i valori PIN desiderati. Possono essere più brevi della passphrase della chiave Certify a causa delle limitate opportunità di forzatura bruta; il PIN utente dovrebbe essere abbastanza comodo da essere ricordato per l'uso quotidiano.

Il *PIN utente* deve contenere almeno 6 caratteri e il *PIN amministratore* deve contenere almeno 8 caratteri. Sono consentiti un massimo di 127 caratteri ASCII. Vedi [GnuPG - Managing PINs](https://www.gnupg.org/howtos/card-howto/en/ch03s02.html) per maggiori informazioni.

Imposta i PIN manualmente o generali, ad esempio un PIN utente di 6 cifre e un PIN amministratore di 8 cifre: 

```console
export ADMIN_PIN=$(LC_ALL=C tr -dc '0-9' < /dev/urandom | fold -w8 | head -1)

export USER_PIN=$(LC_ALL=C tr -dc '0-9' < /dev/urandom | fold -w6 | head -1)

printf "\nAdmin PIN: %12s\nUser PIN: %13s\n\n" "$ADMIN_PIN" "$USER_PIN"
```

Cambia il pin amministratore:

```console
gpg --command-fd=0 --pinentry-mode=loopback --change-pin <<EOF
3
12345678
$ADMIN_PIN
$ADMIN_PIN
q
EOF
```

Cambia il pin utente:

```console
gpg --command-fd=0 --pinentry-mode=loopback --change-pin <<EOF
1
123456
$USER_PIN
$USER_PIN
q
EOF
```

Rimuovi e inserisci la Yubikey

**Avviso:** sbagliati tre *PIN utente* causeranno il blocco e dovra essere sbloccata con il *PIN amministratore* o con il codice di ripristino. Tre *PIN amministratore* o codici di ripristino errati distruggeranno i dati sulla YubiKey. 

Il numero di [tentativi](https://docs.yubico.com/software/yubikey/tools/ykman/OpenPGP_Commands.html#ykman-openpgp-access-set-retries-options-pin-retries-reset-code-retries-admin-pin-retries) può essere modificato, ad esempio a 5 tentativi:

```console
ykman openpgp access set-retries 5 5 5 -f -a $ADMIN_PIN
```

## Set attributes

Imposta gli [attibuti della smart card](https://gnupg.org/howtos/card-howto/en/smartcard-howto-single.html) con `gpg --edit-card` e `admin` mode - use `help` per visualizzare le opzioni disponibili.

Oppure usa valori predeterminati: 

```console
gpg --command-fd=0 --pinentry-mode=loopback --edit-card <<EOF
admin
login
$IDENTITY
$ADMIN_PIN
quit
EOF
```

Esegui `gpg --card-status` per verificare i risultati.

# Transferire le sottochiavi

**Importante:**  Il trasferimento delle chiavi su YubiKey è un'operazione unidirezionale che converte la chiave su disco in uno stub rendendola non più utilizzabile per il trasferimento su YubiKey successive. Assicurarsi che sia stato effettuato un backup prima di procedere. 

Per trasferire le chiavi sono necessari la passphrase della chiave `Certify` e il PIN amministratore. 

## Chiave di firma

Trasferisci la prima chiave:

```console
gpg --command-fd=0 --pinentry-mode=loopback --edit-key $KEYID <<EOF
key 1
keytocard
1
$CERTIFY_PASS
$ADMIN_PIN
save
EOF
```

## Chiave di crittografia

Ripeti la procedura per la seconda chiave:

```console
gpg --command-fd=0 --pinentry-mode=loopback --edit-key $KEYID <<EOF
key 2
keytocard
2
$CERTIFY_PASS
$ADMIN_PIN
save
EOF
```

## Chiave di autenticazione

Ripeti la procedura per la terza chiave:

```console
gpg --command-fd=0 --pinentry-mode=loopback --edit-key $KEYID <<EOF
key 3
keytocard
3
$CERTIFY_PASS
$ADMIN_PIN
save
EOF
```

# Verifica trasferimento

Verifica che le sottochiavi siano state spostate su YubiKey con `gpg -K` e verificare la presenza di `ssb>`, per esempio:

```console
sec   rsa4096/0xF0F2CFEB04341FB5 2024-01-01 [C]
      Key fingerprint = 4E2C 1FA3 372C BA96 A06A  C34A F0F2 CFEB 0434 1FB5
uid                   [ultimate] YubiKey User <yubikey@example>
ssb>  rsa4096/0xB3CD10E502E19637 2024-01-01 [S] [expires: 2026-05-01]
ssb>  rsa4096/0x30CBE8C4B085B9F7 2024-01-01 [E] [expires: 2026-05-01]
ssb>  rsa4096/0xAD9E24E1B8CB9600 2024-01-01 [A] [expires: 2026-05-01]
```

Il `>` indica che la chiave è memorizzata su una smart card.

# Setup finale

Verifica di aver effettuato quanto segue:

- [ ] Memorizzato o annotato la passphrase della chiave di certificazione (identità) in un luogo sicuro e durevole
  * `echo $CERTIFY_PASS` per rivederla; [`passphrase.html`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/passphrase.html) o [`passphrase.csv`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/passphrase.csv) per trascriverla
- [ ] Passphrase memorizzata o annotata sul volume crittografato di un dispositivo di archiviazione portatile 
  * `echo $LUKS_PASS` per rivederla; [`passphrase.html`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/passphrase.html) o [`passphrase.csv`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/passphrase.csv) per trascriverla
- [ ] Salvata la chiave di certificazione e le sottochiavi in ​​un archivio portatile crittografato, da conservare offline
  * Si consigliano almeno due backup, archiviati in posizioni separate 
- [ ] Esportata una copia della chiave pubblica a cui è possibile accedere facilmente in seguito
  * È stato utilizzato un dispositivo separato o una partizione non crittografata
- [ ] Memorizzato o annotato il PIN utente e il PIN amministratore, che sono univoci e modificati rispetto ai valori predefiniti 
  * `echo $USER_PIN $ADMIN_PIN` per rivederli; [`passphrase.html`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/passphrase.html) o [`passphrase.csv`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/passphrase.csv) per trascriverli
- [ ]  Spostato le sottochiavi di crittografia, firma e autenticazione sulla YubiKey 
  * `gpg -K` deve mostrare `ssb>` per ognuna delle chiavi

Riavviare il sistema per cancellare l'ambiente temporaneo e completare la configurazione.

# Usa la YubiKey

Inizializza GnuPG:

```console
gpg -k
```

Importa o crea una [configurazione rafforzata](https://github.com/drduh/config/blob/master/gpg.conf):

```console
cd ~/.gnupg

wget https://raw.githubusercontent.com/drduh/config/master/gpg.conf
```

Imposta la seguente opzione. Ciò evita il problema per cui GnuPG richiederà ripetutamente l'inserimento di una YubiKey già inserita: 

```console
touch scdaemon.conf

echo "disable-ccid" >>scdaemon.conf
```

Installa i pacchetti richiesti:


```console
sudo apt update

sudo apt install -y gnupg gnupg-agent scdaemon pcscd
```

Montare il volume non crittografato con la chiave pubblica:

```console
sudo mkdir /mnt/public

sudo mount /dev/sdc2 /mnt/public
```

Importa la chiave pubblica:

```console
gpg --import /mnt/public/*.asc
```

Oppure scarica la chiave pubblica da un keyserver: 

```console
gpg --recv $KEYID
```

Oppure con l'URL su YubiKey, recupera la chiave pubblica: 

```console
gpg/card> fetch

gpg/card> quit
```

Determinare l'ID della chiave: 

```console
gpg -k

export KEYID=0xF0F2CFEB04341FB5
```

Assegna la massima fiducia digitando `trust` e selezionando l'opzione `5` Poi `quit`: 

```console
gpg --command-fd=0 --pinentry-mode=loopback --edit-key $KEYID <<EOF
trust
5
y
save
EOF
```

Rimuovere e reinserire la YubiKey. 

Verificare lo stato con `gpg --card-status` che elencherà le sottochiavi disponibili: 

```console
Reader ...........: Yubico YubiKey OTP FIDO CCID 00 00
Application ID ...: D2760001240102010006055532110000
Application type .: OpenPGP
Version ..........: 3.4
Manufacturer .....: Yubico
Serial number ....: 05553211
Name of cardholder: YubiKey User
Language prefs ...: en
Salutation .......:
URL of public key : [not set]
Login data .......: yubikey@example
Signature PIN ....: not forced
Key attributes ...: rsa4096 rsa4096 rsa4096
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 3 3
Signature counter : 0
KDF setting ......: on
Signature key ....: CF5A 305B 808B 7A0F 230D  A064 B3CD 10E5 02E1 9637
      created ....: 2024-01-01 12:00:00
Encryption key....: A5FA A005 5BED 4DC9 889D  38BC 30CB E8C4 B085 B9F7
      created ....: 2024-01-01 12:00:00
Authentication key: 570E 1355 6D01 4C04 8B6D  E2A3 AD9E 24E1 B8CB 9600
      created ....: 2024-01-01 12:00:00
General key info..: sub  rsa4096/0xB3CD10E502E19637 2024-01-01 YubiKey User <yubikey@example>
sec#  rsa4096/0xF0F2CFEB04341FB5  created: 2024-01-01  expires: never
ssb>  rsa4096/0xB3CD10E502E19637  created: 2024-01-01  expires: 2026-05-01
                                  card-no: 0006 05553211
ssb>  rsa4096/0x30CBE8C4B085B9F7  created: 2024-01-01  expires: 2026-05-01
                                  card-no: 0006 05553211
ssb>  rsa4096/0xAD9E24E1B8CB9600  created: 2024-01-01  expires: 2026-05-01
                                  card-no: 0006 05553211
```

`sec#` indica che la chiave corrispondente non è disponibile (la chiave `Certify` è offline). 

YubiKey è ora pronto per l'uso! 

## Crittografia

Crittografa un messaggio a te stesso (utile per archiviare credenziali o proteggere i backup): 

```console
echo "\ntest message string" | \
  gpg --encrypt --armor \
      --recipient $KEYID --output encrypted.txt
```

Decrittografa il messaggio: verrà visualizzata una richiesta per il PIN utente:

```console
gpg --decrypt --armor encrypted.txt
```

Per crittografare più destinatari/chiavi, impostare per ultimo l'ID della chiave preferita:

```console
echo "test message string" | \
  gpg --encrypt --armor \
      --recipient $KEYID_2 --recipient $KEYID_1 --recipient $KEYID \
      --output encrypted.txt
```

Utilizza una [funzione shell](https://github.com/drduh/config/blob/master/zshrc) per semplificare la crittografia dei file: 

```console
secret () {
  output="${1}".$(date +%s).enc
  gpg --encrypt --armor --output ${output} \
    -r $KEYID "${1}" && echo "${1} -> ${output}"
}

reveal () {
  output=$(echo "${1}" | rev | cut -c16- | rev)
  gpg --decrypt --output ${output} "${1}" && \
    echo "${1} -> ${output}"
}
```

Esempio di output:

```console
$ secret document.pdf
document.pdf -> document.pdf.1580000000.enc

$ reveal document.pdf.1580000000.enc
gpg: anonymous recipient; trying secret key 0xF0F2CFEB04341FB5 ...
gpg: okay, we are the anonymous recipient.
gpg: encrypted with RSA key, ID 0x0000000000000000
document.pdf.1580000000.enc -> document.pdf
```

[drduh/Purse](https://github.com/drduh/Purse) è un gestore di password basato su GnuPG e YubiKey per archiviare e utilizzare in modo sicuro le credenziali.

## Firma

Firma un messaggio:

```console
echo "test message string" | gpg --armor --clearsign > signed.txt
```

Verifica la firma:

```console
gpg --verify signed.txt
```

L'output sarà simile a:

```console
gpg: Signature made Mon 01 Jan 2024 12:00:00 PM UTC
gpg:                using RSA key CF5A305B808B7A0F230DA064B3CD10E502E19637
gpg: Good signature from "YubiKey User <yubikey@example>" [ultimate]
Primary key fingerprint: 4E2C 1FA3 372C BA96 A06A  C34A F0F2 CFEB 0434 1FB5
     Subkey fingerprint: CF5A 305B 808B 7A0F 230D  A064 B3CD 10E5 02E1 9637
```

## Configura il tocco

Per impostazione predefinita, YubiKey eseguirà operazioni crittografiche senza richiedere alcuna azione da parte dell'utente dopo aver sbloccato la chiave una volta con il PIN.

Per richiedere un tocco per ciascuna operazione chiave, utilizzare [YubiKey Manager](https://developers.yubico.com/yubikey-manager/) e il PIN amministratore per impostare la politica delle chiavi.

Crittografia:

```console
ykman openpgp keys set-touch dec on
```

**Nota** Utilizzare le versioni di YubiKey Manager precedenti alla 5.1.0 `enc` invece di `dec` per la crittografia: 


```console
ykman openpgp keys set-touch enc on
```

Anche le versioni precedenti di YubiKey Manager utilizzano `touch` invece di `set-touch`

Firma:

```console
ykman openpgp keys set-touch sig on
```

Autenticazione:

```console
ykman openpgp keys set-touch aut on
```

Per visualizzare e modificare le opzioni delle policy:

```console
ykman openpgp keys set-touch -h
```

`Cached` o `Cached-Fixed` potrebbe essere auspicabile per l'utilizzo di YubiKey con client di posta elettronica.

YubiKey lampeggerà quando attende un tocco. Su Linux, [maximbaz/yubikey-touch-detector](https://github.com/maximbaz/yubikey-touch-detector) può essere utilizzato per indicare che YubiKey è in attesa di un tocco. 

## SSH

A partire dal 2020-05-09 [Filippo Valsorda ](https://filippo.io/) ha rilasciato [ybikey-agent](https://github.com/FiloSottile/yubikey-agent). Ora sto consigliando questo metodo rispetto all'utilizzo di PKCS#11, tuttavia se desideri comunque utilizzare l'ssh-agent nativo, continua a leggere. 

OpenSSH supporta OpenSC dalla versione 5.4. Ciò significa che tutto ciò che devi fare è installare la libreria OpenSC e dire a SSH di utilizzare quella libreria come identità. 

## Prerequisiti

```console
sudo apt-add-repository ppa:yubico/stable
sudo apt update
sudo apt install opensc yubikey-manager
```

## Cambio PIN e PUK

Se si tratta di un nuovo Yubikey, modificare PIN e PUK predefiniti del modulo PIV. 

```console
ykman piv change-management-key --touch --generate
ykman piv change-pin -P 123456
ykman piv change-puk -p 12345678
```

Assicurati di salvare la password generata in un luogo sicuro, ad esempio un gestore di password. La chiave di gestione è necessaria ogni volta che generi una coppia di chiavi, importi un certificato o modifichi il numero di tentativi PIN o PUK.

Anche il PUK dovrebbe essere conservato in un luogo sicuro. Viene utilizzato se il PIN viene inserito in modo errato troppe volte. 

## Creazione del certificato

Questa guida si basa sulle [istruzioni di Yubico](https://developers.yubico.com/PIV/Guides/SSH_with_PIV_and_PKCS11.html) ma utilizza `ykman` invece del veccio `yubico-piv-tool`. Lo strumento precedente non sembra supportare la generazione di certificati PIV e fornisce [errori fuorvianti](https://github.com/Yubico/yubico-piv-tool/issues/153). 

Assicurati che la modalità CCID sia abilitata su Yubikey:

```console
ykman mode
```

Se CCID non è nell'elenco, abilitalo aggiungendo CCID all'elenco, ad esempio:

```console
ykman mode OTP+FIDO+CCID
```

Ciò presuppone che tu avessi OTP+FIDO in precedenza e che li desideri ancora abilitati.

Genera una chiave PIV e genera la chiave pubblica:

```console
ykman piv generate-key 9a pubkey.pem
```

**Nota:** 9a è lo slot di autenticazione PIV.

In alternativa puoi richiedere di toccare la Yubikey ogni volta che si accede allo slot:

```console
ykman piv generate-key --touch-policy always 9a pubkey.pem
```

Per impostazione predefinita, questa è una chiave RSA a 2048 bit. A seconda di quale Yubikey hai, puoi cambiarlo usando `-a` / `--algorithm`.

Utilizzare la chiave `pubkey.pem` appena generata per creare un certificato X.509 autofirmato. L'unico utilizzo del certificato X.509 è agire come identità SSH per la libreria PKCS 11. Deve essere in grado di estrarre la chiave pubblica dalla smartcard e farlo tramite il certificato X.509.

```console
ykman piv generate-certificate -s "SSH key" 9a pubkey.pem
```
- Il parametro `-s "SSH key"` include una descrizione leggibile dall'uomo della chiave o della macchina in cui è installata la chiave.
- Il parametro facoltativo `-d 365` imposta per quanti giorni la chiave sarà valida. Ad esempio il valore *365* specifica che scadrà tra 1 anno.

A questo punto puoi sbarazzarti di pubkey.pem.

Scopri dove è stato installato opensc-pkcs11. Per un sistema basato su Debian, il modulo pkcs11 finisce in /usr/lib/x86_64-linux-gnu.

Esporta la tua chiave pubblica SSH da Yubikey:

```console
ssh-keygen -D /usr/lib/x86_64-linux-gnu/opensc-pkcs11.so
```

**Nota** Questo comando esporterà tutte le chiavi memorizzate su YubiKey. L'ordine degli slot dovrebbe rimanere lo stesso, facilitando così l'identificazione della chiave pubblica associata alla chiave privata di destinazione. 

E queste sono tutte le cose difficili da fare. 


## Carica la chiave pubblica

Carica la chiave chiave pubblica sulla macchina remota:

```console
ssh-copy-id -i ~/.ssh/mykey user@host
```

Se la macchina remota permette di accedere solo tramite chiave `ssh-copy-id` non consente di specificare direttamente una chiave per l'accesso, aggiriamo questo problema usando l'opzione `-o`:

```console
ssh-copy-id -i ~/.ssh/chiave-da-aggiungere -o 'IdentityFile ~/.ssh/chiave-in-uso' user@hostname
```

## Prova ad accedere

Autenticarsi sul sistema di destinazione utilizzando la nuova chiave:

```console
ssh -i /lib/x86_64-linux-gnu/opensc-pkcs11.so user@hostname
```

Facoltativamente può anche essere configurato per funzionare con ssh-agent:

```console
ssh-add -s /lib/x86_64-linux-gnu/opensc-pkcs11.so
```

Conferma che `ssh-agent` trova la chiave corretta e ottiene la chiave pubblica nel formato corretto eseguendo:

```console
ssh-add -L
```

Ora non resta che accedere (senza specificare la chiave) usando:

```console
ssh user@hostname
```

E' possibile utilizzare la chiave SSH di Yubikey, utilizzando l'opzione PKCS11Provider invece di IdentityFile, per esempio:

```console
Host foo
  Hostname <ip o dominio>
  User <username dell'utente remoto>
  PKCS11Provider /usr/local/lib/opensc-pkcs11.so
  IdentitiesOnly yes
```

## Note aggiuntive

- Durante l'utilizzo di SSH, potrebbe essere richiesto il nome dell'oggetto della chiave, ad esempio `Enter PIN for 'SSH key':`. Ma se aggiungi la chiave all'agente, riceverai un messaggio del tipo `Enter passphrase for PKCS#11:`. Si tratta dello stesso PIN (il tuo PIN PIV).

- Se rimuovi la chiave da ssh-agent utilizzando `ssh-add -d` O `ssh-add -D`, dovrai rimuovere e aggiungere nuovamente la libreria PKCS all'agente oppure riavviare l'agente. Per aggiungere nuovamente la libreria:

  ```console
  ssh-add -e /usr/local/lib/opensc-pkcs11.so
  ssh-add -s /usr/local/lib/opensc-pkcs11.so
  ```

**Importante:** Assicurati di avere un piano di backup in atto per quando questo YubiKey fallisce o viene perso. La chiave che abbiamo generato è locale per la YubiKey e non può essere esportata. Può essere facilmente cancellata da chiunque usando `ykman piv reset` (che non richiede autenticazione!), oppure potresti perdere la chiave. Assicurati di avere un altro modo per entrare in tutti gli host con cui usi questo YubiKey (chiave SSH regolare crittografata, console fuori banda, ecc.).


## GitHub

YubiKey può essere utilizzato per firmare commit e tag e autenticare SSH su GitHub quando configurato nelle [impostazioni](https://github.com/settings/keys).

Configura una chiave di firma:

```console
git config --global user.signingkey $KEYID
```

**Importante:** `user.email` deve corrispondere all'indirizzo email associato all'identità PGP.

Per firmare commit o tag utilizzare l'opzione `-S`.

**Nota** Per l'errore `gpg: signing failed: No secret key` - esegui `gpg --card-status` con YubiKey collegato e riprova il comando git.

## Utilizzo di più yubikey

Quando una chiave GnuPG viene aggiunta a YubiKey utilizzando `keytocard`, la chiave viene eliminata dal portachiavi e viene aggiunto uno **stub** che punta alla YubiKey. Lo stub identifica l'ID della chiave GnuPG e il numero di serie YubiKey.

Quando una sottochiave viene aggiunta a un'ulteriore YubiKey, lo stub viene sovrascritto e ora punterà all'ultima YubiKey. GnuPG richiederà una YubiKey specifica in base al numero di serie, come indicato nello stub, e non riconoscerà un'altra YubiKey con un numero di serie diverso.

Per scansionare una YubiKey aggiuntiva e ricreare lo stub corretto: 

```console
gpg-connect-agent "scd serialno" "learn --force" /bye
```

In alternativa, utilizzare uno script per eliminare la chiave nascosta GnuPG, dove è memorizzato il numero di serie della carta (vedi [GnuPG #T2291](https://dev.gnupg.org/T2291)):

```console
cat >> ~/scripts/remove-keygrips.sh <<EOF
#!/usr/bin/env bash
(( $# )) || { echo "Specify a key." >&2; exit 1; }
KEYGRIPS=$(gpg --with-keygrip --list-secret-keys "$@" | awk '/Keygrip/ { print $3 }')
for keygrip in $KEYGRIPS
do
    rm "$HOME/.gnupg/private-keys-v1.d/$keygrip.key" 2> /dev/null
done

gpg --card-status
EOF

chmod +x ~/scripts/remove-keygrips.sh

~/scripts/remove-keygrips.sh $KEYID
```

Consulta la discussione nei numeri [#19](https://github.com/drduh/YubiKey-Guide/issues/19) e [#112](https://github.com/drduh/YubiKey-Guide/issues/112) per ulteriori informazioni e passaggi per la risoluzione dei problemi. 

## Email

YubiKey può essere utilizzato per decrittografare e firmare e-mail e allegati utilizzando  [Thunderbird](https://www.thunderbird.net/), [Enigmail](https://www.enigmail.net) e [Mutt](http://www.mutt.org/). Thunderbird supporta l'autenticazione OAuth 2 e può essere utilizzato con Gmail. Consulta questa [guida](https://ssd.eff.org/en/module/how-use-pgp-linux) per ulteriori informazioni. Mutt ha il supporto OAuth 2 dalla versione 2.0.

### Thunderbird

Segui le [instruzioni sulla wiki di Mozilla](https://wiki.mozilla.org/Thunderbird:OpenPGP:Smartcards#Configure_an_email_account_to_use_an_external_GnuPG_key) per configurare YubiKey con il tuo client Thunderbird utilizzando il provider gpg esterno.

**Importante:** Thunderbird [non riesce](https://github.com/drduh/YubiKey-Guide/issues/448) a decrittografare le email se l'opzione `armor` è abilitata nel tuo `~/.gnupg/gpg.conf`. Se vedi l'errore `gpg: [don't know]: invalid packet (ctb=2d)` o `message cannot be decrypted (there are unknown problems with this encrypted message)` rimuovi semplicemente questa opzione dal tuo file di configurazione.

## Keyserver

Le chiavi pubbliche possono essere caricate su un server pubblico per essere rilevabili:

```console
gpg --send-key $KEYID

gpg --keyserver keys.gnupg.net --send-key $KEYID

gpg --keyserver hkps://keyserver.ubuntu.com:443 --send-key $KEYID
```

Oppure se si caricano su [keys.openpgp.org](https://keys.openpgp.org/about/usage):

```console
gpg --send-key $KEYID | curl -T - https://keys.openpgp.org
```

L'URL della chiave pubblica può anche essere aggiunto a YubiKey (basato su [Shaw 2003](https://datatracker.ietf.org/doc/html/draft-shaw-openpgp-hkp-00)):

```console
URL="hkps://keyserver.ubuntu.com:443/pks/lookup?op=get&search=${KEYID}"
```

Modifica YubiKey con `gpg --edit-card` e il PIN amministratore:

```console
gpg/card> admin

gpg/card> url
URL to retrieve public key: hkps://keyserver.ubuntu.com:443/pks/lookup?op=get&search=0xFF00000000000000

gpg/card> quit
```

# Aggiorna le chiavi

PGP non fornisce [forward secrecy (segretezza in avanti)](https://en.wikipedia.org/wiki/Forward_secrecy), il che significa che una chiave compromessa può essere utilizzata per decrittografare tutti i messaggi passati. Anche se le chiavi memorizzate su YubiKey sono più difficili da sfruttare, non è impossibile: la chiave e il PIN potrebbero essere fisicamente compromessi, oppure potrebbe essere scoperta una vulnerabilità nel firmware o nel generatore di numeri casuali utilizzato per creare le chiavi. Pertanto, si consiglia di ruotare periodicamente le sottochiavi.

Quando una sottochiave scade, può essere rinnovata o sostituita. Entrambe le azioni richiedono l'accesso alla chiave Certify. 

- Il rinnovo delle sottochiavi aggiornando la scadenza indica il possesso continuato della chiave Certify ed è più conveniente. 

- Sostituire le sottochiavi è meno conveniente ma potenzialmente più sicuro: le nuove sottochiavi **non** saranno in grado di decrittografare i messaggi precedenti, autenticarsi con SSH, ecc. I contatti dovranno ricevere la chiave pubblica aggiornata e tutti i segreti crittografati dovranno essere decrittografati e ricodificati con nuove sottochiavi per essere utilizzabili. Questo processo è funzionalmente equivalente alla perdita di YubiKey e al creazione di una nuova. 

Nessuno dei due metodi è superiore all'altro e spetta alla filosofia personale sulla gestione delle identità e sulla modellazione delle minacce individuali decidere quale utilizzare o se far scadere le sottochiavi. Idealmente, le sottochiavi sarebbero effimere: utilizzate solo una volta per ogni evento univoco di crittografia, firma e autenticazione, tuttavia in pratica ciò non è realmente pratico né utile con YubiKey. Gli utenti avanzati possono dedicare una macchina con air gap per la rotazione frequente delle credenziali. 

Per rinnovare o ruotare le sottochiavi, seguire lo stesso processo della generazione delle chiavi: avviare in un ambiente sicuro, installare il software richiesto e disabilitare la rete.

Collegare il dispositivo di archiviazione portatile con la chiave Certify e identificare l'etichetta del disco.

Decrittografa e monta il volume crittografato:

```console
sudo cryptsetup luksOpen /dev/sdc1 gnupg-secrets

sudo mkdir /mnt/encrypted-storage

sudo mount /dev/mapper/gnupg-secrets /mnt/encrypted-storage
```

Montare la partizione pubblica non crittografata:

```console
sudo mkdir /mnt/public

sudo mount /dev/sdc2 /mnt/public
```

Copia i materiali della chiave privata originale in una directory di lavoro temporanea:

```console
export GNUPGHOME=$(mktemp -d -t gnupg-$(date +%Y-%m-%d)-XXXXXXXXXX)

cd $GNUPGHOME

cp -avi /mnt/encrypted-storage/gnupg-*/* $GNUPGHOME
```

Conferma che l'identità è disponibile, imposta l'ID della chiave e l'impronta digitale:

```console
gpg -K

export KEYID=$(gpg -k --with-colons "$IDENTITY" | awk -F: '/^pub:/ { print $5; exit }')

export KEYFP=$(gpg -k --with-colons "$IDENTITY" | awk -F: '/^fpr:/ { print $10; exit }')

echo $KEYID $KEYFP
```

Richiamare la passphrase della chiave Certify e impostarla, ad esempio:

```console
export CERTIFY_PASS=ABCD-0123-IJKL-4567-QRST-UVWX
```

## Rinnova le sottochiavi

Determinare la scadenza aggiornata, ad esempio:

```console
export EXPIRATION=2026-09-01

export EXPIRATION=2y
```

Rinnova le sottochiavi:

```console
gpg --batch --pinentry-mode=loopback \
  --passphrase "$CERTIFY_PASS" --quick-set-expire "$KEYFP" "$EXPIRATION" \
  $(gpg -K --with-colons | awk -F: '/^fpr:/ { print $10 }' | tail -n "+2" | tr "\n" " ")
```

Esporta la chiave pubblica aggiornata:

```console
gpg --armor --export $KEYID | sudo tee /mnt/public/$KEYID-$(date +%F).asc
```

Trasferisci la chiave pubblica all'host di destinazione e importala:

```console
gpg --import /mnt/public/*.asc
```

In alternativa, pubblica su un server la chiave pubblica e scaricala:

```console
gpg --send-key $KEYID

gpg --recv $KEYID
```

La validità dell'identità GnuPG verrà estesa, consentendone il riutilizzo per operazioni di crittografia e firma.

Non è necessario aggiornare la chiave pubblica SSH sugli host remoti.

## Ruota le sottochiavi

Seguire la procedura originale per [Creare sottochiavi](#creare-sottochiavi).

Le sottochiavi precedenti possono essere eliminate dall'identità.

Termina trasferendo le nuove sottochiavi su YubiKey.

Copia la **nuova** directory di lavoro temporanea nell'archivio crittografato, che è ancora montato:

```console
sudo cp -avi $GNUPGHOME /mnt/encrypted-storage
```

Esporta la chiave pubblica aggiornata:

```console
sudo umount /mnt/encrypted-storage

sudo cryptsetup luksClose gnupg-secrets
```

Esporta la chiave pubblica aggiornata:

```console
sudo mkdir /mnt/public

sudo mount /dev/sdc2 /mnt/public

gpg --armor --export $KEYID | sudo tee /mnt/public/$KEYID-$(date +%F).asc

sudo umount /mnt/public
```

Rimuovere il dispositivo di archiviazione e seguire i passaggi originali per trasferire le nuove sottochiavi (`4`, `5` e `6`) a YubiKey, sostituendo quelli esistenti.

Riavvia o cancella in modo sicuro la directory di lavoro temporanea di GnuPG. 

# Reset YubiKey

Se i tentativi del PIN vengono superati, la YubiKey viene bloccata e deve essere [riprstinata](https://developers.yubico.com/ykneo-openpgp/ResetApplet.html) e configurata nuovamente utilizzando il backup crittografato.

Copia quanto segue in un file ed esegui `gpg-connect-agent -r $file` per bloccare e terminare la carta. Quindi reinserisci YubiKey per completare il ripristino. 

```console
/hex
scd serialno
scd apdu 00 20 00 81 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 81 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 81 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 81 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 83 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 83 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 83 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 83 08 40 40 40 40 40 40 40 40
scd apdu 00 e6 00 00
scd apdu 00 44 00 00
/echo Card has been successfully reset.
/bye
```

Oppure utilizzare `ykman` (a volte dentro `~/.local/bin/`):

```console
$ ykman openpgp reset
WARNING! This will delete all stored OpenPGP keys and data and restore factory settings? [y/N]: y
Resetting OpenPGP data, don't remove your YubiKey...
Success! All data has been cleared and default PINs are set.
PIN:         123456
Reset code:  NOT SET
Admin PIN:   12345678
```

# Indurimento opzionale

I seguenti passaggi possono migliorare la sicurezza e la privacy della YubiKey.

## Migliorare l'entropia

La generazione di chiavi crittografiche richiede una [casualità](https://www.random.org/randomness/) misurata come entropia, di alta qualità. La maggior parte dei sistemi operativi utilizza generatori di numeri pseudocasuali basati su software o generatori di numeri casuali hardware basati su CPU (HRNG). 

Facoltativamente, con un dispositivo come [OneRNG](https://onerng.info/onerng/) è possibile [incrementare](https://lwn.net/Articles/648550/) la qualità dell'entropia disponibile.

Prima di creare la chaive, configura [rng-tools](https://wiki.archlinux.org/title/Rng-tools):

```console
sudo apt -y install at rng-tools python3-gnupg openssl

wget https://github.com/OneRNG/onerng.github.io/raw/master/sw/onerng_3.7-1_all.deb
```

Verifica il pacchetto:

```console
sha256sum onerng_3.7-1_all.deb
```

Il valore deve corrispondere:

```console
b7cda2fe07dce219a95dfeabeb5ee0f662f64ba1474f6b9dddacc3e8734d8f57
```

Installa il pacchetto:

```console
sudo dpkg -i onerng_3.7-1_all.deb

echo "HRNGDEVICE=/dev/ttyACM0" | sudo tee /etc/default/rng-tools
```

Inserisci il dispositivo e riavvia rng-tools:

```console
sudo atd

sudo service rng-tools restart
```

## Abilità KDF

**Nota** Questa funzionalità potrebbe non essere compatibile con le versioni precedenti di GnuPG, in particolare con i client mobili. Questi client incompatibili non funzioneranno perché il PIN verrà sempre rifiutato.

Questo passaggio deve essere completato prima di modificare i PIN o spostare le chiavi altrimenti si verificherà un errore: `gpg: error for setup KDF: Conditions of use not satisfied`

La funzione Key Derived (KDF) consente a YubiKey di memorizzare l'hash del PIN, impedendo che il PIN venga passato come testo normale.

Abilita KDF utilizzando il PIN amministratore predefinito `12345678`:

```console
gpg --command-fd=0 --pinentry-mode=loopback --card-edit <<EOF
admin
kdf-setup
12345678
EOF
```

## Network

Questa sezione si concentra principalmente sui sistemi basati su Debian/Ubuntu, ma lo stesso concetto si applica a qualsiasi sistema connesso a una rete.

Che tu stia utilizzando una VM, hardware dedicato o eseguendo temporaneamente un sistema operativo Live, avvia senza una connessione di rete e disabilita tutti i servizi non necessari in ascolto su tutte le interfacce prima di connetterti alla rete.

Il motivo di ciò è perché servizi come cups o avahi possono essere in ascolto per impostazione predefinita. Anche se questo non è un problema immediato, allarga semplicemente la superficie di attacco. Non tutti avranno una sottorete dedicata o apparecchiature di rete affidabili da poter controllare e, ai fini di questa guida, questi passaggi trattano qualsiasi rete come non affidabile/ostile. 

**Disabilita i servizi di ascolto**

- Garantisce che siano in esecuzione solo i servizi di rete essenziali
- Se il servizio non esiste riceverai un messaggio "Impossibile arrestare", il che va bene
- Disabilita il `Bluetooth` se non ne hai bisogno 

```cosole
sudo systemctl stop bluetooth exim4 cups avahi avahi-daemon sshd
```

**Firewall**

Abilita una policy firewall di base che *nega l'ingresso e consente l'uscita*. Tieni presente che Debian non è dotata di firewall, va bene semplicemente disabilitare i servizi nel passaggio precedente. Le seguenti opzioni hanno in mente Ubuntu e sistemi simili.

Su Ubuntu, `ufw` è integrato ed è facile da abilitare: 

```bash
sudo ufw enable
```

Sui sistemi senza `ufw`, `nftables` sta sostituendo `iptables`. La [ wiki di nftables contiene esempi](https://wiki.nftables.org/wiki-nftables/index.php/Simple_ruleset_for_a_workstation) di criteri di base per negare l'ingresso e consentire l'uscita. La policy di `fw.inet.basic` copre sia IPv4 che IPv6.

(Ricordati di scaricare questo README e qualsiasi altra risorsa su un'altra unità esterna durante la creazione del supporto di avvio, per avere queste informazioni pronte per l'uso offline)

Indipendentemente dalla policy che utilizzi, scrivi il contenuto in un file (es. `nftables.conf`) e applicare la policy con il seguente comando:

```bash
sudo nft -f ./nftables.conf
```

**Esaminare lo stato del sistema**

`NetworkManager` dovrebbe essere l'unico servizio in ascolto sulla porta 68/udp a ottenere un lease DHCP (e 58/icmp6 se si dispone di IPv6).

Se vuoi esaminare gli argomenti della riga di comando di ogni processo puoi usare `ps axjf`. Questo stampa un albero del processo che può avere un gran numero di righe ma dovrebbe essere facile da leggere su un'immagine live o su una nuova installazione. 

```bash
sudo ss -anp -A inet    # Dump all network state information
ps axjf                 # List all processes in a process tree
ps aux                  # BSD syntax, list all processes but no process tree
```

Se trovi processi aggiuntivi in ​​ascolto sulla rete che non sono necessari, prendi nota e disabilitali con uno dei seguenti: 

```bash
sudo systemctl stop <process-name>                      # Stops services managed by systemctl
sudo pkill -f '<process-name-or-command-line-string>'   # Terminate the process by matching it's command line string
pgrep -f '<process-name-or-command-line-string>'        # Obtain the PID
sudo kill <pid>                                         # Terminate the process via its PID
```

Ora connettiti a una rete.

# Note

1. YubiKey ha due configurazioni, richiamate con una pressione breve o lunga. Per impostazione predefinita, la modalità pressione breve è configurata per HID OTP; un breve tocco emetterà una stringa OTP che inizia con `cccccccc`. La modalità OTP può essere cambiata nella seconda configurazione tramite lo strumento di personalizzazione YubiKey o disabilitata completamente utilizzando [YubiKey Manager](https://developers.yubico.com/yubikey-manager): `ykman config usb -d OTP`

1. L'uso di YubiKey per GnuPG non impedisce l'uso di [altre funzionalità](https://developers.yubico.com/), come [WebAuthn](https://developers.yubico.com/WebAuthn/) e [OTP](https://developers.yubico.com/OTP/).

1. Aggiungi identità aggiuntive a una chiave Certify con il comando `adduid` durante l'installazione, quindi fidati di esso `trust` e infine `5` per completare la configurazione.

1. Per passare da una YubiKey all'altra, rimuovere la prima YubiKey e riavviare gpg-agent, ssh-agent e pinentry con `pkill "gpg-agent|ssh-agent|pinentry" ; eval $(gpg-agent --daemon --enable-ssh-support)` quindi inserisci l'altra YubiKey ed esegui `gpg-connect-agent updatestartuptty /bye`

1. Per utilizzare YubiKey su più computer, importa le chiavi pubbliche corrispondenti, quindi conferma che YubiKey è visibile con `gpg --card-status`. Alla fine fidati delle chiavi pubbliche importate con `trust` e `5`, poi `gpg --list-secret-keys` mostrerà la chiave corretta e affidabile.

# Risoluzione dei problemi

- Utilizza `man gpg` per comprendere le opzioni di GnuPG e i flag della riga di comando.

- Per ottenere maggiori informazioni sui potenziali errori, riavviare il `gpg-agent`  processo con output di debug sulla console con `pkill gpg-agent; gpg-agent --daemon --no-detach -v -v --debug-level advanced --homedir ~/.gnupg`.

- Molti problemi possono essere risolti rimuovendo e reinserindo YubiKey o riavviando il processo `gpg-agent`.

- Se ricevi l'errore, `Yubikey core error: no yubikey present` - assicurati che la YubiKey sia inserita correttamente. Dovrebbe lampeggiare una volta quando è collegato.

- Se ricevi ancora l'errore, `Yubikey core error: no yubikey present` - Probabilmente dovrai installare le versioni più recenti di ybikey-personalize come indicato in [Installare il software](#installare-il-software).

- Se vedi `General key info..: [none]` nell'output di `gpg card-status`: importa la chiave pubblica.

- Se ricevi l'errore, `gpg: decryption failed: secret key not available` - probabilmente dovrai installare GnuPG versione 2.x. Un'altra possibilità è che ci sia un problema con il PIN, ad esempio che sia troppo corto o bloccato.

- Se ricevi l'errore, `Yubikey core error: write error` - Probabilmente la YubiKey è bloccata. Installa ed esegui ybikey-personalization-gui per sbloccarlo.

- Se ricevi l'errore, `Key does not match the card's capability` - probabilmente dovrai utilizzare dimensioni della chiave RSA a 2048 bit. 

- Se ricevi l'errore, `sign_and_send_pubkey: signing failed: agent refused operation` - assicurati di aver sostituito `ssh-agent` con `gpg-agent` come notato sopra. 

- Se ricevi ancora l'errore, `sign_and_send_pubkey: signing failed: agent refused operation` - [esegui il comando](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=835394) `gpg-connect-agent updatestartuptty /bye`

- Se ricevi ancora l'errore, `sign_and_send_pubkey: signing failed: agent refused operation` - modifica `~/.gnupg/gpg-agent.conf` per impostare un valore valido `pinentry` percorso del programma. `gpg: decryption failed: No secret key` potrebbe anche indicare un sentiero `pinentry` non valido.

- Se ricevi ancora l'errore, `sign_and_send_pubkey: signing failed: agent refused operation` - è un [problema noto](https://bbs.archlinux.org/viewtopic.php?id=274571) che openssh 8.9p1 e versioni successive presentano problemi con YubiKey. Aggiungere `KexAlgorithms -sntrup761x25519-sha512@openssh.com` a `/etc/ssh/ssh_config` spesso risolve il problema.

- Se ricevi l'errore, `The agent has no identities` da `ssh-add -L`, assicurati di aver installato e avviato `scdaemon`

- Se ricevi l'errore, `Error connecting to agent: No such file or directory` da `ssh-add -L`, il socket UNIX utilizzato dall'agente per la comunicazione con altri processi potrebbe non essere impostato correttamente. Su Debian, prova `export SSH_AUTH_SOCK="/run/user/$UID/gnupg/S.gpg-agent.ssh"`. Vedi anche `gpgconf --list-dirs agent-ssh-socket` sta restituendo il percorso unico, all'esistente socket `S.gpg-agent.ssh`.

- Se ricevi l'errore, `Permission denied (publickey)`, aumenta la verbosità di ssh con il file -v contrassegnare e verificare che venga offerta la chiave pubblica della carta: `Offering public key: RSA SHA256:abcdefg... cardno:00060123456`. In tal caso, verificare l'utente corretto del sistema di destinazione, non l'utente del sistema locale. Altrimenti assicurati che `IdentitiesOnly` non sia [abilitato](https://github.com/FiloSottile/whosthere#how-do-i-stop-it) per questo host.

- Se l'autenticazione SSH continua a fallire, aggiungi fino a 3 `-v` bandiere al comando `ssh` per aumentare la verbosità. 

- Se il problema persiste, potrebbe essere utile interrompere lo sfondo `sshd` servizio di processo daemon sul server (ad esempio usando `sudo systemctl stop sshd`) e avviarlo invece in primo piano con un ampio output di debug, utilizzando `/usr/sbin/sshd -eddd`. Tieni presente che il server non si biforcherà ed elaborerà solo una connessione, pertanto dovrà essere riavviato dopo ciascuna `ssh` test. 

- Se ricevi l'errore, `There is no assurance this key belongs to the named user` or `encryption failed: Unusable public key` o `No public key` usa `gpg --edit-key` per impostare `trust` a `5 = I trust ultimately`

- Se, quando provi il comando precedente, ricevi l'errore `Need the secret key to do this` - specificare l'attendibilità per la chiave `~/.gnupg/gpg.conf` utilizzando la diretiva `trust-key [key ID]`.

- Se, quando si utilizza una YubiKey precedentemente fornita su un nuovo computer con `pass`, viene visualizzato il seguente errore su `pass insert`, è necessario modificare l'attendibilità associata alla chiave. Vedi la nota sopra. 

  ```console
  gpg: 0x0000000000000000: There is no assurance this key belongs to the named user
  gpg: [stdin]: encryption failed: Unusable public key
  ```

- Se ricevi l'errore, `gpg: 0x0000000000000000: skipped: Unusable public key`, `signing failed: Unusable secret key`, o `encryption failed: Unusable public key` la sottochiave potrebbe essere scaduta e non può più essere utilizzata per crittografare o firmare messaggi. Può comunque essere utilizzato per decrittografare e autenticare. 

- Se la finestra di dialogo grafica della _pinentry_ non viene visualizzata e viene visualizzato questo errore: `sign_and_send_pubkey: signing failed: agent refused operation`, installare il pacchetto `dbus-user-session` e riavviare per il `dbus` sessione utente da ereditare completamente. Questo è perché pinentry si lamenta `No $DBUS_SESSION_BUS_ADDRESS found`, ritorna a `curses` ma non trova quello atteso `tty`.

- Se, quando provi quanto sopra `--card-status` comando, riceverai l'errore, `gpg: selecting card failed: No such device` o `gpg: OpenPGP card not available: No such device`, è possibile che l'ultima versione di pcscd richieda ora le regole polkit per funzionare correttamente. Crea il seguente file per consentire agli utenti di accedere al file `wheel` gruppo per utilizzare la carta. Assicurati di riavviare pcscd quando hai finito per consentire alle nuove regole di avere effetto. 

  ```console
  cat << EOF >  /etc/polkit-1/rules.d/99-pcscd.rules
  polkit.addRule(function(action, subject) {
          if (action.id == "org.debian.pcsc-lite.access_card" &&
                  subject.isInGroup("wheel")) {
                  return polkit.Result.YES;
          }
  });
  polkit.addRule(function(action, subject) {
          if (action.id == "org.debian.pcsc-lite.access_pcsc" &&
                  subject.isInGroup("wheel")) {
                  return polkit.Result.YES;
          }
  });
  EOF
  ```

- Se la chiave pubblica viene persa, segui [questa guida](https://www.nicksherlock.com/2021/08/recovering-lost-gpg-public-keys-from-your-yubikey/) per recuperarla da YubiKey

- Fare riferimento all'articolo [Yubico Risoluzione dei problemi con GPG](https://support.yubico.com/hc/en-us/articles/360013714479-Troubleshooting-Issues-with-GPG) per ulteriori indicazioni. 

# Soluzioni alternative

* [`drduh/YubiKey-Guide`](https://github.com/drduh/YubiKey-Guide) - la guida principale utilizzata per redirigere questa guida
* [`jamesog/yubikey-ssh`](https://github.com/jamesog/yubikey-ssh) - un ottima guida che spiega come usare ssh con il modulo PIV della Yubikey
* [`vorburger/ed25519-sk.md`](https://github.com/vorburger/vorburger.ch-Notes/blob/develop/security/ed25519-sk.md) -  usa YubiKey per SSH senza GnuPG 
* [`smlx/piv-agent`](https://github.com/smlx/piv-agent) - Agente SSH e GnuPG che può essere utilizzato con dispositivi PIV 
* [`keytotpm`](https://www.gnupg.org/documentation/manuals/gnupg/OpenPGP-Key-Management.html) - utilizzare GnuPG con sistemi TPM

# Risorse aggiuntive

* [Yubico - PGP](https://developers.yubico.com/PGP/)
* [Yubico - Yubikey Personalization](https://developers.yubico.com/yubikey-personalization/)
* [A Visual Explanation of GPG Subkeys (2022)](https://rgoulter.com/blog/posts/programming/2022-06-10-a-visual-explanation-of-gpg-subkeys.html)
* [dhess/nixos-yubikey](https://github.com/dhess/nixos-yubikey)
* [lsasolutions/makegpg](https://gitlab.com/lsasolutions/makegpg)
* [Trammell Hudson - Yubikey (2020)](https://trmm.net/Yubikey)
* [Yubikey forwarding SSH keys (2019)](https://blog.onefellow.com/post/180065697833/yubikey-forwarding-ssh-keys)
* [GPG Agent Forwarding (2018)](https://mlohr.com/gpg-agent-forwarding/)
* [Stick with security: YubiKey, SSH, GnuPG, macOS (2018)](https://evilmartians.com/chronicles/stick-with-security-yubikey-ssh-gnupg-macos)
* [PGP and SSH keys on a Yubikey NEO (2015)](https://www.esev.com/blog/post/2015-01-pgp-ssh-key-on-yubikey-neo/)
* [Offline GnuPG Master Key and Subkeys on YubiKey NEO Smartcard (2014)](https://blog.josefsson.org/2014/06/23/offline-gnupg-master-key-and-subkeys-on-yubikey-neo-smartcard/)
* [Creating the perfect GPG keypair (2013)](https://alexcabal.com/creating-the-perfect-gpg-keypair/)
