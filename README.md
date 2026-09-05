# vps-monitoring

Observabilité du VPS, **tous projets confondus** (`ustdy_production`,
`ustdy_staging`, `ustudyguide_vitrine`, `aitalkmentor`, et ceux à venir).

Ce dépôt est indépendant des dépôts applicatifs : la stack se déploie une fois
pour la machine, pas une fois par projet.

## Ce que ça répond

- Quel conteneur consomme quoi, **maintenant** et **il y a trois jours**.
- Quel projet occupe le VPS, en RAM et en CPU.
- Lequel *dérive* : mémoire qui monte sans redescendre, processus qui
  s'accumulent, conteneur qui redémarre en boucle.

## Composition

| Service | Rôle | Exposé |
|---|---|---|
| `cadvisor` | CPU / RAM / réseau / nb de processus **par conteneur** | non |
| `node-exporter` | CPU / RAM / disque / I/O **de l'hôte** | non |
| `prometheus` | stockage 30 j (plafonné à 4 Go) + règles d'alerte | non |
| `grafana` | dashboards | `https://$MONITORING_DOMAIN` |

Empreinte plafonnée à **1,6 Go de RAM et 2,25 vCPU** par les limites du compose ;
consommation réelle attendue autour de 550–700 Mo au repos.

Aucun `ports:` n'est publié. Seul Grafana sort, via Traefik ; les trois autres
services ne sont joignables que sur le réseau `monitoring_internal`.

**Rien n'est à déclarer par projet** : cAdvisor découvre tous les conteneurs du
démon Docker. Les étiquettes `com.docker.compose.project` et `.service` sont
conservées, ce qui permet de filtrer et de regrouper par projet dans Grafana.
Le seul endroit qui nomme des conteneurs est l'alerte `ConteneurDisparu`, à
mettre à jour quand un projet critique arrive ou part du VPS.

## Installation

### 1. DNS

Créer un enregistrement **A** pour le domaine de monitoring → IP du VPS.

Le resolver Traefik dépend de l'hébergeur de la zone, et c'est le piège
classique de cette infra :

| Zone | Hébergeur | `MONITORING_CERT_RESOLVER` |
|---|---|---|
| `ustudyguide.com` | LWS | `letsencrypt` (HTTP-01) |
| `ustdy.com` et wildcards | Cloudflare | `letsencrypt-dns` |

Avec le choix par défaut (`monitor.ustudyguide.com`), c'est **`letsencrypt`**.
Le challenge HTTP-01 impose que l'entrypoint `web` (port 80) reste joignable
depuis Internet, et que le DNS soit propagé **avant** le premier démarrage.

Un routeur pointé sur le mauvais resolver n'échoue pas visiblement : Traefik
sert silencieusement son certificat auto-signé `TRAEFIK DEFAULT CERT`.

### 2. Fichiers sur le VPS

```bash
ssh <user>@<vps>
sudo mkdir -p /opt/monitoring && sudo chown $USER: /opt/monitoring
# depuis le poste de dev :
scp -r ./* ./.env.example <user>@<vps>:/opt/monitoring/
```

### 3. Secrets

```bash
cd /opt/monitoring
cp .env.example .env

# Mot de passe admin Grafana
sed -i "s|^GF_SECURITY_ADMIN_PASSWORD=.*|GF_SECURITY_ADMIN_PASSWORD=$(openssl rand -base64 24)|" .env

# BasicAuth Traefik — le sed double les $, que Compose consomme ensuite
sudo apt-get install -y apache2-utils
htpasswd -nb admin 'VotreMotDePasse' | sed -e 's/\$/\$\$/g'
# → coller le résultat dans MONITORING_BASIC_AUTH=

chmod 600 .env
```

**Le doublement des `$` est obligatoire** et c'est le seul réglage de cette
stack qui se trompe en silence. Comportement vérifié :

| Valeur dans `.env` | Label réellement posé |
|---|---|
| `admin:$$apr1$$AbCd$$xyz` | `admin:$apr1$AbCd$xyz` ✅ |
| `admin:$apr1$AbCd$xyz` | `admin:` ❌ |

Avec des `$` simples, Compose interprète `$apr1` et `$AbCd` comme des variables
d'environnement vides : le hash disparaît et le middleware se retrouve sans
utilisateur — accès refusé à tout le monde, sans message d'erreur.

Ne pas contrôler avec `docker compose config` : sa sortie ré-échappe les `$` en
`$$`, donc elle affiche des doubles dans les deux cas et ne distingue rien. La
vérification fiable se fait après démarrage, sur le conteneur :

```bash
docker inspect monitoring-grafana \
  --format '{{index .Config.Labels "traefik.http.middlewares.monitoring-auth.basicauth.users"}}'
```

La sortie doit contenir des `$` **simples**.

### 4. Démarrage

```bash
docker compose up -d
docker compose ps
```

Traefik doit rejoindre le réseau de la stack pour router Grafana. S'il ne le
fait pas de lui-même (selon sa configuration sur le VPS) :

```bash
docker network connect monitoring_internal <nom-du-conteneur-traefik>
```

### 5. Vérification

```bash
# Les trois cibles répondent-elles ?
docker compose exec prometheus wget -qO- localhost:9090/api/v1/targets \
  | grep -o '"health":"[a-z]*"'          # trois fois "up"

# cAdvisor voit-il bien tous les projets du VPS ?
docker compose exec prometheus wget -qO- \
  'localhost:9090/api/v1/label/container_label_com_docker_compose_project/values'

# Le certificat est-il le bon ?
echo | openssl s_client -connect monitor.ustudyguide.com:443 \
  -servername monitor.ustudyguide.com 2>/dev/null | grep -E "issuer|subject"
# issuer doit mentionner Let's Encrypt, jamais TRAEFIK DEFAULT CERT
```

Puis ouvrir le domaine → basicauth → login Grafana (`admin`).

## Le dashboard

`UStudyGuide / VPS — Ressources & conteneurs`, provisionné automatiquement, avec
une variable **Projet** en haut pour isoler un projet ou tout afficher.

Quatre panneaux portent le diagnostic :

- **Mémoire par projet** — qui occupe le VPS, vue empilée.
- **Mémoire par conteneur** — désigne le conteneur fautif à l'intérieur du projet.
- **Nombre de processus par conteneur** — gunicorn et celery sont en préfork,
  cette courbe doit être **plate**. Si elle monte, des processus enfants ne sont
  pas récupérés : signature d'un Chromium Playwright jamais fermé
  (`usg_ges_ets/apps/notes/pdf_utils.py:124`).
- **Croissance mémoire sur 24 h** — classement par mémoire *gagnée*. Une charge
  normale revient à zéro, une fuite reste positive jour après jour.

Dashboards communautaires en complément (Grafana → Import par ID, nécessite un
accès Internet sortant) : **193** (Docker/cAdvisor), **1860** (Node Exporter Full).

Les dashboards du dépôt sont montés en lecture seule (`allowUiUpdates: false`) :
une modification faite dans l'UI est écrasée au redémarrage. Pour la conserver,
l'exporter en JSON et la committer dans `grafana/dashboards/`.

## Alertes

Règles dans `prometheus/alertes.yml`, consultables dans Prometheus :

```bash
docker compose exec prometheus wget -qO- localhost:9090/api/v1/rules
```

**Hôte** — RAM > 85 % / 95 %, swap > 25 %, CPU > 80 %, disque < 15 %.
**Conteneurs** — > 85 % de sa limite, crashloop, prolifération de processus,
conteneur critique disparu, cible de scrape injoignable.

`FuiteMemoireProbable` est la règle qui aurait attrapé le problème avant le
redémarrage : elle extrapole la tendance des 6 dernières heures sur 24 h et se
déclenche si la projection dépasse la limite du conteneur.

**Ces règles s'affichent mais ne notifient pas encore.** Pour recevoir des
mails, brancher un point de contact Grafana (Alerting → Contact points) sur
`ALERTS{alertstate="firing"}`, ou ajouter un conteneur Alertmanager.

Trois règles (`ConteneurProcheDeSaLimite`, `FuiteMemoireProbable`, et le panneau
« Part de la limite mémoire ») dépendent des `deploy.resources.limits` posés sur
les stacks applicatives : sans limite, `container_spec_memory_limit_bytes` vaut
0 et elles restent muettes.

## Exploitation

```bash
docker compose logs -f grafana
docker compose restart prometheus                        # après édition de prometheus/
docker compose exec prometheus wget -qO- --post-data='' \
  localhost:9090/-/reload                                # recharge à chaud
docker compose down                                      # les volumes sont conservés
```
