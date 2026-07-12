# Homelab

Configurazione Docker Compose del mio homelab. L'infrastruttura è suddivisa in stack indipendenti che condividono reti Docker esterne per proxy, database, metriche e posta.

Il repository contiene le definizioni Compose, non i dati persistenti né i segreti dell'host. Non è quindi un'installazione pronta all'uso: file `.env`, directory montate, certificati e configurazioni applicative devono essere preparati localmente.

## Struttura

| Directory | Ruolo | Servizi principali |
| --- | --- | --- |
| `network/` | Edge e servizi di rete | Traefik, Cloudflare Tunnel/DDNS, Pi-hole, Stalwart, Watchtower, Autoheal |
| `db/` | Dati e metriche condivise | PostgreSQL con pgvector, Prometheus |
| `dashboard/` | Amministrazione e osservabilità | Portainer, Grafana, cAdvisor, Node Exporter |
| `security/` | Identità, accesso e sicurezza | Authentik, Vaultwarden, NetBird, Frigate |
| `storage/` | Documenti, dati e produttività | ByteStash, Paperless-ngx, Seafile, Teable, Mathesar |
| `streaming/` | Media server e automazione | Plex, Overseerr, Sonarr, Radarr, qBittorrent, Jackett, Plex Syncer |
| `travel/` | Applicazioni di viaggio | Trek, AirTrail, Dawarich |
| `ai/` | Area AI e automazione | Stack riservato; i servizi sono attualmente disattivati nel Compose |

Le directory `console/`, `service/` e `media/` contengono configurazioni precedenti alla suddivisione corrente in `dashboard/`, `storage/` e `streaming/`. Le applicazioni locali sotto `projects/`, `website/` e `work/` sono escluse dal repository principale.

## Modello di rete

- `proxy_public` è l'unica rete usata dai servizi pubblicati tramite Traefik.
- `db_internal`, `metrics_internal` e `mail_internal` collegano gli stack alle infrastrutture condivise.
- Le reti private delle singole applicazioni isolano frontend, worker, cache e database dedicati.
- Le porte host sono riservate ai protocolli che richiedono accesso diretto; le interfacce HTTP passano normalmente da Traefik.

Le reti esterne devono esistere prima dell'avvio:

```bash
for network in proxy_public db_internal metrics_internal mail_internal; do
  docker network inspect "$network" >/dev/null 2>&1 || docker network create "$network"
done
```

## Prerequisiti

- Docker Engine con il plugin Docker Compose
- host Linux con directory e permessi compatibili con i bind mount dichiarati
- accesso alle risorse host richieste dai singoli servizi, come Docker socket e `/dev/net/tun`
- DNS e tunnel/reverse proxy configurati per i nomi host utilizzati
- NVIDIA Container Toolkit per i servizi che usano l'accelerazione GPU, come Plex

I Compose usano bind mount relativi, inclusi percorsi che risolvono nella directory `../storage` rispetto alla root del repository. Conservare questa struttura oppure aggiornare consapevolmente i mount per il proprio host.

## Configurazione

Ogni stack legge i valori specifici dell'installazione dal proprio file `.env`, che non deve essere committato. Per vedere le variabili richieste da un Compose:

```bash
cd db
docker compose config --variables
```

Preparare anche tutte le directory usate dai bind mount e assegnare l'owner richiesto dal relativo container. Prima dell'avvio, validare sempre la configurazione risolta:

```bash
docker compose config --quiet
```

## Avvio e gestione

Avviare prima rete e servizi condivisi, quindi gli stack applicativi. Per un singolo stack:

```bash
cd db
docker compose up -d
```

Comandi operativi comuni:

```bash
docker compose ps
docker compose logs -f
docker compose pull
docker compose up -d
docker compose down
```

Sull'host principale è disponibile anche un launcher esterno al repository, collocato accanto alla directory `docker/`:

```bash
../dock.sh help
../dock.sh up
```

L'ordine operativo del launcher è: `network` → `db` → `dashboard` → `security` → `storage` → `streaming` → `travel`, seguito dagli stack applicativi locali. Lo spegnimento avviene in ordine inverso.

## Persistenza e segreti

- I dati persistenti risiedono in bind mount sul filesystem dell'host; non vengono usati volumi Docker nominati.
- Password, token, chiavi e valori dipendenti dall'ambiente devono restare nei file `.env` locali.
- Prima di modificare o rimuovere un mount, verificare dove si trova il dato reale e predisporre un backup.
- `docker compose down` rimuove i container e le reti del progetto, ma non i dati salvati nei bind mount.

## Convenzioni per le modifiche

Quando si aggiunge o modifica un servizio:

1. scegliere lo stack coerente con la funzione del servizio;
2. usare bind mount per la persistenza e reti private per i sidecar;
3. esporre le UI HTTP tramite Traefik su `proxy_public`, evitando porte host non necessarie;
4. dichiarare `mem_limit`, `cpus`, policy di restart e un healthcheck quando applicabile;
5. mantenere i segreti fuori dal Compose e verificare il risultato con `docker compose config --quiet`.
