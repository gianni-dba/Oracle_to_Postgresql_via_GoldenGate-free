# Oracle_to_Postgresql_via_GoldenGate-free
Replica completa e continua del database Oracle a Posgresql usando GoldenGate-free in docker

**DISCLAIMER:** Questo repository documenta un **laboratorio didattico e di studio personale**. Tutti i software Oracle (Database 23ai e GoldenGate Free) e PostgreSQL sono utilizzati in ambiente virtualizzato isolato esclusivamente per scopi di apprendimento, test e formazione personale, nel pieno rispetto delle rispettive licenze gratuite/OTN.

Questo progetto racconta come mettere in piedi una pipeline di **Change Data Capture (CDC)** eterogenea in tempo reale tra **Oracle Database 23ai** e **PostgreSQL 18**, passando attraverso **Oracle GoldenGate Free** eseguito su container **Docker**.

L'obiettivo non era solo far funzionare la replica, ma simulare un caso reale completo: partire da un database Oracle già popolato, effettuare il caricamento iniziale (**Initial Load**) senza perdere transazioni e agganciare la **sincronizzazione continua** in tempo reale.

---

## Architettura dell'Infrastruttura

Il laboratorio gira interamente su hypervisor **Proxmox VE**:

* **Nodo Sorgente (Oracle 23ai):** VM Rocky Linux 9 (`oracle-node5` - IP `192.168.1.156`) con istanza CDB `ORCL` e Pluggable Database `PDB1` (schema applicativo di logistica `LUIGI`, 6 tabelle relazionali + 1 tabella temporanea).
* **Nodo Target & Middleware (Debian + Docker + PostgreSQL 18):** VM Debian (`debian-xfce`) con doppio indirizzo IP configurato sulla stessa interfaccia di rete:
  * **PostgreSQL 18 (`192.168.1.153:5432`):** Database di destinazione `target_pdb1`, schema `luigi`.
  * **Container 1 — GoldenGate Free for Oracle (`192.168.1.153:9011-9015`):** Estrae i dati dal CDB/PDB Oracle e li spedisce via WebSocket (`ws`).
  * **Container 2 — GoldenGate Free for PostgreSQL (`192.168.1.151:9011-9015`):** Riceve i *Trail Files* e applica le modifiche su PostgreSQL 18.

```text
[Oracle 23ai - PDB1]  --->  [Container: ogg-oracle]  === (ws / Trail) ===>  [Container: ogg-postgres]  --->  [PostgreSQL 18]
  (192.168.1.156)             (192.168.1.153:9011)                            (192.168.1.151:9011)           (192.168.1.153:5432)
```

---

## Fasi principali del lavoro

### 1. Preparazione dei database (Sorgente e Destinazione)
* **Lato Oracle 23ai:** Abilitato `enable_goldengate_replication = TRUE` e il *Supplemental Logging* minimo a livello CDB. Creato l'utente comune `c##ggadmin` (`CONTAINER=ALL`) con privilegi di cattura e attivato il supplemental logging a livello di schema (`ADD SCHEMATRANDATA LUIGI`) affinché nei Redo Log vengano registrate tutte le informazioni sulle chiavi primarie.
* **Lato PostgreSQL 18:** Creato l'utente `ggadmin`, il database `target_pdb1` e lo schema `luigi` (impostando il `search_path` predefinito del ruolo su `luigi, public`). Tradotto il DDL delle 6 tabelle permanenti da Oracle a PostgreSQL (convertendo `VARCHAR2` in `VARCHAR`, `NUMBER` in `BIGINT`/`NUMERIC` e `DATE` Oracle in `TIMESTAMP(0)` per non perdere l'orario), escludendo la `Global Temporary Table` che non genera redo persistente.

### 2. Deploy dei due GoldenGate Free su Docker
Invece di creare due VM separate, entrambi i motori GoldenGate Microservices sono stati avviati sulla stessa VM Debian tramite `docker-compose.yml` utilizzando le immagini ufficiali `goldengate-oracle-free` e `goldengate-postgresql-free` (accettando la licenza, preventivamente letta sul sito, tramite variabile d'ambiente `ACCEPT_LICENSE=true`). I volumi dei deployment (`/u02`) sono stati marcati come `external: true` per proteggere i dati da rimozioni accidentali.

### 3. Initial Load (Migrazione dei dati già esistenti)
Avendo già record presenti nelle 6 tabelle su Oracle prima di avviare la replica, è stato eseguito un caricamento iniziale coerente:
1. Avviato prima l'**Extract continuo (`EXT_LUI`)** sul trail `ab` per iniziare a catturare eventuali modifiche in corso.
2. Rilevato l'esatto `CURRENT_SCN` sul `PDB1` e lanciato un **Initial Load Extract (`INIT_EXT`)** rispettando l'ordine delle Foreign Key (prima tabelle padre, poi figlie).
3. Trasferito il file trail `il` tra i due container tramite un **Distribution Path (`PATH_INIT`)** su protocollo `ws` (porta interna `9014`) e caricati i 22 record su PostgreSQL tramite un **Classic Replicat (`INIT_REP`)**.

### 4. Sincronizzazione Continua (Live CDC)
Completato l'Initial Load e rimossi i processi temporanei, è stata attivata la catena definitiva:
* **Extract (`EXT_LUI`)** → cattura in tempo reale ogni `INSERT`, `UPDATE` e `DELETE` sullo schema `LUIGI` nel trail `ab`.
* **Distribution Path (`PATH_LIVE`)** → invia i record dal container `ogg-oracle` al Receiver Server di `ogg-postgres` sulla rete interna Docker.
* **Classic Replicat (`REP_LUI`)** → legge dal trail `ab` (tracciando lo stato nella tabella `luigi.gg_chkpt`) e applica le transazioni su PostgreSQL in pochi secondi.

---

## Sfide tecniche risolte (e lezioni imparate)

Mettere insieme due motori diversi e due container GoldenGate sulla stessa macchina ha riservato diversi imprevisti interessanti:

1. **Conflitto delle porte interne `9011-9015` e soluzione con Doppio IP (`nmcli`):**
   GoldenGate Microservices usa diverse porte interne (`9011`-`9015`), e rimappare le porte del secondo container su `9021-9025` non basta: il frontend web del secondo container continua a fare chiamate REST verso le porte `9012-9015`, finendo per parlare con il primo container!
   * **Soluzione:** Aggiunto un secondo indirizzo IP statico (`192.168.1.151/24`) alla scheda di rete della VM Debian tramite `nmcli` (`+ipv4.addresses`), legando ciascun container alle proprie porte native `9011-9015` sul rispettivo IP dedicato.
2. **Permessi dei volumi Docker sui certificati SSL:**
   Montare un volume Docker vuoto direttamente su `/etc/nginx/cert` causava un crash all'avvio (`Permission denied` durante `genrsa`), poiché l'utente interno `oracle` (UID `54321`) non aveva permessi di scrittura sulla directory creata da `root`.
   * **Soluzione:** Rimosso il mount su `/etc/nginx/cert` lasciando persistente solo `/u02` (gestito correttamente dall'entrypoint dell'immagine).
3. **Bug UI su Trandata e uso di `adminclient`:**
   Nella Web UI di GoldenGate 23ai/26ai, la ricerca delle tabelle di luigi, cercate anche solo come tabelle, con o senza wildcards, nella sezione Trandata restituiva un errore di validazione JSON (`OGG-12111`).
   * **Soluzione:** Attivato il supplemental logging direttamente tramite CLI ufficiale (`adminclient` con `ADD SCHEMATRANDATA LUIGI`), verificando subito dopo la comparsa delle tabelle nella console web.
4. **Espansione disco senza LVM (`parted` + `resize2fs`):**
   Per avere più spazio da dedicare a questo progetto sulla VM Debian senza LVM, già occupata da Postgresql in PITR, è stato aumentato lo spazio disco, ma la partizione di swap in coda al disco impediva a `growpart` di espandere `/dev/sda1`. Disattivata temporaneamente la swap, ridimensionata la partizione principale con `parted`, espanso il filesystem `ext4` a caldo con `resize2fs` e ricreata la swap aggiornando `/etc/fstab`.
5. **Differenza tra Utente e Schema su PostgreSQL (`search_path`):**
   A differenza di Oracle dove Utente e Schema coincidono, su PostgreSQL l'utente `ggadmin` cercava di default lo schema `public`. Impostando `ALTER ROLE ggadmin SET search_path TO luigi, public;` le tabelle replicate sono diventate visibili immediatamente in ogni sessione `psql`.

