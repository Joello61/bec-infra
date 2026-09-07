# bec-infra

Orchestration Docker de Cobage (Bagage Express Cameroun). Aucun code applicatif ici — juste `docker-compose.yml` et la configuration nginx qui relie `bec-backend`, `bec-frontend`, PostgreSQL et Mercure.

Doit vivre en sibling de `bec-backend/` et `bec-frontend/` (chemins relatifs dans `docker-compose.yml`) :

```text
.
├── bec-backend/
├── bec-frontend/
├── bec-infra/       <- ce dépôt
└── bec-docs/
```

## Démarrer en local

```bash
cd bec-infra
docker compose up -d
docker compose exec backend php bin/console doctrine:migrations:migrate --no-interaction   # première fois uniquement
```

Application disponible sur `http://localhost:8000`. Hub Mercure sur `http://localhost:3001/.well-known/mercure`. Emails capturés par Mailpit (jamais livrés réellement) sur `http://localhost:8025`.

Si un PostgreSQL local tourne déjà sur le port 5432, définir `POSTGRES_HOST_PORT` avant de démarrer (ex. `POSTGRES_HOST_PORT=55432 docker compose up -d`, ou dans un `.env` local à ce dépôt, jamais committé).

Les secrets applicatifs réels (OAuth, Twilio, `JWT_PASSPHRASE`, Mercure...) vivent dans `bec-backend/.env` et `bec-frontend/.env.local` (gitignorés), montés en volume — jamais dupliqués ici.

## Arrêter

```bash
docker compose down          # conserve les données (volumes nommés)
docker compose down -v       # repartir propre (supprime aussi les volumes)
```

## État

Phase D0 (portabilité locale) terminée — voir `bec-docs/docs/deploiement/deploiement-cobage.md` pour le détail et les phases suivantes (D1-D6 : production, Traefik, observabilité, CI/CD).
