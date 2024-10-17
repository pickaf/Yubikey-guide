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
- [Creare sottochiavi](#creare-sottochiavi)
- [Verificare le chiavi](#verificare-le-chiavi)
- [Backup delle chavi](#backup-delle-chiavi)
- [Esporta la chiave pubblica](#esporta-la-chiave-pubblica)
- [Configura YubiKey](#configura-yubikey)
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
      + [Copy public key](#copy-public-key)
      + [Import SSH keys](#import-ssh-keys)
      + [SSH agent forwarding](#ssh-agent-forwarding)
         - [Use ssh-agent](#use-ssh-agent)
         - [Use S.gpg-agent.ssh](#use-sgpg-agentssh)
         - [Chained forwarding](#chained-forwarding)
   * [GitHub](#github)
   * [GnuPG agent forwarding](#gnupg-agent-forwarding)
      + [Legacy distributions](#legacy-distributions)
      + [Chained GnuPG agent forwarding](#chained-gnupg-agent-forwarding)
   * [Using multiple YubiKeys](#using-multiple-yubikeys)
   * [Email](#email)
      + [Thunderbird](#thunderbird)
      + [Mailvelope](#mailvelope)
      + [Mutt](#mutt)
   * [Keyserver](#keyserver)
- [Aggiornamento delle chaivi](#updating-keys)
   * [Rinnova le sottochiavi](#renew-subkeys)
   * [Ruota le sottochaivi](#rotate-subkeys)
- [Reset YubiKey](#reset-yubikey)
- [Optional hardening](#optional-hardening)
   * [Improving entropy](#improving-entropy)
   * [Enable KDF](#enable-kdf)
   * [Network considerations](#network-considerations)
- [Note](#notes)
- [Risoluzione dei problemi](#troubleshooting)
- [Soluzioni alternative](#alternative-solutions)
- [Risorse aggiuntive](#additional-resources)

# Acquistare Yubikey

[Le attuli Yubikey](https://www.yubico.com/store/compare/) ad eccezione della serie di chiavi di sicurezza solo FIDO e delle YubiKey della serie Bio, sono compatibili con questa guida.

[Verifica Yubikey](https://support.yubico.com/hc/en-us/articles/360013723419-How-to-Confirm-Your-Yubico-Device-is-Genuine) visitando [yubico.com/genuine](https://www.yubico.com/genuine/). Seleziona *Verify Device* per iniziare il processo. Tocca YubiKey quando richiesto e consenti al sito di vedere la marca e il modello del dispositivo quando richiesto. Questa attestazione del dispositivo può aiutare a mitigare [supply chain attacks](https://media.defcon.org/DEF%20CON%2025/DEF%20CON%2025%20presentations/DEF%20CON%2025%20-%20r00killah-and-securelyfitz-Secure-Tokin-and-Doobiekeys.pdf).

Si consigliano inoltre diversi dispositivi di archiviazione portatili (come le schede microSD) per l'archiviazione di backup crittografati.

# Preparazione

Si consiglia un ambiente operativo dedicato e sicuro per generare chiavi crittografiche.

In questa guida useremo Debian Live perchè bilancia usabilità e sicurezza.

Scarica i file d' immagine e firma più recenti: 

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

Verifica la fir:

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

**Note** Se lo schermo si blocca su Debian Live, sbloccalo usando `user` / `live`

Apri il terminale e installa i pacchetti software richiesti.

**Debian/Ubuntu**

```console
sudo apt update

sudo apt -y upgrade

sudo apt -y install \
  wget gnupg2 gnupg-agent dirmngr \
  cryptsetup scdaemon pcscd \
  yubikey-personalization yubikey-manager
```

**Note**  Per l'utilizzo potrebbe essere necessario installare una dipendenza aggiuntiva del pacchetto python [`ykman`](https://support.yubico.com/support/solutions/articles/15000012643-yubikey-manager-cli-ykman-user-guide) - `pip install yubikey-manager`

**Fedora**

```console
sudo dnf install wget

wget https://github.com/rpmsphere/noarch/raw/master/r/rpmsphere-release-38-1.noarch.rpm

sudo rpm -Uvh rpmsphere-release*rpm

sudo dnf install \
  gnupg2 dirmngr cryptsetup gnupg2-smime \
  pcsc-tools opensc pcsc-lite secure-delete \
  pgp-tools yubikey-personalization-gui
```

# Preparare GnuPG

Crea una directory temporanea che verrà cancellata al [riavvio](https://en.wikipedia.org/wiki/Tmpfs) e impostala come directory GnuPG:

```console
export GNUPGHOME=$(mktemp -d -t gnupg-$(date +%Y-%m-%d)-XXXXXXXXXX)
```

## Configurazione

Importa orcrea una [configurazione rafforzata](https://github.com/drduh/config/blob/master/gpg.conf):

```console
cd $GNUPGHOME

wget https://raw.githubusercontent.com/drduh/config/master/gpg.conf
```

**Note** La rete può essere disabilitata per il resto della configurazione.

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

Le sottochiavi devono essere rinnovate o ruotate utilizzando la chiave Certifica: vedere Aggiornamento delle chiavi.

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

Questo repository include il modello [`passphrase.html`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/passphrase.html) per facilitare la trascrizione delle credenziali. Salva il file raw, aprilo con un browser e stampa. Utilizzare una penna o un pennarello indelebile per selezionare una lettera o un numero su ciascuna riga per ciascun carattere della passphrase. [`passphrase.csv`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/passphrase.csv) può essere stampato anche senza browser:

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

# Creare sottochiavi

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

**Attezione** Confermare la destinazione ( `of `) prima di emettere il comando seguente - è distruttivo! Questa guida utilizza `/dev/sdc` in tutto, ma questo valore potrebbe essere diverso sul tuo sistema. 

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

Genera un'altra [Passphrase](#passphrase) (ideally different from the one used for the Certify key) univoca per proteggere il volume crittografato:

```console
export LUKS_PASS=$(LC_ALL=C tr -dc 'A-Z1-9' < /dev/urandom | \
  tr -d "1IOS5U" | fold -w 30 | sed "-es/./ /"{1..26..5} | \
  cut -c2- | tr " " "-" | head -1) ; printf "\n$LUKS_PASS\n\n"
```

Questa passphrase verrà utilizzata anche raramente per accedere alla chiave Certify e dovrebbe essere molto complessa.

Annotare la passphrase o memorizzarla. 

Formatta la partizione:

```console
echo $LUKS_PASS | sudo cryptsetup -q luksFormat /dev/sdc1
```

Mounta la partizione:

```console
echo $LUKS_PASS | sudo cryptsetup -q luksOpen /dev/sdc1 gnupg-secrets
```

Crea un file system ext2:

```console
sudo mkfs.ext2 /dev/mapper/gnupg-secrets -L gnupg-$(date +%F)
```

Monta il filesystem e copia la directory di lavoro temporanea di GnuPG con i materiali chiave:

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

**Importante** Senza la chiave pubblica, non sarà possibile utilizzare GnuPG per decrittografare o firmare i messaggi. Tuttavia, YubiKey può ancora essere utilizzata per l'autenticazione SSH. 

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

# Configura YubiKey

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

**Avviso** sbagliati tre *PIN utente* immessi ne causeranno il blocco e dovranno essere sbloccati con il *PIN amministratore* o con il codice di ripristino. Tre *PIN amministratore* o codici di reimpostazione errati distruggeranno i dati sulla YubiKey. 

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

**Importante**  Il trasferimento delle chiavi su YubiKey è un'operazione unidirezionale che converte la chiave su disco in uno stub rendendola non più utilizzabile per il trasferimento su YubiKey successive. Assicurarsi che sia stato effettuato un backup prima di procedere. 

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

Verifica che le sottochiavi siano state spostate su YubiKey con `gpg -K` e creare `ssb>`, per esempio:

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
- [ ] Passphrase memorizzata o annotata sul volume crittografato su un dispositivo di archiviazione portatile 
  * `echo $LUKS_PASS` per rivederla; [`passphrase.html`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/passphrase.html) o [`passphrase.csv`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/passphrase.csv) per trascriverla
- [ ] Salvata la chiave di certificazione e le sottochiavi in ​​un archivio portatile crittografato, da conservare offline
  * Si consigliano almeno due backup, archiviati in posizioni separate 
- [ ] Esportata una copia della chiave pubblica a cui è possibile accedere facilmente in seguito
  * È stato utilizzato un dispositivo separato o una partizione non crittografata
- [ ] Memorizzato o annotare il PIN utente e il PIN amministratore, che sono univoci e modificati rispetto ai valori predefiniti 
  * `echo $USER_PIN $ADMIN_PIN` per rivederli; [`passphrase.html`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/passphrase.html) o [`passphrase.csv`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/passphrase.csv) per trascriverli
- [ ]  Spostate le sottochiavi di crittografia, firma e autenticazione sulla YubiKey 
  * `gpg -K` deve mostrare `ssb>` per ognuna delle chiavi

Riavviare per cancellare l'ambiente temporaneo e completare la configurazione.

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

**Debian/Ubuntu**

```console
sudo apt update

sudo apt install -y gnupg gnupg-agent scdaemon pcscd
```

Montare il volume non crittografato con la chiave pubblica:

**Debian/Ubuntu**

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

Rimuovere e reinserire YubiKey. 

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

# SSH

A partire dal 2020-05-09 [Filippo Valsorda ](https://filippo.io/) ha rilasciato [ybikey-agent](https://github.com/FiloSottile/yubikey-agent). Ora sto consigliando questo metodo rispetto all'utilizzo di PKCS#11, tuttavia se desideri comunque utilizzare l'ssh-agent nativo, continua a leggere. 

OpenSSH supporta OpenSC dalla versione 5.4. Ciò significa che tutto ciò che devi fare è installare la libreria OpenSC e dire a SSH di utilizzare quella libreria come identità. 

## Prerequisiti

```console
sudo apt-add-repository ppa:yubico/stable
sudo apt update
sudo apt install opensc yubikey-manager
```

## Cambio PIN e PUK

Se si tratta di un nuovo Yubikey, modificare la chiave di gestione PIV predefinita, PIN e PUK. 

```console
ykman piv change-management-key --touch --generate
ykman piv change-pin -P 123456
ykman piv change-puk -p 12345678
```

Assicurati di salvare la password generata in un luogo sicuro, ad esempio un gestore di password. La chiave di gestione è necessaria ogni volta che generi una coppia di chiavi, importi un certificato o modifichi il numero di tentativi PIN o PUK.

Anche il PUK dovrebbe essere conservato in un luogo sicuro. Viene utilizzato se il PIN viene inserito in modo errato troppe volte. 

## Creazione certificato

Questa guida si basa sulle [istruzioni di Yubico](https://developers.yubico.com/PIV/Guides/SSH_with_PIV_and_PKCS11.html) ma utilizza `ykman` invece del veccio `yubico-piv-tool`. Lo strumento precedente non sembra supportare la generazione di certificati PIV e fornisce [errori fuorvianti](https://github.com/Yubico/yubico-piv-tool/issues/153). 

Assicurati che la modalità CCID sia abilitata su Yubikey:

```console
ykman mode
```

Se CCID non è nell'elenco, abilitalo aggiungendo CCID all'elenco, ad esempio:

```console
ykman mode OTP+FIDO+CCID
```

(Ciò presuppone che tu avessi OTP+FIDO in precedenza e che li desideri ancora abilitati.)

Genera una chiave PIV e genera la chiave pubblica :

```console
ykman piv generate-key 9a pubkey.pem
```

**Nota:** 9a è lo slot di autenticazione PIV.

In alternativa puoi richiedere di toccare la Yubikey ogni volta che si accede allo slot:

```console
ykman piv generate-key --touch-policy always 9a pubkey.pem
```

Per impostazione predefinita, questa è una chiave RSA a 2048 bit. A seconda di quale Yubikey hai, puoi cambiarlo usando `-a` / `--algorithm`.

Genera un certificato X.509 autofirmato per quella chiave. L'unico utilizzo del certificato X.509 è soddisfare PIV/PKCS #11 lib. Deve essere in grado di estrarre la chiave pubblica dalla smartcard e farlo tramite il certificato X.509.

```console
ykman piv generate-certificate -s "SSH key" 9a pubkey.pem
```

Scopri dove è stato installato opensc-pkcs11. Per un sistema basato su Debian, il modulo pkcs11 finisce in /usr/lib/x86_64-linux-gnu.

Esporta la tua chiave pubblica SSH da Yubikey:

```console
ssh-keygen -D /usr/lib/x86_64-linux-gnu/opensc-pkcs11.so
```

E queste sono tutte le cose difficili da fare. 

Esporta la chiave pubblica nel formato corretto per SSH, quindi aggiungila ad `Authorized_keys` sul sistema di destinazione e prova ad autenticarti:

```console
ssh -I XXX/opensc-pkcs11.so -i XXX/opensc-pkcs11.so -o IdentitiesOnly=yes utente@server.example.com
```

Dovrebbe essere richiesto per il tuo PIN PIV di Yubikey.

**Nota:** Questo comando esporterà tutte le chiavi memorizzate su YubiKey. L'ordine degli slot dovrebbe rimanere lo stesso, facilitando così l'identificazione della chiave pubblica associata alla chiave privata di destinazione. 

## GitHub

YubiKey can be used to sign commits and tags, and authenticate SSH to GitHub when configured in [Settings](https://github.com/settings/keys).

Configure a signing key:

```console
git config --global user.signingkey $KEYID
```

**Important** The `user.email` option must match the email address associated with the PGP identity.

To sign commits or tags, use the `-S` option.

**Windows**

Configure authentication:

```console
git config --global core.sshcommand "plink -agent"

git config --global gpg.program 'C:\Program Files (x86)\GnuPG\bin\gpg.exe'
```

Then update the repository URL to `git@github.com:USERNAME/repository`

**Note** For the error `gpg: signing failed: No secret key` - run `gpg --card-status` with YubiKey plugged in and try the git command again.

## GnuPG agent forwarding

YubiKey can be used sign git commits and decrypt files on remote hosts with GnuPG Agent Forwarding. To ssh through another network, especially to push to/pull from GitHub using ssh, see [Remote Machines (SSH Agent forwarding)](#ssh-agent-forwarding).

`gpg-agent.conf` is not needed on the remote host; after forwarding, remote GnuPG directly communicates with `S.gpg-agent` without starting `gpg-agent` on the remote host.

On the remote host, edit `/etc/ssh/sshd_config` to set `StreamLocalBindUnlink yes`

**Optional** Without root access on the remote host to edit `/etc/ssh/sshd_config`, socket located at `gpgconf --list-dir agent-socket` on the remote host will need to be removed before forwarding works. See [AgentForwarding GNUPG wiki page](https://wiki.gnupg.org/AgentForwarding) for more information.

Import the public key on the remote host. On the local host, copy the public keyring to the remote host:

```console
scp ~/.gnupg/pubring.kbx remote:~/.gnupg/
```

On modern distributions, such as Fedora 30, there is no need to set `RemoteForward` in `~/.ssh/config`

### Legacy distributions

On the local host, run:

```console
gpgconf --list-dirs agent-extra-socket
```

This should return a path to agent-extra-socket - `/run/user/1000/gnupg/S.gpg-agent.extra` - though on older Linux distros (and macOS) it may be `/home/<user>/.gnupg/S/gpg-agent.extra`

Find the agent socket on the **remote** host:

```console
gpgconf --list-dirs agent-socket
```

This should return a path such as `/run/user/1000/gnupg/S.gpg-agent`

Finally, enable agent forwarding for a given host by adding the following to the local host's `~/.ssh/config` (agent sockets may differ):

```
Host
  Hostname remote-host.tld
  StreamLocalBindUnlink yes
  RemoteForward /run/user/1000/gnupg/S.gpg-agent /run/user/1000/gnupg/S.gpg-agent.extra
  #RemoteForward [remote socket] [local socket]
```

It may be necessary to edit `gpg-agent.conf` on the *local* host to add the following information:

```
pinentry-program /usr/bin/pinentry-gtk-2
extra-socket /run/user/1000/gnupg/S.gpg-agent.extra
```

**Note** The pinentry program starts on the *local* host, not remote.

**Important** Any pinentry program except `pinentry-tty` or `pinentry-curses` may be used. This is because local `gpg-agent` may start headlessly (by systemd without `$GPG_TTY` set locally telling which tty it is on), thus failed to obtain the pin. Errors on the remote may be misleading saying that there is *IO Error*. (Yes, internally there is actually an *IO Error* since it happens when writing to/reading from tty while finding no tty to use, but for end users this is not friendly.)

See [Issue 85](https://github.com/drduh/YubiKey-Guide/issues/85) for more information and troubleshooting.

### Chained GnuPG agent forwarding

Assume you have gone through the steps above and have `S.gpg-agent` on the *remote*, and you would like to forward this agent into a *third* box, first you may need to configure `sshd_config` of *third* in the same way as *remote*, then in the ssh config of *remote*, add the following lines:

```console
Host third
  Hostname third-host.tld
  StreamLocalBindUnlink yes
  RemoteForward /run/user/1000/gnupg/S.gpg-agent /run/user/1000/gnupg/S.gpg-agent
  #RemoteForward [remote socket] [local socket]
```

You should change the path according to `gpgconf --list-dirs agent-socket` on *remote* and *third*.

**Note** On *local* you have `S.gpg-agent.extra` whereas on *remote* and *third*, you only have `S.gpg-agent`

## Using multiple YubiKeys

When a GnuPG key is added to YubiKey using `keytocard`, the key is deleted from the keyring and a **stub** is added, pointing to the YubiKey. The stub identifies the GnuPG key ID and YubiKey serial number.

When a Subkey is added to an additional YubiKey, the stub is overwritten and will now point to the latest YubiKey. GnuPG will request a specific YubiKey by serial number, as referenced by the stub, and will not recognize another YubiKey with a different serial number.

To scan an additional YubiKey and recreate the correct stub:

```console
gpg-connect-agent "scd serialno" "learn --force" /bye
```

Alternatively, use a script to delete the GnuPG shadowed key, where the card serial number is stored (see [GnuPG #T2291](https://dev.gnupg.org/T2291)):

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

See discussion in Issues [#19](https://github.com/drduh/YubiKey-Guide/issues/19) and [#112](https://github.com/drduh/YubiKey-Guide/issues/112) for more information and troubleshooting steps.

## Email

YubiKey can be used to decrypt and sign emails and attachments using [Thunderbird](https://www.thunderbird.net/), [Enigmail](https://www.enigmail.net) and [Mutt](http://www.mutt.org/). Thunderbird supports OAuth 2 authentication and can be used with Gmail. See [this EFF guide](https://ssd.eff.org/en/module/how-use-pgp-linux) for more information. Mutt has OAuth 2 support since version 2.0.

### Thunderbird

Follow [instructions on the mozilla wiki](https://wiki.mozilla.org/Thunderbird:OpenPGP:Smartcards#Configure_an_email_account_to_use_an_external_GnuPG_key) to setup your YubiKey with your thunderbird client using the external gpg provider.

**Important** Thunderbird [fails](https://github.com/drduh/YubiKey-Guide/issues/448) to decrypt emails if the ASCII `armor` option is enabled in your `~/.gnupg/gpg.conf`. If you see the error `gpg: [don't know]: invalid packet (ctb=2d)` or `message cannot be decrypted (there are unknown problems with this encrypted message)` simply remove this option from your config file.

### Mailvelope

[Mailvelope](https://www.mailvelope.com/en) allows YubiKey to be used with Gmail and others.

**Important** Mailvelope [does not work](https://github.com/drduh/YubiKey-Guide/issues/178) with the `throw-keyids` option set in `gpg.conf`

On macOS, install gpgme using Homebrew:

```console
brew install gpgme
```

To allow Chrome to run gpgme, edit `~/Library/Application\ Support/Google/Chrome/NativeMessagingHosts/gpgmejson.json` to add:

```json
{
    "name": "gpgmejson",
    "description": "Integration with GnuPG",
    "path": "/usr/local/bin/gpgme-json",
    "type": "stdio",
    "allowed_origins": [
        "chrome-extension://kajibbejlbohfaggdiogboambcijhkke/"
    ]
}
```

Edit the default path to allow Chrome to find GnuPG:

```console
sudo launchctl config user path /usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
```

Finally, install the [Mailvelope extension](https://chromewebstore.google.com/detail/mailvelope/kajibbejlbohfaggdiogboambcijhkke) from the Chrome web store.

### Mutt

Mutt has both CLI and TUI interfaces - the latter provides powerful functions for processing email. In addition, PGP can be integrated such that cryptographic operations can be done without leaving TUI.

To enable GnuPG support, copy `/usr/share/doc/mutt/samples/gpg.rc`

Edit the file to enable options `pgp_default_key`, `pgp_sign_as` and `pgp_autosign`

`source` the file in `muttrc`

**Important** `pinentry-tty` set as the pinentry program in `gpg-agent.conf` is reported to cause problems with Mutt TUI, because it uses curses. It is recommended to use `pinentry-curses` or other graphic pinentry program instead.

## Keyserver

Public keys can be uploaded to a public server for discoverability:

```console
gpg --send-key $KEYID

gpg --keyserver keys.gnupg.net --send-key $KEYID

gpg --keyserver hkps://keyserver.ubuntu.com:443 --send-key $KEYID
```

Or if [uploading to keys.openpgp.org](https://keys.openpgp.org/about/usage):

```console
gpg --send-key $KEYID | curl -T - https://keys.openpgp.org
```

The public key URL can also be added to YubiKey (based on [Shaw 2003](https://datatracker.ietf.org/doc/html/draft-shaw-openpgp-hkp-00)):

```console
URL="hkps://keyserver.ubuntu.com:443/pks/lookup?op=get&search=${KEYID}"
```

Edit YubiKey with `gpg --edit-card` and the Admin PIN:

```console
gpg/card> admin

gpg/card> url
URL to retrieve public key: hkps://keyserver.ubuntu.com:443/pks/lookup?op=get&search=0xFF00000000000000

gpg/card> quit
```

# Updating keys

PGP does not provide [forward secrecy](https://en.wikipedia.org/wiki/Forward_secrecy), meaning a compromised key may be used to decrypt all past messages. Although keys stored on YubiKey are more difficult to exploit, it is not impossible: the key and PIN could be physically compromised, or a vulnerability may be discovered in firmware or in the random number generator used to create keys, for example. Therefore, it is recommended practice to rotate Subkeys periodically.

When a Subkey expires, it can either be renewed or replaced. Both actions require access to the Certify key.

- Renewing Subkeys by updating expiration indicates continued possession of the Certify key and is more convenient.

- Replacing Subkeys is less convenient but potentially more secure: the new Subkeys will **not** be able to decrypt previous messages, authenticate with SSH, etc. Contacts will need to receive the updated public key and any encrypted secrets need to be decrypted and re-encrypted to new Subkeys to be usable. This process is functionally equivalent to losing the YubiKey and provisioning a new one.

Neither rotation method is superior and it is up to personal philosophy on identity management and individual threat modeling to decide which one to use, or whether to expire Subkeys at all. Ideally, Subkeys would be ephemeral: used only once for each unique encryption, signature and authentication event, however in practice that is not really practical nor worthwhile with YubiKey. Advanced users may dedicate an air-gapped machine for frequent credential rotation.

To renew or rotate Subkeys, follow the same process as generating keys: boot to a secure environment, install required software and disable networking.

Connect the portable storage device with the Certify key and identify the disk label.

Decrypt and mount the encrypted volume:

```console
sudo cryptsetup luksOpen /dev/sdc1 gnupg-secrets

sudo mkdir /mnt/encrypted-storage

sudo mount /dev/mapper/gnupg-secrets /mnt/encrypted-storage
```

Mount the non-encrypted public partition:

```console
sudo mkdir /mnt/public

sudo mount /dev/sdc2 /mnt/public
```

Copy the original private key materials to a temporary working directory:

```console
export GNUPGHOME=$(mktemp -d -t gnupg-$(date +%Y-%m-%d)-XXXXXXXXXX)

cd $GNUPGHOME

cp -avi /mnt/encrypted-storage/gnupg-*/* $GNUPGHOME
```

Confirm the identity is available, set the key id and fingerprint:

```console
gpg -K

export KEYID=$(gpg -k --with-colons "$IDENTITY" | awk -F: '/^pub:/ { print $5; exit }')

export KEYFP=$(gpg -k --with-colons "$IDENTITY" | awk -F: '/^fpr:/ { print $10; exit }')

echo $KEYID $KEYFP
```

Recall the Certify key passphrase and set it, for example:

```console
export CERTIFY_PASS=ABCD-0123-IJKL-4567-QRST-UVWX
```

## Renew Subkeys

Determine the updated expiration, for example:

```console
export EXPIRATION=2026-09-01

export EXPIRATION=2y
```

Renew the Subkeys:

```console
gpg --batch --pinentry-mode=loopback \
  --passphrase "$CERTIFY_PASS" --quick-set-expire "$KEYFP" "$EXPIRATION" \
  $(gpg -K --with-colons | awk -F: '/^fpr:/ { print $10 }' | tail -n "+2" | tr "\n" " ")
```

Export the updated public key:

```console
gpg --armor --export $KEYID | sudo tee /mnt/public/$KEYID-$(date +%F).asc
```

Transfer the public key to the destination host and import it:

```console
gpg --import /mnt/public/*.asc
```

Alternatively, publish to a public key server and download it:

```console
gpg --send-key $KEYID

gpg --recv $KEYID
```

The validity of the GnuPG identity will be extended, allowing it to be used again for encryption and signature operations.

The SSH public key does **not** need to be updated on remote hosts.

## Rotate Subkeys

Follow the original procedure to [Create Subkeys](#create-subkeys).

Previous Subkeys can be deleted from the identity.

Finish by transfering new Subkeys to YubiKey.

Copy the **new** temporary working directory to encrypted storage, which is still mounted:

```console
sudo cp -avi $GNUPGHOME /mnt/encrypted-storage
```

Unmount and close the encrypted volume:

```console
sudo umount /mnt/encrypted-storage

sudo cryptsetup luksClose gnupg-secrets
```

Export the updated public key:

```console
sudo mkdir /mnt/public

sudo mount /dev/sdc2 /mnt/public

gpg --armor --export $KEYID | sudo tee /mnt/public/$KEYID-$(date +%F).asc

sudo umount /mnt/public
```

Remove the storage device and follow the original steps to transfer new Subkeys (`4`, `5` and `6`) to YubiKey, replacing existing ones.

Reboot or securely erase the GnuPG temporary working directory.

# Reset YubiKey

If PIN attempts are exceeded, the YubiKey is locked and must be [Reset](https://developers.yubico.com/ykneo-openpgp/ResetApplet.html) and set up again using the encrypted backup.

Copy the following to a file and run `gpg-connect-agent -r $file` to lock and terminate the card. Then re-insert YubiKey to complete reset.

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

Or use `ykman` (sometimes in `~/.local/bin/`):

```console
$ ykman openpgp reset
WARNING! This will delete all stored OpenPGP keys and data and restore factory settings? [y/N]: y
Resetting OpenPGP data, don't remove your YubiKey...
Success! All data has been cleared and default PINs are set.
PIN:         123456
Reset code:  NOT SET
Admin PIN:   12345678
```

# Optional hardening

The following steps may improve the security and privacy of YubiKey.

## Improving entropy

Generating cryptographic keys requires high-quality [randomness](https://www.random.org/randomness/), measured as entropy. Most operating systems use software-based pseudorandom number generators or CPU-based hardware random number generators (HRNG).

Optionally, a device such as [OneRNG](https://onerng.info/onerng/) may be used to [increase the speed](https://lwn.net/Articles/648550/) and possibly the quality of available entropy.

Before creating keys, configure [rng-tools](https://wiki.archlinux.org/title/Rng-tools):

```console
sudo apt -y install at rng-tools python3-gnupg openssl

wget https://github.com/OneRNG/onerng.github.io/raw/master/sw/onerng_3.7-1_all.deb
```

Verify the package:

```console
sha256sum onerng_3.7-1_all.deb
```

The value must match:

```console
b7cda2fe07dce219a95dfeabeb5ee0f662f64ba1474f6b9dddacc3e8734d8f57
```

Install the package:

```console
sudo dpkg -i onerng_3.7-1_all.deb

echo "HRNGDEVICE=/dev/ttyACM0" | sudo tee /etc/default/rng-tools
```

Insert the device and restart rng-tools:

```console
sudo atd

sudo service rng-tools restart
```

## Enable KDF

**Note** This feature may not be compatible with older GnuPG versions, especially mobile clients. These incompatible clients will not function because the PIN will always be rejected.

This step must be completed before changing PINs or moving keys or an error will occur: `gpg: error for setup KDF: Conditions of use not satisfied`

Key Derived Function (KDF) enables YubiKey to store the hash of PIN, preventing the PIN from being passed as plain text.

Enable KDF using the default Admin PIN of `12345678`:

```console
gpg --command-fd=0 --pinentry-mode=loopback --card-edit <<EOF
admin
kdf-setup
12345678
EOF
```

## Network considerations

This section is primarily focused on Debian / Ubuntu based systems, but the same concept applies to any system connected to a network.

Whether you're using a VM, installing on dedicated hardware, or running a Live OS temporarily, start *without* a network connection and disable any unnecessary services listening on all interfaces before connecting to the network.

The reasoning for this is because services like cups or avahi can be listening by default. While this isn't an immediate problem it simply broadens the attack surface. Not everyone will have a dedicated subnet or trusted network equipment they can control, and for the purposes of this guide, these steps treat *any* network as untrusted / hostile.

**Disable Listening Services**

- Ensures only essential network services are running
- If the service doesn't exist you'll get a "Failed to stop" which is fine
- Only disable `Bluetooth` if you don't need it

```bash
sudo systemctl stop bluetooth exim4 cups avahi avahi-daemon sshd
```

**Firewall**

Enable a basic firewall policy of *deny inbound, allow outbound*. Note that Debian does not come with a firewall, simply disabling the services in the previous step is fine. The following options have Ubuntu and similar systems in mind.

On Ubuntu, `ufw` is built in and easy to enable:

```bash
sudo ufw enable
```

On systems without `ufw`, `nftables` is replacing `iptables`. The [nftables wiki has examples](https://wiki.nftables.org/wiki-nftables/index.php/Simple_ruleset_for_a_workstation) for a baseline *deny inbound, allow outbound* policy. The `fw.inet.basic` policy covers both IPv4 and IPv6.

(Remember to download this README and any other resources to another external drive when creating the bootable media, to have this information ready to use offline)

Regardless of which policy you use, write the contents to a file (e.g. `nftables.conf`) and apply the policy with the following comand:

```bash
sudo nft -f ./nftables.conf
```

**Review the System State**

`NetworkManager` should be the only listening service on port 68/udp to obtain a DHCP lease (and 58/icmp6 if you have IPv6).

If you want to look at every process's command line arguments you can use `ps axjf`. This prints a process tree which may have a large number of lines but should be easy to read on a live image or fresh install.

```bash
sudo ss -anp -A inet    # Dump all network state information
ps axjf                 # List all processes in a process tree
ps aux                  # BSD syntax, list all processes but no process tree
```

If you find any additional processes listening on the network that aren't needed, take note and disable them with one of the following:

```bash
sudo systemctl stop <process-name>                      # Stops services managed by systemctl
sudo pkill -f '<process-name-or-command-line-string>'   # Terminate the process by matching it's command line string
pgrep -f '<process-name-or-command-line-string>'        # Obtain the PID
sudo kill <pid>                                         # Terminate the process via its PID
```

Now connect to a network.

# Notes

1. YubiKey has two configurations, invoked with either a short or long press. By default, the short-press mode is configured for HID OTP; a brief touch will emit an OTP string starting with `cccccccc`. OTP mode can be swapped to the second configuration via the YubiKey Personalization tool or disabled entirely using [YubiKey Manager](https://developers.yubico.com/yubikey-manager): `ykman config usb -d OTP`

1. Using YubiKey for GnuPG does not prevent use of [other features](https://developers.yubico.com/), such as [WebAuthn](https://developers.yubico.com/WebAuthn/) and [OTP](https://developers.yubico.com/OTP/).

1. Add additional identities to a Certify key with the `adduid` command during setup, then trust it ultimately with `trust` and `5` to configure for use.

1. To switch between YubiKeys, remove the first YubiKey and restart gpg-agent, ssh-agent and pinentry with `pkill "gpg-agent|ssh-agent|pinentry" ; eval $(gpg-agent --daemon --enable-ssh-support)` then insert the other YubiKey and run `gpg-connect-agent updatestartuptty /bye`

1. To use YubiKey on multiple computers, import the corresponding public keys, then confirm YubiKey is visible with `gpg --card-status`. Trust the imported public keys ultimately with `trust` and `5`, then `gpg --list-secret-keys` will show the correct and trusted key.

# Troubleshooting

- Use `man gpg` to understand GnuPG options and command-line flags.

- To get more information on potential errors, restart the `gpg-agent` process with debug output to the console with `pkill gpg-agent; gpg-agent --daemon --no-detach -v -v --debug-level advanced --homedir ~/.gnupg`.

- A lot of issues can be fixed by removing and re-inserting YubiKey, or restarting the `gpg-agent` process.

- If you receive the error, `Yubikey core error: no yubikey present` - make sure the YubiKey is inserted correctly. It should blink once when plugged in.

- If you still receive the error, `Yubikey core error: no yubikey present` - you likely need to install newer versions of yubikey-personalize as outlined in [Install software](#install-software).

- If you see `General key info..: [none]` in card status output - import the public key.

- If you receive the error, `gpg: decryption failed: secret key not available` - you likely need to install GnuPG version 2.x. Another possibility is that there is a problem with the PIN, e.g., it is too short or blocked.

- If you receive the error, `Yubikey core error: write error` - YubiKey is likely locked. Install and run yubikey-personalization-gui to unlock it.

- If you receive the error, `Key does not match the card's capability` - you likely need to use 2048-bit RSA key sizes.

- If you receive the error, `sign_and_send_pubkey: signing failed: agent refused operation` - make sure you replaced `ssh-agent` with `gpg-agent` as noted above.

- If you still receive the error, `sign_and_send_pubkey: signing failed: agent refused operation` - [run the command](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=835394) `gpg-connect-agent updatestartuptty /bye`

- If you still receive the error, `sign_and_send_pubkey: signing failed: agent refused operation` - edit `~/.gnupg/gpg-agent.conf` to set a valid `pinentry` program path. `gpg: decryption failed: No secret key` could also indicate an invalid `pinentry` path

- If you still receive the error, `sign_and_send_pubkey: signing failed: agent refused operation` - it is a [known issue](https://bbs.archlinux.org/viewtopic.php?id=274571) that openssh 8.9p1 and higher has issues with YubiKey. Adding `KexAlgorithms -sntrup761x25519-sha512@openssh.com` to `/etc/ssh/ssh_config` often resolves the issue.

- If you receive the error, `The agent has no identities` from `ssh-add -L`, make sure you have installed and started `scdaemon`

- If you receive the error, `Error connecting to agent: No such file or directory` from `ssh-add -L`, the UNIX file socket that the agent uses for communication with other processes may not be set up correctly. On Debian, try `export SSH_AUTH_SOCK="/run/user/$UID/gnupg/S.gpg-agent.ssh"`. Also see that `gpgconf --list-dirs agent-ssh-socket` is returning single path, to existing `S.gpg-agent.ssh` socket.

- If you receive the error, `Permission denied (publickey)`, increase ssh verbosity with the `-v` flag and verify the public key from the card is being offered: `Offering public key: RSA SHA256:abcdefg... cardno:00060123456`. If it is, verify the correct user the target system - not the user on the local system. Otherwise, be sure `IdentitiesOnly` is not [enabled](https://github.com/FiloSottile/whosthere#how-do-i-stop-it) for this host.

- If SSH authentication still fails - add up to 3 `-v` flags to the `ssh` command to increase verbosity.

- If it still fails, it may be useful to stop the background `sshd` daemon process service on the server (e.g. using `sudo systemctl stop sshd`) and instead start it in the foreground with extensive debugging output, using `/usr/sbin/sshd -eddd`. Note that the server will not fork and will only process one connection, therefore has to be re-started after every `ssh` test.

- If you receive the error, `Please insert the card with serial number` see [Using Multiple Keys](#using-multiple-keys).

- If you receive the error, `There is no assurance this key belongs to the named user` or `encryption failed: Unusable public key` or `No public key` use `gpg --edit-key` to set `trust` to `5 = I trust ultimately`

- If, when you try the above command, you get the error `Need the secret key to do this` - specify trust for the key in `~/.gnupg/gpg.conf` by using the `trust-key [key ID]` directive.

- If, when using a previously provisioned YubiKey on a new computer with `pass`, you see the following error on `pass insert`, you need to adjust the trust associated with the key. See the note above.

```
gpg: 0x0000000000000000: There is no assurance this key belongs to the named user
gpg: [stdin]: encryption failed: Unusable public key
```

- If you receive the error, `gpg: 0x0000000000000000: skipped: Unusable public key`, `signing failed: Unusable secret key`, or `encryption failed: Unusable public key` the Subkey may be expired and can no longer be used to encrypt nor sign messages. It can still be used to decrypt and authenticate, however.

- If the _pinentry_ graphical dialog does not show and this error appears: `sign_and_send_pubkey: signing failed: agent refused operation`, install the `dbus-user-session` package and restart for the `dbus` user session to be fully inherited. This is because `pinentry` complains about `No $DBUS_SESSION_BUS_ADDRESS found`, falls back to `curses` but doesn't find the expected `tty`

- If, when you try the above `--card-status` command, you get receive the error, `gpg: selecting card failed: No such device` or `gpg: OpenPGP card not available: No such device`, it's possible that the latest release of pcscd is now requires polkit rules to operate properly. Create the following file to allow users in the `wheel` group to use the card. Be sure to restart pcscd when you're done to allow the new rules to take effect.

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

- If the public key is lost, follow [this guide](https://www.nicksherlock.com/2021/08/recovering-lost-gpg-public-keys-from-your-yubikey/) to recover it from YubiKey.

- Refer to Yubico article [Troubleshooting Issues with GPG](https://support.yubico.com/hc/en-us/articles/360013714479-Troubleshooting-Issues-with-GPG) for additional guidance.

# Alternative solutions

* [`vorburger/ed25519-sk.md`](https://github.com/vorburger/vorburger.ch-Notes/blob/develop/security/ed25519-sk.md) - use YubiKey for SSH without GnuPG
* [`smlx/piv-agent`](https://github.com/smlx/piv-agent) - SSH and GnuPG agent which can be used with PIV devices
* [`keytotpm`](https://www.gnupg.org/documentation/manuals/gnupg/OpenPGP-Key-Management.html) - use GnuPG with TPM systems

# Additional resources

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
