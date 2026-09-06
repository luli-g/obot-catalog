# MediaCMS auf Coolify (mit OIDC-Anmeldung über authentik)

Stand: September 2026. Grundlage ist das offizielle `docker-compose.yaml` von MediaCMS,
angepasst an die Coolify-Konventionen (keine Ports, keine Traefik-Labels, Secrets über
`SERVICE_*`-Magic-Variablen).

## Images

| Dienst | Image | Anmerkung |
|---|---|---|
| MediaCMS | `mediacms/mediacms:latest` | Auf Docker Hub zuletzt am 25.08.2026 gebaut, entspricht Release v8.4.0. Versionierte Tags werden nur bis `7.2.0` veröffentlicht; wer pinnen möchte, verwendet `mediacms/mediacms:7.2.0`. |
| PostgreSQL | `postgres:17.2-alpine` | Tag wie im Upstream-Compose. |
| Redis | `redis:alpine` | Tag wie im Upstream-Compose. |

## Aufbau

- `mediacms` – nginx und gunicorn, erhaelt die Domain (Container-Port 80).
- `migrations` – Einmal-Container: `migrate`, Fixtures, Admin-Anlage, `collectstatic`.
- `celery-worker` / `celery-beat` – Transkodierung und geplante Aufgaben.
- `db`, `redis` – nur intern erreichbar, mit Healthcheck.

Die Dienste teilen sich die Volumes `mediacms-media` (Uploads, HLS) und `mediacms-static`
(Ergebnis von `collectstatic`). Der `SECRET_KEY` wird nicht mehr als Datei geteilt, sondern
über die Umgebungsvariable `SECRET_KEY` gesetzt.

Die Datei `cms/local_settings.py` wird vom Entrypoint des Images bei jedem Start aus
`deploy/docker/local_settings.py` kopiert. Deshalb wird die eigene Konfiguration über den
Compose-`configs`-Block genau an diesen Pfad gemountet.

## Variablen in der Coolify-Oberfläche

Erforderlich (die Bereitstellung startet erst nach dem Setzen):

- `OIDC_SERVER_URL` – Discovery-URL, z. B. `https://[authentik-domain]/application/o/mediacms/.well-known/openid-configuration`
- `OIDC_CLIENT_ID`
- `OIDC_CLIENT_SECRET`

Optional: `PORTAL_NAME`, `POSTGRES_DATABASE`, `TZ`, `ADMIN_USER`, `ADMIN_EMAIL`,
`OIDC_PROVIDER_ID` (Standard `authentik`, Teil der Callback-URL), `OIDC_PROVIDER_NAME`.

Von Coolify erzeugt: `SERVICE_USER_POSTGRES`, `SERVICE_PASSWORD_POSTGRES`,
`SERVICE_PASSWORD_64_REDIS`, `SERVICE_BASE64_64_MEDIACMS` (Django-`SECRET_KEY`) und
`SERVICE_PASSWORD_ADMIN` (Kennwort des Portal-Administrators, in der Oberfläche einsehbar).

Coolify vergibt für den Dienst `mediacms` eine Domain nach dem Muster
`https://mediacms-<uuid>.<wildcard-domain>:80`; der Port-Suffix bleibt beim Umbenennen erhalten.

## Einrichtung in authentik (2026.8)

1. Provider anlegen: Typ *OAuth2/OpenID Provider*, Client-Typ *Confidential*.
2. Redirect-URI (Matching-Mode *Strict*):
   `https://[mediacms-domain]/accounts/oidc/authentik/login/callback/`
   Bei abweichendem `OIDC_PROVIDER_ID` ist `authentik` im Pfad entsprechend zu ersetzen.
3. Scopes: `openid`, `email`, `profile`. Signaturschlüssel setzen.
4. Application anlegen, Slug z. B. `mediacms`, mit dem Provider verbinden.
5. `OIDC_SERVER_URL` auf die Discovery-URL der Application setzen (siehe oben).
6. Client-ID und Client-Secret in Coolify eintragen und neu bereitstellen.

## Anmeldung

Der Einstiegspunkt lautet `https://[mediacms-domain]/accounts/oidc/authentik/login/`.
Die Login-Seite von MediaCMS rendert nur das Formular für Benutzername und Kennwort; eine
Schaltfläche für den OIDC-Anbieter erscheint dort nicht automatisch und wäre über die
Portal-Anpassung zu ergänzen.

Vorhandene Konten werden anhand der Mailadresse verknüpft
(`SOCIALACCOUNT_EMAIL_AUTHENTICATION`), neue Konten werden ohne Zwischenschritt angelegt
(`SOCIALACCOUNT_AUTO_SIGNUP`). Wer das nicht wünscht, setzt beide Werte in der
`local_settings.py` im `configs`-Block auf `False`.
