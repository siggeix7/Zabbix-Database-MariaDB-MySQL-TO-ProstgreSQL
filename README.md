# Guida dettagliata: migrazione del database Zabbix da MySQL/MariaDB a PostgreSQL per Zabbix 7.0 e 7.4

> Adattamento della guida storica **“Migrating from MySQL to PostgreSQL”** (https://assets.zabbix.com/files/events/meetup_20200702/meetup_20200702_MySQL2PgSQL-ENG.pdf) presentata allo Zabbix Meetup del 2020, aggiornata per gli scenari **Zabbix 7.0 LTS** e **Zabbix 7.4 current**.
>
> Questa guida è pensata per una **migrazione del backend database** da **MySQL/MariaDB** a **PostgreSQL**, con opzione **TimescaleDB**. La logica resta quella della guida originale: creare la struttura PostgreSQL compatibile con la versione target di Zabbix, copiare i dati con `pgloader`, quindi avviare Zabbix contro PostgreSQL.

---

## 1. Scopo e perimetro

Questa procedura serve quando hai già una installazione Zabbix funzionante su **MySQL/MariaDB** e vuoi spostare il database applicativo su **PostgreSQL**.

Sono coperti due target:

- **Zabbix 7.0**
- **Zabbix 7.4**

La guida è volutamente più conservativa della presentazione del 2020:

- evita assunzioni specifiche su **CentOS 7**;
- evita di dipendere da layout storici dei pacchetti (`*-scl`, vecchi path, vecchi script);
- separa nettamente la migrazione del **motore database** dalla migrazione di **versione applicativa**;
- introduce i punti importanti comparsi dalla serie 7.0 in poi, soprattutto lato **TimescaleDB**.

---

## 2. Differenze principali rispetto alla guida del 2020

La guida originale era basata su:

- Zabbix **5.0**;
- **CentOS 7**;
- **MariaDB 5.5**;
- PostgreSQL 12;
- pacchetti frontend di vecchio tipo, per esempio `zabbix-web-pgsql-scl` e `zabbix-apache-conf-scl`.

Nelle versioni moderne cambiano soprattutto questi aspetti:

1. **Zabbix 7.x usa script e layout aggiornati**.
   In particolare, per **TimescaleDB**, dalla serie 7.0 lo script da eseguire è `postgresql/timescaledb/schema.sql`, e la documentazione segnala esplicitamente che posizione e nome dello script sono cambiati rispetto al passato.

2. **I pacchetti frontend dipendono da distribuzione e web server**.
   Non conviene riciclare alla cieca i vecchi pacchetti `*-scl`: su sistemi moderni i nomi possono cambiare e vanno presi dalla **pagina ufficiale di download** della combinazione OS/web server/versione.

3. **Zabbix 7.4 ha un perimetro di supporto più ampio**.
   Nella documentazione corrente, Zabbix 7.4 supporta PostgreSQL **13–18** e TimescaleDB **2.13.0–2.25.x**.

4. **TimescaleDB non è supportato per Zabbix proxy**.
   È supportato per il database del **server**, non per il **proxy**.

5. **Il proprietario del DB conta**.
   La documentazione raccomanda che l'utente `zabbix` sia **owner** del database PostgreSQL, perché deve poter modificare la struttura durante gli upgrade.

---

## 3. Strategia consigliata

### 3.1 Strategia consigliata in assoluto

La scelta più prudente è questa:

1. **Congelare la versione applicativa target**.
2. Portare l'istanza Zabbix nello stato applicativo desiderato **prima o dopo**, ma **non** mischiare nello stesso change sia:
   - cambio motore DB;
   - upgrade di major/minor Zabbix;
   - abilitazione TimescaleDB.

Operativamente, la strada più semplice da gestire è:

- **Scenario A**: l'istanza è già su **7.0** e vuoi solo passare a PostgreSQL → migra mantenendo **7.0**.
- **Scenario B**: l'istanza è già su **7.4** e vuoi solo passare a PostgreSQL → migra mantenendo **7.4**.
- **Scenario C**: sei su una versione precedente e vuoi arrivare a **7.4 PostgreSQL** → molto meglio fare:
  1. upgrade applicativo fino alla versione intermedia/target su MySQL,
  2. verifica completa,
  3. migrazione del motore DB,
  4. eventuale abilitazione di TimescaleDB.

### 3.2 Perché evitare il “doppio salto” nella stessa finestra

Se fai nello stesso momento:

- upgrade Zabbix;
- cambio da MySQL a PostgreSQL;
- attivazione TimescaleDB;

in caso di problema sarà molto più difficile capire se l'errore dipende da:

- schema applicativo;
- differenze SQL tra MySQL e PostgreSQL;
- pacchetti frontend/server;
- compressione/partizionamento TimescaleDB.

---

## 4. Prerequisiti

## 4.1 Requisiti applicativi

Prima di iniziare, assicurati che:

- lo **Zabbix server** attuale sia sano;
- il frontend sia accessibile;
- non ci siano upgrade DB pendenti;
- il log del server non mostri errori strutturali sul DB;
- siano note le credenziali del DB MySQL/MariaDB corrente.

## 4.2 Requisiti PostgreSQL

Per **Zabbix 7.4**, la documentazione corrente supporta:

- **PostgreSQL 13–18**;
- **TimescaleDB 2.13.0–2.25.x**.

Per **Zabbix 7.0**, lato TimescaleDB va considerato almeno il requisito minimo introdotto dalla serie **7.0.1**, cioè **TimescaleDB 2.13.0**.

### 4.3 Encoding

Usa **UTF-8**. Zabbix supporta come encoding database **solo UTF-8**.

### 4.4 Ownership del database

L'utente PostgreSQL usato da Zabbix deve essere **owner** del database. Questo serve anche per futuri upgrade schema.

### 4.5 Downtime

Questa non è una migrazione “live”. Devi prevedere una finestra con:

- **stop di Zabbix server**;
- eventuale stop frontend/web service se vuoi una finestra completamente coerente;
- freeze delle scritture lato applicazione durante il passaggio finale.

### 4.6 Backup obbligatori

Prima di qualunque attività:

1. backup logico del DB MySQL/MariaDB;
2. snapshot/backup VM o volume;
3. copia di:
   - `zabbix_server.conf`;
   - configurazione frontend;
   - eventuali unit file custom;
   - script esterni e alertscripts;
   - SELinux/firewall customizzazioni.

---

## 5. Strumenti da usare

Ti servono almeno:

- **PostgreSQL**
- **pgloader**
- i **pacchetti o sorgenti Zabbix della versione target**
- opzionalmente **TimescaleDB**

### 5.1 Nota pratica sui pacchetti Zabbix

Per installazioni da pacchetti, sui sistemi moderni conviene installare anche il pacchetto che contiene gli **SQL scripts** di Zabbix, perché torna utile sia per creare lo schema sia per eventuali post-step:

- tipicamente: `zabbix-sql-scripts`

Il path usato nella documentazione corrente per gli script PostgreSQL è del tipo:

```bash
/usr/share/zabbix/sql-scripts/postgresql/
```

Per TimescaleDB, in particolare:

```bash
/usr/share/zabbix/sql-scripts/postgresql/timescaledb/schema.sql
```

---

## 6. Scelta architetturale: PostgreSQL “plain” o PostgreSQL + TimescaleDB

## 6.1 Quando scegliere PostgreSQL senza TimescaleDB

Sceglilo se:

- l'ambiente è piccolo o medio;
- vuoi ridurre complessità iniziale;
- vuoi separare la migrazione del motore DB dall'introduzione di partizionamento/compressione.

## 6.2 Quando scegliere TimescaleDB

Sceglilo se:

- hai molto storico/trend;
- vuoi housekeeping più efficiente;
- hai dataset grandi o crescita rapida;
- vuoi usare compressione e hypertables.

## 6.3 Limite importante

**TimescaleDB non è supportato per Zabbix proxy**. Vale per il database del **server**.

---

## 7. Piano operativo ad alto livello

La sequenza consigliata è questa:

1. prepari PostgreSQL;
2. prepari un database PostgreSQL **vuoto** e l'utente `zabbix` owner;
3. generi uno **schema PostgreSQL pulito** esattamente compatibile con la tua versione target Zabbix;
4. separi lo schema in:
   - **pre-data** (tabelle, sequenze, funzioni, tipi);
   - **post-data** (indici, foreign key, constraint secondarie);
5. fermi Zabbix server;
6. lanci `pgloader` in modalità **data only**;
7. applichi la parte **post-data**;
8. opzionalmente abiliti **TimescaleDB** e lanci lo script dedicato;
9. installi/abiliti i pacchetti Zabbix per PostgreSQL;
10. riconfiguri `zabbix_server.conf`;
11. verifichi frontend e log;
12. solo a validazione conclusa spegni definitivamente MySQL.

Questa variante è più robusta dell'estrazione “artigianale” `create.sql`/`alter.sql` dalla vecchia `schema.sql`, perché usa direttamente gli **script ufficiali della versione target** per costruire uno schema corretto e poi lo re-esporta nelle sezioni utili alla migrazione.

---

## 8. Preparazione del server PostgreSQL

## 8.1 Installazione PostgreSQL

Installa una versione **supportata dalla release target**.

Esempio generico:

```bash
# RHEL-like / Rocky / Alma / Oracle Linux
sudo dnf install postgresql15-server postgresql15

# Debian / Ubuntu
sudo apt install postgresql-15 postgresql-client-15
```

> Sostituisci `15` con la major PostgreSQL che hai deciso di usare e che è supportata dalla tua release Zabbix target.

## 8.2 Inizializzazione e avvio

Esempio generico RHEL-like:

```bash
sudo /usr/pgsql-15/bin/postgresql-15-setup initdb
sudo systemctl enable --now postgresql-15
```

Esempio generico Debian/Ubuntu:

```bash
sudo systemctl enable --now postgresql
```

## 8.3 Parametri PostgreSQL da considerare

La documentazione Zabbix segnala che, a seconda della dimensione dell'installazione, può essere necessario aumentare almeno `work_mem`.

Per una migrazione/import consistente è inoltre sensato valutare:

- `shared_buffers`
- `work_mem`
- `maintenance_work_mem`
- `wal_buffers`
- `max_wal_size`
- `checkpoint_completion_target`

Per ambienti con TimescaleDB, dopo l'installazione può essere utile anche `timescaledb-tune`.

> Non mettere valori casuali in produzione: adeguali a RAM, IOPS e volume reale dei dati.

---

## 9. Creazione di utente e database PostgreSQL

## 9.1 Creazione utente

```bash
sudo -u postgres createuser --pwprompt zabbix
```

## 9.2 Creazione database

```bash
sudo -u postgres createdb -O zabbix -E UTF8 zabbix
```

## 9.3 Verifica owner

```bash
sudo -u postgres psql -d zabbix -c "\l+ zabbix"
```

Devi verificare che il proprietario sia `zabbix`.

---

## 10. Preparare gli script SQL corretti per la versione target

Questa è la parte chiave dell'adattamento moderno.

Invece di tagliare a mano una vecchia `schema.sql`, conviene:

1. creare un database **temporaneo** con lo schema ufficiale della versione target;
2. esportare da lì due file:
   - `create.sql` = **pre-data**
   - `post.sql` = **post-data**

Così ti assicuri che la struttura corrisponda esattamente a **Zabbix 7.0** o **Zabbix 7.4**.

## 10.1 Installare gli script SQL ufficiali

Su installazioni da pacchetti, installa i pacchetti della versione target, incluso lo script package.

Esempio generico:

```bash
sudo dnf install zabbix-server-pgsql zabbix-sql-scripts
```

oppure:

```bash
sudo apt install zabbix-server-pgsql zabbix-sql-scripts
```

> Il nome esatto dei pacchetti frontend dipende dalla combinazione OS/web server/versione. Per la parte server SQL, ciò che conta qui è avere gli **script PostgreSQL ufficiali** della tua release target.

## 10.2 Creare un database temporaneo di staging

```bash
sudo -u postgres createdb -O zabbix -E UTF8 zabbix_stage
```

## 10.3 Caricare lo schema ufficiale completo nello staging

Per installazioni da pacchetti, in genere il file completo server è disponibile come:

```bash
/usr/share/zabbix/sql-scripts/postgresql/server.sql.gz
```

Caricalo nel database temporaneo:

```bash
zcat /usr/share/zabbix/sql-scripts/postgresql/server.sql.gz | sudo -u zabbix psql zabbix_stage
```

## 10.4 Esportare schema diviso in pre-data e post-data

Ora genera i due file che userai con `pgloader`:

```bash
mkdir -p ~/zbx-db-migration
cd ~/zbx-db-migration

sudo -u postgres pg_dump \
  --section=pre-data \
  --no-owner \
  --no-privileges \
  zabbix_stage > create.sql

sudo -u postgres pg_dump \
  --section=post-data \
  --no-owner \
  --no-privileges \
  zabbix_stage > post.sql
```

## 10.5 Ripulire il database temporaneo

```bash
sudo -u postgres dropdb zabbix_stage
```

## 10.6 Perché questo approccio è preferibile

Perché:

- usa **lo schema ufficiale** della release target;
- evita di indovinare come siano strutturati i file SQL tra 5.0, 7.0 e 7.4;
- separa automaticamente la parte di **definizione oggetti** dalla parte di **indici/vincoli**;
- riduce il rischio di mismatch tra schema applicativo e dati migrati.

---

## 11. Preparare pgloader

## 11.1 Installazione

Esempio generico:

```bash
# RHEL-like
sudo dnf install pgloader

# Debian/Ubuntu
sudo apt install pgloader
```

## 11.2 Verifica versione

```bash
pgloader --version
```

---

## 12. File di migrazione pgloader

Crea un file, per esempio `zabbix_migrate.load`:

```lisp
LOAD DATABASE
     FROM mysql://zabbix:MYSQL_PASSWORD@MYSQL_HOST/zabbix
     INTO postgresql://zabbix:PG_PASSWORD@PG_HOST/zabbix

 WITH data only,
      truncate,
      reset sequences

 BEFORE LOAD EXECUTE 'create.sql'
 AFTER LOAD EXECUTE 'post.sql';
```

## 12.1 Spiegazione direttive

- `data only`:
  dice a pgloader di copiare i dati ma non di costruire in autonomia le tabelle.
- `truncate`:
  svuota le tabelle target se presenti.
- `reset sequences`:
  riallinea le sequence PostgreSQL ai valori importati.
- `BEFORE LOAD EXECUTE 'create.sql'`:
  crea la struttura base.
- `AFTER LOAD EXECUTE 'post.sql'`:
  applica indici, FK e oggetti post-data dopo il caricamento.

## 12.2 Nota importante su password e caratteri speciali

Se la password contiene caratteri speciali (`@`, `:`, `/`, `#`, `%`), valuta di:

- usare URL encoding, oppure
- usare credenziali temporanee semplici per la finestra di migrazione.

---

## 13. Checkpoint prima del cutover

Prima di fermare Zabbix, fai questi controlli:

### 13.1 Controllo lato MySQL/MariaDB

- verifica dimensione DB;
- verifica numero righe grandi tabelle (`history*`, `trends*`, `events`, `auditlog`, `problem`, `alerts`);
- controlla eventuali tabelle orfane o custom.

### 13.2 Controllo lato PostgreSQL

- DB `zabbix` vuoto e accessibile;
- utente `zabbix` owner;
- spazio disco sufficiente;
- WAL, filesystem e backup pronti.

### 13.3 Controllo lato applicazione

- hai già installato o hai pronti i pacchetti **Zabbix server pgsql** della release target;
- sai già quale package frontend userai dopo la migrazione;
- hai già copiato le configurazioni correnti.

---

## 14. Finestra di migrazione

## 14.1 Fermare Zabbix server

```bash
sudo systemctl stop zabbix-server
```

Se vuoi massima coerenza anche lato UI e servizi collegati:

```bash
sudo systemctl stop httpd
# oppure
sudo systemctl stop nginx php-fpm
```

Se usi `zabbix-web-service`:

```bash
sudo systemctl stop zabbix-web-service
```

## 14.2 Eseguire pgloader

Dalla directory dove hai `create.sql`, `post.sql` e `zabbix_migrate.load`:

```bash
cd ~/zbx-db-migration
pgloader zabbix_migrate.load
```

## 14.3 Cosa aspettarsi

Su database grandi, il grosso del tempo sarà quasi sempre su:

- `history`
- `history_uint`
- `history_str`
- `history_text`
- `history_log`
- `trends`
- `trends_uint`
- `auditlog`
- `events`

È normale che il caricamento richieda molto tempo.

---

## 15. Validazioni immediate post-import

## 15.1 Verifica tabelle principali

```bash
sudo -u zabbix psql zabbix -c "\dt"
```

## 15.2 Verifica conteggi orientativi

Confronta alcuni conteggi tra MySQL e PostgreSQL. Esempi:

```sql
SELECT COUNT(*) FROM hosts;
SELECT COUNT(*) FROM items;
SELECT COUNT(*) FROM triggers;
SELECT COUNT(*) FROM events;
SELECT COUNT(*) FROM problem;
SELECT COUNT(*) FROM history_uint;
SELECT COUNT(*) FROM trends_uint;
SELECT COUNT(*) FROM auditlog;
```

> Sulle tabelle molto grandi può bastare un confronto per campione o per range temporale, se il downtime è stretto.

## 15.3 Verifica sequence

```bash
sudo -u zabbix psql zabbix -c "SELECT MAX(eventid) FROM events;"
sudo -u zabbix psql zabbix -c "SELECT last_value FROM events_eventid_seq;"
```

La sequence deve essere coerente col massimo ID importato. `reset sequences` di pgloader di norma lo gestisce, ma va comunque verificato.

## 15.4 Verifica ownership e schema

```bash
sudo -u postgres psql zabbix -c "\dn+"
sudo -u postgres psql zabbix -c "\dt+ public.*"
```

L'obiettivo è avere tabelle usabili dal server Zabbix senza residui strani di ownership.

---

## 16. Abilitazione opzionale di TimescaleDB

## 16.1 Quando farla

Puoi farla:

- subito dopo la migrazione a PostgreSQL;
- oppure in una finestra separata.

La seconda opzione è più prudente se vuoi isolare meglio i rischi.

## 16.2 Installare l'estensione

Installa una versione supportata di TimescaleDB sul server PostgreSQL.

## 16.3 Abilitare l'estensione nel DB Zabbix

```bash
echo "CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;" | sudo -u postgres psql zabbix
```

Se usi uno schema diverso da `public`, la documentazione richiede la clausola `SCHEMA`.

## 16.4 Eseguire lo script TimescaleDB di Zabbix

Con Zabbix 7.x il file corretto è:

```bash
cat /usr/share/zabbix/sql-scripts/postgresql/timescaledb/schema.sql | sudo -u zabbix psql zabbix
```

## 16.5 Effetti dello script

Lo script configura gli hypertable e imposta parametri di housekeeping/compressione. La documentazione avverte che la migrazione di storico, trend e audit log può richiedere molto tempo e che **server e frontend devono restare fermi durante la migrazione**.

## 16.6 Nota 7.0 importante

Dalla serie **7.0** lo `auditlog` è stato convertito in hypertable nelle nuove installazioni con TimescaleDB. Inoltre, per upgrade verso 7.x con TimescaleDB, la documentazione richiede l'esecuzione di `postgresql/timescaledb/schema.sql` dopo l'upgrade del DB.

---

## 17. Installazione/attivazione dei componenti Zabbix per PostgreSQL

Qui dipende fortemente dal sistema operativo e dal web server.

## 17.1 Principio generale

Devi sostituire i componenti **server/frontend MySQL** con quelli **PostgreSQL** della **stessa release target**.

### Componenti tipici

- `zabbix-server-pgsql`
- pacchetto frontend PostgreSQL compatibile con Apache o Nginx
- eventuale `zabbix-apache-conf` oppure configurazione Nginx/PHP-FPM equivalente
- `zabbix-sql-scripts` se non ancora installato

## 17.2 Caso storico vs caso moderno

Nella guida del 2020 comparivano pacchetti tipo:

```bash
zabbix-web-pgsql-scl
zabbix-apache-conf-scl
```

Questi riferimenti erano coerenti con l'ecosistema **CentOS 7 / SCL** del tempo. Non vanno presi come riferimento universale per 7.0/7.4.

### Regola pratica corretta

Per Zabbix **7.0** e **7.4**, recupera i comandi esatti dalla **pagina ufficiale di download Zabbix** selezionando:

- versione Zabbix;
- distribuzione;
- release OS;
- database = PostgreSQL;
- web server = Apache o Nginx.

---

## 18. Configurazione di `zabbix_server.conf`

Apri il file:

```bash
sudo vi /etc/zabbix/zabbix_server.conf
```

Imposta almeno:

```ini
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=LA_TUA_PASSWORD
```

### Note utili

- se PostgreSQL gira su socket Unix e vuoi usarlo, puoi adeguare `DBHost`;
- se usi TLS per PostgreSQL, configura lato PostgreSQL e lato libreria client secondo documentazione;
- verifica anche eventuali parametri custom già presenti nel file originale.

---

## 19. Frontend Zabbix

## 19.1 Configurazione frontend precedente

Se il frontend aveva una configurazione DB statica precedente, può essere necessario rimuoverla o rigenerarla.

Storicamente:

```bash
sudo rm -f /etc/zabbix/web/zabbix.conf.php
```

Su alcuni layout più recenti il file può trovarsi altrove, ma il principio resta: il frontend deve essere riallineato alla nuova connessione PostgreSQL.

## 19.2 Wizard frontend

Alla riapertura del frontend, il wizard ti chiederà i parametri DB PostgreSQL.

Campi tipici:

- Database type: `PostgreSQL`
- Database host: `localhost` o host reale
- Database port: `5432`
- Database name: `zabbix`
- User: `zabbix`
- Password: la password impostata

---

## 20. Riavvio dei servizi

Esempio con Apache:

```bash
sudo systemctl restart zabbix-server
sudo systemctl restart httpd
```

Esempio con Nginx/PHP-FPM:

```bash
sudo systemctl restart zabbix-server
sudo systemctl restart nginx
sudo systemctl restart php-fpm
```

Se usi `zabbix-web-service`:

```bash
sudo systemctl restart zabbix-web-service
```

---

## 21. Validazione funzionale post-cutover

Dopo il riavvio, verifica in questo ordine.

## 21.1 Log del server

Controlla immediatamente:

```bash
sudo journalctl -u zabbix-server -b
```

oppure:

```bash
sudo tail -f /var/log/zabbix/zabbix_server.log
```

Cerca in particolare:

- errori di autenticazione PostgreSQL;
- errori di schema;
- errori su sequence;
- errori su TimescaleDB/hypertable;
- warning relativi a compressione non supportata.

## 21.2 Login al frontend

Verifica:

- login riuscito;
- dashboard caricata;
- nessun errore PHP lato frontend;
- Administration / Housekeeping accessibile;
- Problem view caricata correttamente;
- Latest data funzionante.

## 21.3 Scrittura dati nuova generazione

Controlla che arrivino nuovi valori:

- item attivi aggiornati;
- nuovi eventi/problemi;
- trend/storico in crescita;
- queue non fuori controllo.

## 21.4 Report e componenti accessori

Se usi:

- report schedulati;
- media types custom;
- script esterni;
- API client;
- integrazioni esterne;

verifica esplicitamente anche quelle.

---

## 22. Cosa fare solo dopo validazione completa

Solo quando sei sicuro che tutto funzioni:

1. disabilita definitivamente MySQL/MariaDB;
2. archivia dump e snapshot di rollback;
3. aggiorna documentazione interna;
4. aggiorna monitoring/backup del nuovo PostgreSQL.

Esempio:

```bash
sudo systemctl stop mariadb
sudo systemctl disable mariadb
```

oppure:

```bash
sudo systemctl stop mysql
sudo systemctl disable mysql
```

---

## 23. Procedura sintetica per Zabbix 7.0

Questa sezione riassume il flusso se il target è **7.0**.

## 23.1 Raccomandazioni specifiche 7.0

- Non usare riferimenti a pacchetti frontend vecchi `*-scl` come verità assoluta.
- Se abiliti TimescaleDB, considera almeno **2.13.0**.
- Ricorda che dalla serie 7.0 il file corretto è:

```bash
/usr/share/zabbix/sql-scripts/postgresql/timescaledb/schema.sql
```

- Se stai partendo da una versione precedente a 7.0, è molto meglio non combinare nello stesso passaggio upgrade applicativo e cambio motore DB.

## 23.2 Sequenza pratica 7.0

```bash
# 1) Installa PostgreSQL supportato e pgloader
# 2) Crea utente/db zabbix su PostgreSQL
# 3) Installa gli SQL scripts Zabbix 7.0
# 4) Crea zabbix_stage e importa server.sql.gz
# 5) Esporta create.sql e post.sql con pg_dump --section
# 6) Ferma zabbix-server
# 7) Esegui pgloader
# 8) (Opzionale) abilita TimescaleDB e lancia schema.sql
# 9) Installa/attiva zabbix-server-pgsql e frontend pgsql compatibili
# 10) Aggiorna zabbix_server.conf
# 11) Riavvia e valida
```

---

## 24. Procedura sintetica per Zabbix 7.4

Questa sezione riassume il flusso se il target è **7.4**.

## 24.1 Raccomandazioni specifiche 7.4

- Usa una versione PostgreSQL nel range supportato **13–18**.
- Se usi TimescaleDB, stai nel range supportato documentato per 7.4.
- Considera `work_mem` e tuning PostgreSQL più seriamente se il dataset è grande.
- Se parti da una release più vecchia, cerca di arrivare prima a una situazione applicativa pulita e poi migrare il motore DB.

## 24.2 Sequenza pratica 7.4

```bash
# 1) Installa PostgreSQL supportato e pgloader
# 2) Crea utente/db zabbix su PostgreSQL
# 3) Installa zabbix-server-pgsql + zabbix-sql-scripts 7.4
# 4) Crea zabbix_stage e importa server.sql.gz 7.4
# 5) Genera create.sql e post.sql
# 6) Ferma zabbix-server
# 7) Esegui pgloader in data only
# 8) (Opzionale) abilita TimescaleDB e lancia lo schema dedicato
# 9) Allinea frontend e web service ai pacchetti 7.4 PostgreSQL
# 10) Aggiorna zabbix_server.conf
# 11) Riavvia e valida end-to-end
```

---

## 25. Troubleshooting

## 25.1 pgloader fallisce su tipi o conversioni

Azioni:

- controlla la versione di pgloader;
- ripeti il test in staging;
- riduci oggetti custom nel DB sorgente;
- verifica eventuali tabelle/plugin non standard.

## 25.2 Zabbix server parte ma segnala errore schema

Possibili cause:

- `create.sql`/`post.sql` generati da release diversa da quella dei binari Zabbix;
- pacchetto server 7.4 contro schema 7.0, o viceversa;
- mancata esecuzione dello script TimescaleDB dopo l'upgrade/migrazione.

## 25.3 Frontend non si connette a PostgreSQL

Controlla:

- credenziali;
- `pg_hba.conf`;
- firewall;
- host/porta;
- pacchetto frontend corretto;
- eventuale file `zabbix.conf.php` residuo.

## 25.4 Sequence fuori sync

Esempio di correzione manuale:

```sql
SELECT setval('events_eventid_seq', (SELECT MAX(eventid) FROM events));
```

Ripetere, se necessario, per altre sequence.

## 25.5 Performance scarse dopo la migrazione

Controlla:

- `work_mem`;
- `shared_buffers`;
- autovacuum;
- ANALYZE/VACUUM;
- eventuale opportunità di TimescaleDB;
- storage e WAL.

## 25.6 TimescaleDB: warning su compression/hypertables

Controlla:

- versione TimescaleDB supportata;
- esecuzione riuscita di `postgresql/timescaledb/schema.sql`;
- licenza/edition che consenta la compressione;
- parametri di housekeeping in frontend.

---

## 26. Runbook di rollback minimo

Se il cutover fallisce:

1. stoppa Zabbix server;
2. ripristina i pacchetti/config server MySQL originali;
3. ripristina il file `zabbix_server.conf` originale;
4. riporta online il DB MySQL/MariaDB;
5. riavvia Zabbix su MySQL;
6. conserva PostgreSQL solo per analisi post-mortem.

---

## 27. Checklist finale

## Pre-migrazione

- [ ] Backup MySQL/MariaDB eseguito
- [ ] Snapshot host/VM eseguito
- [ ] Versione target definita: 7.0 oppure 7.4
- [ ] PostgreSQL supportato installato
- [ ] Utente `zabbix` owner del DB PostgreSQL
- [ ] `pgloader` installato
- [ ] Pacchetti/script Zabbix target disponibili
- [ ] Creati `create.sql` e `post.sql`
- [ ] Finestra di downtime approvata

## Durante migrazione

- [ ] `zabbix-server` fermato
- [ ] `pgloader` completato senza errori bloccanti
- [ ] Conteggi principali validati
- [ ] Sequence validate
- [ ] TimescaleDB abilitato e schema applicato, se previsto
- [ ] `zabbix_server.conf` aggiornato

## Post-migrazione

- [ ] `zabbix-server` avviato correttamente
- [ ] Frontend raggiungibile
- [ ] Login funzionante
- [ ] Nuovi dati in arrivo
- [ ] Problemi/eventi visualizzati correttamente
- [ ] Housekeeping verificato
- [ ] MySQL spento solo dopo validazione completa

---

## 28. Riferimenti ufficiali e fonte originale

### Fonte originale fornita

- Zabbix Meetup 2020: **Migrating from MySQL to PostgreSQL**

### Documentazione Zabbix rilevante

- Zabbix documentation 7.0 / 7.4
- Requirements
- Database creation
- Installation from sources
- Web interface installation
- Securing PostgreSQL/TimescaleDB
- TimescaleDB setup
- Upgrade notes 7.0.1 / 7.0.2 / 7.0.x
- Download page per i pacchetti esatti della propria piattaforma

---

## 29. Conclusione

L'adattamento corretto della guida del 2020 per **Zabbix 7.0** e **Zabbix 7.4** non consiste tanto nel cambiare due nomi di pacchetto, quanto nel cambiare approccio:

- usare lo **schema ufficiale della release target**;
- generare in modo pulito **pre-data** e **post-data**;
- trattare **TimescaleDB** come passo esplicito e consapevole;
- riallineare pacchetti server/frontend alla versione e alla distro reali;
- validare bene prima di spegnere il vecchio MySQL.

Se questa procedura viene eseguita in staging e poi ripetuta in produzione con lo stesso runbook, il rischio operativo scende moltissimo rispetto a una migrazione “one shot” improvvisata.
