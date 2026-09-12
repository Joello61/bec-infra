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
docker compose exec backend php -d memory_limit=1G bin/console app:import-geodata          # première fois uniquement (~3 min, ~40k villes)
docker compose exec backend php bin/console app:seed:currency                              # première fois uniquement
docker compose exec backend php bin/console app:seed:subscription-plans                    # première fois uniquement
```

Application disponible sur `http://localhost:8000` — toujours ce nom d'hôte, jamais `127.0.0.1` : `JWT_COOKIE_DOMAIN=localhost` (`bec-backend/.env`) fait qu'un cookie de session posé pour `localhost` n'est jamais envoyé par le navigateur à `127.0.0.1`, malgré la même boucle locale (constaté en Phase 7b-A, `bec-docs/docs/plan-correction/plan-correction-cobage.md`). Hub Mercure sur `http://localhost:3001/.well-known/mercure`. Emails capturés par Mailpit (jamais livrés réellement) sur `http://localhost:8025`.

Sans les deux premières commandes de seed ci-dessus, `countries`/`cities`/`currencies` restent vides : la sélection de ville dans les formulaires de voyage/demande n'affiche jamais de résultat, et compléter son profil plante en 500 (`AddressService` ne trouve aucun pays correspondant). `memory_limit=1G` sur `app:import-geodata` : l'import des ~40k villes dépasse la limite par défaut de PHP (128M) autour de 15-20% de progression. Sans `app:seed:subscription-plans` (monétisation Lot 1), le plan "free" n'existe pas et **toute création de voyage/demande échoue en 500** (`SubscriptionService::getEffectivePlan()` ne trouve aucun plan par défaut, consommé par le quota freemium dans `VoyageVoter`/`DemandeVoter`).

Si un PostgreSQL local tourne déjà sur le port 5432, définir `POSTGRES_HOST_PORT` avant de démarrer (ex. `POSTGRES_HOST_PORT=55432 docker compose up -d`, ou dans un `.env` local à ce dépôt, jamais committé).

Les secrets applicatifs réels (OAuth, Twilio, `JWT_PASSPHRASE`, Mercure...) vivent dans `bec-backend/.env` et `bec-frontend/.env.local` (gitignorés), montés en volume — jamais dupliqués ici.

## Créer le premier administrateur

Aucun endpoint API ne permet de créer un `ROLE_ADMIN` (l'API de gestion des rôles exige déjà d'être admin — problème de l'œuf et de la poule sur une base neuve, cf. `bec-docs/docs/plan-correction/plan-correction-cobage.md`, Phase 7b-B). Étape manuelle, à faire une fois par environnement :

```bash
# 1. Créer un compte normal via l'UI (http://localhost:8000/auth/register) ou l'API
# 2. Le promouvoir administrateur
docker compose exec backend php bin/console app:user:promote-admin admin@example.com
```

Opération inverse en dernier recours (incident de sécurité, etc.) : `app:user:revoke-admin <email>` (refuse de retirer le dernier admin sans `--force`). Les deux commandes ne sont accessibles qu'en CLI, jamais via HTTP.

## Lancer les tests backend

Toujours dans le conteneur `backend`, jamais avec un PHP/PostgreSQL natif sur la machine hôte (c'est tout le sens de la portabilité de la Phase D0). `config/packages/doctrine.yaml` de `bec-backend` isole automatiquement la base de test par suffixe (`dbname_suffix: _test`), mais cette base n'existe pas tant qu'elle n'a pas été créée une première fois sur le volume Postgres du compose :

```bash
docker compose exec backend php bin/console doctrine:database:create --env=test   # première fois uniquement, ou après un docker compose down -v
docker compose exec backend php bin/console doctrine:migrations:migrate --env=test --no-interaction
docker compose exec backend php bin/phpunit --testdox
```

## Arrêter

```bash
docker compose down          # conserve les données (volumes nommés)
docker compose down -v       # repartir propre (supprime aussi les volumes)
```

## État

Phase D0 (portabilité locale) terminée — voir `bec-docs/docs/deploiement/deploiement-cobage.md` pour le détail et les phases suivantes (D1-D6 : production, Traefik, observabilité, CI/CD).
