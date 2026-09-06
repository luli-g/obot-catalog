# MediaCMS auf Coolify (vorbereitet für OIDC über authentik)

Stand: September 2026. Grundlage ist das offizielle `docker-compose.yaml` von MediaCMS,
angepasst an die Coolify-Konventionen (keine Ports, keine Traefik-Labels, Secrets über
`SERVICE_*`-Magic-Variablen).

Vorgesehene Adressen: MediaCMS unter `https://lernplattform.funke-service.com`,
authentik unter `https://apps.funke-service.com`.

## Images

| Dienst | Image | Anmerkung |
|---|---|---|
| MediaCMS | `mediacms/mediacms:latest` | Auf Docker Hub zuletzt am 25.08.2026 gebaut, entspricht Release v8.4.0. Versionierte Tags werden nur bis `7.2.0` veröffentlicht; wer pinnen möchte, verwendet `mediacms/mediacms:7.2.0`. |
| PostgreSQL | `postgres:17.2-alpine` | Tag wie im Upstream-Compose. |
| Redis | `redis:alpine` | Tag wie im Upstream-Compose. |

## Aufbau

- `mediacms` – nginx und gunicorn, erhält die Domain (Container-Port 80).
- `migrations` – Einmal-Container: `migrate`, Fixtures, Admin-Anlage, `collectstatic`.
- `celery-worker` / `celery-beat` – Transkodierung und geplante Aufgaben.
- `db`, `redis` – nur intern erreichbar, mit Healthcheck.

Die Dienste teilen sich die Volumes `mediacms-media` (Uploads, HLS) und `mediacms-static`
(Ergebnis von `collectstatic`). Der `SECRET_KEY` wird nicht als Datei geteilt, sondern über
die Umgebungsvariable `SECRET_KEY` gesetzt.

Die Datei `cms/local_settings.py` wird vom Entrypoint des Images bei jedem Start aus
`deploy/docker/local_settings.py` kopiert. Deshalb wird die eigene Konfiguration über den
Compose-`configs`-Block genau an diesen Pfad gemountet.

## Domain in Coolify

Nach dem Anlegen des Dienstes ist die automatisch erzeugte Domain des Dienstes `mediacms`
auf `https://lernplattform.funke-service.com:80` zu ändern. Der Port-Suffix `:80` bleibt
erhalten, er steuert nur die Weiterleitung von Traefik auf den Container-Port. Aus dieser
Domain speist sich `FRONTEND_HOST` und damit auch `CSRF_TRUSTED_ORIGINS`.

## Variablen in der Coolify-Oberfläche

Keine Variable blockiert die erste Bereitstellung. Optional sind `PORTAL_NAME`,
`POSTGRES_DATABASE`, `TZ`, `ADMIN_USER`, `ADMIN_EMAIL`, `OIDC_PROVIDER_ID`
(Standard `authentik`, Teil der Callback-URL) und `OIDC_PROVIDER_NAME`.

Von Coolify erzeugt werden `SERVICE_USER_POSTGRES`, `SERVICE_PASSWORD_POSTGRES`,
`SERVICE_PASSWORD_64_REDIS`, `SERVICE_BASE64_64_MEDIACMS` (Django-`SECRET_KEY`) und
`SERVICE_PASSWORD_ADMIN` (Kennwort des Portal-Administrators, in der Oberfläche einsehbar).

## OIDC später aktivieren

Die Anbindung ist vorbereitet, aber inaktiv: Der Provider wird in der `local_settings.py`
erst registriert, wenn `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET` und `OIDC_SERVER_URL` gesetzt
sind. `OIDC_SERVER_URL` ist mit
`https://apps.funke-service.com/application/o/mediacms/.well-known/openid-configuration`
vorbelegt; abweichend ist der Wert an den Slug der Application in authentik anzupassen.

Schritte in authentik (2026.8):

1. Provider anlegen: Typ *OAuth2/OpenID Provider*, Client-Typ *Confidential*.
2. Redirect-URI (Matching-Mode *Strict*):
   `https://lernplattform.funke-service.com/accounts/oidc/authentik/login/callback/`
   Bei abweichendem `OIDC_PROVIDER_ID` ist `authentik` im Pfad entsprechend zu ersetzen.
3. Scopes `openid`, `email`, `profile` zuweisen und einen Signaturschlüssel setzen.
4. Application mit dem Slug `mediacms` anlegen und mit dem Provider verbinden.
5. Client-ID und Client-Secret in Coolify eintragen und neu bereitstellen.

Der Einstiegspunkt lautet dann
`https://lernplattform.funke-service.com/accounts/oidc/authentik/login/`. Die Login-Seite
von MediaCMS rendert nur das Formular für Benutzername und Kennwort; eine Schaltfläche für
den OIDC-Anbieter erscheint dort nicht automatisch und wäre über die Portal-Anpassung zu
ergänzen.

Vorhandene Konten werden anhand der Mailadresse verknüpft
(`SOCIALACCOUNT_EMAIL_AUTHENTICATION`), neue Konten werden ohne Zwischenschritt angelegt
(`SOCIALACCOUNT_AUTO_SIGNUP`). Wer das nicht wünscht, setzt beide Werte in der
`local_settings.py` im `configs`-Block auf `False`.
