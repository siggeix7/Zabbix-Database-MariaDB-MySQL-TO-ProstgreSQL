**migrare il database di Zabbix 5.0 da MySQL a PostgreSQL**.

### 🛠️ Prerequisiti
*   **Sistema Operativo:** CentOS 7
*   **Versione Zabbix:** Zabbix 5.0
*   **Database di partenza:** MariaDB 5.5.65 (MySQL)
*   **Strumenti richiesti:** PostgreSQL, `pgloader` e il codice sorgente di Zabbix.

---

### Fase 1: Installazione di PostgreSQL e `pgloader`

**1. Aggiungere la repository di PostgreSQL e installarlo**
```bash
yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
yum install postgresql12-server
/usr/pgsql-12/bin/postgresql-12-setup initdb
systemctl enable postgresql-12
systemctl start postgresql-12
```

**2. Installare pgloader**
```bash
yum install pgloader
pgloader -V
# L'output atteso dovrebbe essere simile a: pgloader version "3.6.2"
```

---

### Fase 2: Download del sorgente di Zabbix

**1. Creare una directory di lavoro e spostarcisi**
```bash
mkdir myzabbix-pgzabbix
cd myzabbix-pgzabbix
```

**2. Scaricare e scompattare il codice sorgente di Zabbix**
```bash
yum install wget
wget https://cdn.zabbix.com/zabbix/sources/stable/5.0/zabbix-5.0.1.tar.gz
tar -zxvf zabbix-5.0.1.tar.gz
```

---

### Fase 3: Preparazione di PostgreSQL

**1. Separare lo schema di default (`schema.sql`) in due file (`create.sql` e `alter.sql`)**
```bash
cd myzabbix-pgzabbix/zabbix-5.0.1/database/postgresql/
sed -n '/CREATE.*/,/INSERT.*\$/p' schema.sql | head -n -1 > create.sql
grep ALTER schema.sql > alter.sql
```

**2. Creare un utente e il database in PostgreSQL**
```bash
sudo -u postgres createuser --pwprompt zabbix
sudo -u postgres createdb -O zabbix zabbix
```

---

### Fase 4: Configurazione dello script di migrazione

Nella stessa directory in cui hai generato `create.sql` e `alter.sql`, crea un file chiamato `zabbix_migrate.load`. 

Questo è il file di configurazione per `pgloader`. Inserisci il seguente contenuto, assicurandoti di sostituire `zabbix-password` con le tue reali password:

```text
LOAD DATABASE
FROM mysql://zabbix:zabbix-password@localhost/zabbix
INTO postgresql://zabbix:zabbix-password@localhost/zabbix
WITH include no drop,
truncate,
create no tables,
create no indexes,
no foreign keys,
reset sequences,
data only

SET maintenance_work_mem TO '1024MB', work_mem to '256MB'
ALTER SCHEMA 'zabbix' RENAME TO 'public'
BEFORE LOAD EXECUTE create.sql
AFTER LOAD EXECUTE alter.sql;
```

---

### Fase 5: Esecuzione della Migrazione

**1. Ferma il server Zabbix**
```bash
systemctl stop zabbix-server
```

**2. Esegui pgloader per migrare i dati**
```bash
pgloader zabbix-migrate.load
```
*Nota: Potresti visualizzare alcuni messaggi di WARNING (es. riguardo i campi `widget_field`). La guida specifica che è un comportamento normale.*

---

### Fase 6: Transizione a PostgreSQL e Passaggi Finali

**1. Rimuovi i vecchi pacchetti Zabbix per MySQL**
```bash
yum remove Zabbix-server-mysql
yum remove zabbix-web-*
```

**2. Installa i pacchetti Zabbix per PostgreSQL**
```bash
yum install zabbix-server-pgsql
yum install zabbix-web-pgsql-scl zabbix-apache-conf-scl
```

**3. Modifica la configurazione del nuovo server Zabbix**
```bash
vi /etc/zabbix/zabbix_server.conf
```
Assicurati di inserire i nuovi dati di accesso del DB (es. aggiungendo/modificando `# DBPassword=zabbix`).

**4. Elimina la vecchia configurazione web**
```bash
rm /etc/zabbix/web/zabbix.conf.php
```

**5. Sistema la timezone (se necessario)**
Decommenta la configurazione del fuso orario per il nuovo frontend:
```bash
vi /etc/httpd/conf.d/zabbix.conf
```

**6. Ferma il vecchio DB e avvia il nuovo ambiente**
```bash
systemctl stop mariadb
systemctl restart zabbix-server httpd
```

**7. Riconfigurare il frontend**
Vai sull'interfaccia web di Zabbix. Partirà nuovamente la schermata di setup (poiché hai cancellato il file PHP di configurazione):
* Seleziona `PostgreSQL` come *Database type*.
* Compila i campi `Database host` (localhost), `Database port` (0), `Database name`, `User` e `Password` con i dati PostgreSQL appena creati.
* Completa il wizard e goditi il tuo server Zabbix migrato!
