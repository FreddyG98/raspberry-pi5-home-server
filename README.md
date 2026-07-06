# 🎛️ Raspberry Pi 5 Home Server: Infrastruttura DevOps & Self-Hosting

Progettazione e implementazione di un'architettura di rete e self-hosting end-to-end su server dedicato, configurata per centralizzare la gestione dei media privati, ottimizzare la sicurezza di rete e ospitare istanze database relazionali per attività di Data Analysis.

L'intero ecosistema è orchestrato in container e gestito in modo professionale per garantire isolamento delle risorse, scalabilità e riavvii automatici in caso di interruzione di corrente.

---

## 🗺️ Schema dell'Infrastruttura
![Architettura di Rete e Container](./assets/schema-infrastruttura.png.png)

---

## 🚀 Obiettivi del Progetto
* **Indipendenza dai servizi Cloud commerciali:** Creazione di un'alternativa locale e privata a Google Foto per il backup automatico dei dispositivi mobili senza limiti di spazio o abbonamenti.
* **Ambiente DevOps/Sistemistico Sandbox:** Esercitazione pratica con l'orchestrazione di container Docker e gestione avanzata di server Linux headless.
* **Data Staging Hub:** Configurazione di un'istanza solida PostgreSQL per ospitare database aziendali a supporto di progetti di Business Intelligence e analisi dati.

---

## 🧰 Stack Tecnologico & Hardware
* **Hardware:** Kit Raspberry Pi 5 (8 GB RAM) + SSD Crucial NVMe da 500 GB (Alimentazione via cavo dedicata a 5A).
* **Sistema Operativo:** Raspberry Pi OS Lite (64-bit, Headless via SSH).
* **Orchestrazione:** Docker & Docker Compose.
* **Management GUI:** Portainer CE.
* **Network & Security:** Tailscale VPN (Peer-to-Peer).
* **Database Engine:** PostgreSQL + pgAdmin 4.
* **Media Cloud:** Immich (PostgreSQL + Typesense + Microservices) + Immich-GO.

---

## 🛠️ Step Implementativi & Soluzioni a Problemi Tecnici

### 1. Ottimizzazione Hardware & Configurazione SSD 500GB
Durante la fase di inizializzazione, l'alta velocità dell'SSD NVMe Crucial entrava in conflitto con l'attivazione automatica del risparmio energetico del Raspberry Pi 5, causando errori critici di lettura e fallimenti nella formattazione in Ext4.
* **Soluzione:** È stata forzata la velocità del bus a **PCIE Gen 2** modificando i parametri di boot del firmware del Raspberry e disattivando esplicitamente il risparmio energetico all'avvio.
* **Stabilità del File System:** Per evitare loop di mancato avvio dell'intero server in caso di problemi hardware all'SSD, il mount point è stato configurato nel file `/etc/fstab` recuperando l'UUID univoco del disco e applicando il flag `nofail`. Il Sistema Operativo principale risiede in clonazione sicura su SSD per preservare l'integrità dei dati rispetto alle fragili micro-SD.

### 2. Orchestrazione con Portainer
Al posto di interfacce preconfigurate semplificate (es. CasaOS), si è scelto l'approccio Enterprise tramite **Portainer CE** per gestire gli Stack di deployment. Questo ha permesso di mantenere un controllo granulare sulle reti isolate dei container, sui volumi mappati direttamente sull'archiviazione persistente dell'SSD e sul ciclo di vita delle applicazioni.

### 3. Migrazione Massiva Google Foto -> Immich
Scelto come Cloud multimediale per le performance strutturali (basato su database PostgreSQL nativo).
* **Data Ingestion:** Il caricamento dei dati storici è stato eseguito estraendo l'archivio *Google Takeout* (file .zip) ed eseguendo l'ingestion tramite utility a riga di comando **Immich-GO** via sessione SSH protetta.
* **Pipeline AI:** Al termine del caricamento, il server ha elaborato i metadati eseguendo task in background di Machine Learning dedicati al riconoscimento facciale e alla tecnologia OCR (estrazione testo dalle immagini), analizzando i file di log per validare l'importazione escludendo file corrotti o duplicati.

### 4. Rete Privata e Accesso Remoto Sicuro (Tailscale VPN)
Per accedere a Immich e ai database fuori casa senza esporre le porte del modem su internet (evitando attacchi informatici e port forwarding insicuri), è stata implementata una rete VPN Mesh tramite **Tailscale** installata nativamente sul sistema operativo host.
* **Ottimizzazione Batteria Mobile:** Il Raspberry è stato configurato come **Subnet Router**. Questa configurazione avanzata permette ai dispositivi autorizzati di raggiungere i container della rete locale senza costringere il client mobile a mantenere sessioni di routing pesanti, abbattendo il consumo di batteria dello smartphone all'1-3% giornaliero.

### 5. Configurazione e Tuning del Database PostgreSQL
L'istanza PostgreSQL è stata isolata all'interno di uno stack dedicato su Portainer per fungere da hub analitico centrale.
* **Ottimizzazione della Memoria:** È stata allocata una shared memory size (`shm_size: 256mb`) per prevenire crash del motore database a fronte di query massive o caricamenti di dataset complessi.
* **Fault Tolerance (Resistenza ai Blackout):** In assenza di un gruppo di continuità (UPS) fisico a causa di vincoli di spazio e requisiti di amperaggio del Pi 5 (5A), la sicurezza dell'infrastruttura è stata delegata a logiche software. Tutti i container critici (Immich e PostgreSQL) includono la policy `restart: always`. In caso di spegnimento improvviso della rete elettrica domestica, al ritorno della corrente il server si riavvia in autonomia ripristinando lo stato dei database e dei servizi senza alcun intervento manuale.
* **Gestione delle Dipendenze:** All'interno del file YAML è stata inserita la clausola `depends_on: -db` per garantire che l'interfaccia grafica di amministrazione (pgAdmin 4) si avvii solo dopo la completa inizializzazione e attivazione del motore database sottostante.

---

## 📄 Configurazione Docker Compose (Stack YAML)

Di seguito vengono riportati gli script di configurazione utilizzati all'interno di Portainer per il deployment dei servizi principali.

### Stack: PostgreSQL & pgAdmin 4
```yaml
version: '3.8'

services:
  # Il Motore del Database
  db:
    image: postgres:latest
    container_name: postgres_db
    restart: always
    shm_size: 256mb
    environment:
      POSTGRES_USER: <IL_TUO_UTENTE>
      POSTGRES_PASSWORD: <LA_TUA_PASSWORD>
      POSTGRES_DB: Progetto_1
    ports:
      - "5432:5432"
    volumes:
      - /mnt/nvme/postgres_data:/var/lib/postgresql

  # L'Interfaccia Web (pgAdmin)
  pgadmin:
    image: dpage/pgadmin4
    container_name: pgadmin_web
    restart: always
    environment:
      PGADMIN_DEFAULT_EMAIL: <LA_TUA_EMAIL>
      PGADMIN_DEFAULT_PASSWORD: <LA_TUA_PASSWORD>
    ports:
      - "8080:80"
    depends_on:
      - db
```

### Stack: Immich Media Cloud
```yaml
name: immich

services:
  immich-server:
    container_name: immich_server
    image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}
    volumes:
      - ${UPLOAD_LOCATION}:/usr/src/app/upload
      - /etc/localtime:/etc/localtime:ro
    env_file:
      - stack.env
    ports:
      - '2283:2283'
    depends_on:
      - redis
      - database
    restart: always
    healthcheck:
      disable: false

  immich-machine-learning:
    container_name: immich_machine_learning
    image: ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION:-release}
    volumes:
      - model-cache:/cache
    env_file:
      - stack.env
    restart: always
    healthcheck:
      disable: false

  redis:
    container_name: immich_redis
    image: docker.io/valkey/valkey:9@sha256:3b55fbaa0cd93cf0d9d961f405e4dfcc70efe325e2d84da207a0a8e6d8fde4f9
    healthcheck:
      test: redis-cli ping || exit 1
    restart: always

  database:
    container_name: immich_postgres
    image: ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0@sha256:bcf63357191b76a916ae5eb93464d65c07511da41e3bf7a8416db519b40b1c23
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_DB: ${DB_DATABASE_NAME}
      POSTGRES_INITDB_ARGS: '--data-checksums'
    volumes:
      - ${DB_DATA_LOCATION}:/var/lib/postgresql/data
    shm_size: 128mb
    restart: always
    healthcheck:
      disable: false

volumes:
  model-cache:
```
```
