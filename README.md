# LODE (lode-ng)

LODE (Live OWL Documentation Environment) genera automaticamente documentazione HTML navigabile a partire da ontologie OWL.

Questo fork ([teamdigitale/dati-semantic-lode](https://github.com/teamdigitale/dati-semantic-lode)) modernizza il progetto originale [essepuntato/LODE](https://github.com/essepuntato/LODE) portandolo a:

- **Java 21** (Eclipse Temurin)
- **Spring Boot 3.x** con Tomcat embedded
- **Gradle** come build system
- Artifact JAR eseguibile con `java -jar` (niente Tomcat esterno)

> **Nota per chi usa la versione originale con Tomcat:** le versioni precedenti di LODE richiedevano il deploy di un file WAR su un servlet container (Tomcat, Jetty). Questo fork utilizza Spring Boot con Tomcat embedded: è sufficiente eseguire `java -jar lode.jar`. Il deploy su un application server esterno **non è supportato né consigliato**.

---

## Prerequisiti

- Server Ubuntu 22.04+ (per installazione nativa) oppure Docker

---

## Opzione 1: Docker (consigliata)

Le immagini ufficiali sono pubblicate su GitHub Container Registry. Questa è la modalità di installazione consigliata.

### 1.1 Installare Docker

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
```

### 1.2 Creare il file di environment

Creare il file `lode.env` e **personalizzare i valori** in base al proprio ambiente:

```env
# URL pubblico dove LODE è raggiungibile
EXTERNAL_URL=<URL_PUBBLICO_LODE>

# URL di WebVOWL per la visualizzazione grafica delle ontologie
# Se WebVOWL gira sullo stesso host, usare il suo URL pubblico
WEBVOWL_URL=<URL_WEBVOWL>/#iri=

# Origini consentite per CORS (default: * = tutte)
CORS_ALLOWED_ORIGINS=*
```

### 1.3 Avviare il container

```bash
docker run -d \
  --name lode \
  --restart unless-stopped \
  -p 8080:8080 \
  --env-file lode.env \
  ghcr.io/teamdigitale/dati-semantic-lode:latest
```

Verificare:

```bash
docker logs lode
# L'applicazione è disponibile su http://localhost:8080
```

---

## Opzione 2: Installazione nativa con systemd

Questa modalità prevede il build dai sorgenti e l'avvio come servizio di sistema.

### 2.1 Installare Java 21

```bash
sudo apt update
sudo apt install -y eclipse-temurin-21-jdk
```

Se il pacchetto non è disponibile, aggiungere il repository Adoptium:

```bash
sudo apt install -y wget apt-transport-https gpg
wget -qO - https://packages.adoptium.net/artifactory/api/gpg/key/public | sudo gpg --dearmor -o /usr/share/keyrings/adoptium.gpg
echo "deb [signed-by=/usr/share/keyrings/adoptium.gpg] https://packages.adoptium.net/artifactory/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/adoptium.list
sudo apt update
sudo apt install -y temurin-21-jdk
```

Verificare:

```bash
java -version
# openjdk version "21.x.x" ...
```

### 2.2 Scaricare i sorgenti e buildare

```bash
sudo useradd -r -s /usr/sbin/nologin lode

cd /opt
sudo git clone https://github.com/teamdigitale/dati-semantic-lode.git
cd dati-semantic-lode

# Build senza test
sudo ./gradlew clean build -x test

# Copiare l'artifact nella directory di installazione
sudo mkdir -p /opt/lode
sudo cp build/libs/lode.jar /opt/lode/lode.jar
sudo chown -R lode:lode /opt/lode
```

**(Opzionale)** Rimuovere i sorgenti dopo il build per liberare spazio:

```bash
sudo rm -rf /opt/dati-semantic-lode
```

### 2.3 Creare il file di environment

Creare il file `/opt/lode/lode.env` e **personalizzare i valori** (il formato è lo stesso usato per Docker):

```env
# URL pubblico dove LODE è raggiungibile
EXTERNAL_URL=<URL_PUBBLICO_LODE>

# URL di WebVOWL per la visualizzazione grafica delle ontologie
# Se WebVOWL gira sullo stesso host, usare il suo URL pubblico
WEBVOWL_URL=<URL_WEBVOWL>/#iri=

# Origini consentite per CORS (default: * = tutte)
CORS_ALLOWED_ORIGINS=*
```

### 2.4 Creare il file di servizio systemd

Creare il file `/etc/systemd/system/lode.service`:

```ini
[Unit]
Description=LODE - Live OWL Documentation Environment
After=network.target

[Service]
Type=simple
User=lode
Group=lode
WorkingDirectory=/opt/lode

EnvironmentFile=/opt/lode/lode.env
ExecStart=/usr/bin/java -jar /opt/lode/lode.jar

Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### 2.5 Avviare il servizio

```bash
sudo systemctl daemon-reload
sudo systemctl enable lode
sudo systemctl start lode

# Verificare lo stato
sudo systemctl status lode

# Il servizio è disponibile su http://localhost:8080
```

---

## Variabili d'ambiente

| Variabile | Descrizione | Default |
|---|---|---|
| `EXTERNAL_URL` | URL pubblico dove LODE è raggiungibile | vuoto |
| `WEBVOWL_URL` | URL di WebVOWL per la visualizzazione grafica (es. `https://webvowl.example.com/#iri=`) | `/webvowl/#iri=` |
| `CORS_ALLOWED_ORIGINS` | Origini consentite per CORS | `*` |

---

## Nota per chi usa Apache HTTPD come reverse proxy

Se si dispone già di un reverse proxy Apache HTTPD configurato per la versione precedente (Tomcat esterno), tenere presente che il modello architetturale è cambiato:

- **Prima:** Apache parlava con un unico processo Tomcat su una singola porta, smistando le richieste per path (es. `/lodview`, `/lode`, `/webvowl`).
- **Ora:** ogni visualizzatore è un processo Spring Boot autonomo in ascolto sulla propria porta locale.

Tutte le applicazioni partono di default sulla porta **8080**. Se si eseguono più visualizzatori sulla stessa macchina, è necessario assegnare porte diverse tramite la variabile d'ambiente `SERVER_PORT` (vedi la [sezione porte](#porte) e il [README di LodView](https://github.com/teamdigitale/dati-semantic-lodview) per la tabella completa).

Apache può continuare a fare reverse proxy, ma il backend non è più un unico Tomcat condiviso.

### Virtual host dedicati

Se si usa un dominio (o sottodominio) dedicato per ogni visualizzatore, la configurazione è minimale:

```apache
<VirtualHost *:443>
    ServerName lode.example.com

    ProxyPass / http://localhost:8081/
    ProxyPassReverse / http://localhost:8081/

    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-Port "443"

    # ... configurazione SSL ...
</VirtualHost>
```

### Path-based proxy (più visualizzatori sullo stesso dominio)

Se si vogliono esporre più visualizzatori sotto path diversi dello stesso dominio, è necessario configurare `SERVER_PORT`, `SERVER_SERVLET_CONTEXT_PATH` e **aggiornare le variabili d'ambiente di LODE per riflettere gli URL pubblici effettivi**.

Esempio di file `.env` per LODE in modalità path-based su `example.com`:

```env
SERVER_PORT=8081
SERVER_SERVLET_CONTEXT_PATH=/lode

# IMPORTANTE: in modalità path-based, EXTERNAL_URL e WEBVOWL_URL
# devono riflettere i path pubblici effettivi
EXTERNAL_URL=https://example.com/lode
WEBVOWL_URL=https://example.com/webvowl/#iri=

CORS_ALLOWED_ORIGINS=*
```

Configurazione Apache:

```apache
ProxyPass /lode http://localhost:8081/lode
ProxyPassReverse /lode http://localhost:8081/lode
```

> **Nota:** senza `SERVER_SERVLET_CONTEXT_PATH`, le applicazioni Spring Boot servono su `/` (root) e il path-based proxy non funzionerebbe correttamente. Per la configurazione Apache completa con tutti i visualizzatori, vedere il [README di LodView](https://github.com/teamdigitale/dati-semantic-lodview).

---

## Porte

| Porta | Protocollo | Descrizione |
|---|---|---|
| 8080 | HTTP | Interfaccia web LODE (default, configurabile con `SERVER_PORT`) |
