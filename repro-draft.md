# NomNom #220 — Reproduction Report

Issue: https://github.com/WorldconVotingSystems/nomnom/issues/220
Date: October 31, 2026
Result: **Reproduced** in a local development environment (details and
limitations below).

## Summary

Clicking "Email my nominations to me" produced an email whose subject and
first body line contain a timestamp with no time-zone identifier. This is the
behavior the issue reports. The UTC origin of the timestamp is supported by
log output from an earlier attempt (see Evidence), not by the successful
email itself.

## Environment

Source
- Repository: WorldconVotingSystems/nomnom, cloned and run from source (no
  released package)
- Branch: `main`
- Commit: `22d531f04da7748ac0daee7f5d7d8e59981d1f77`
- Application source: unchanged (`git status` showed only the untracked
  `repro-draft.md`)

Platform and tools
- macOS on Apple Silicon (arm64)
- Python 3.13.5; Django 6.0.8; Celery 5.6.3; uv 0.13.6
- Docker Compose services: PostgreSQL 17, Redis 7, MailCrab
  (`marlonb/mailcrab:latest`)

Time-zone and mail settings (as read from the running configuration)
- Django `TIME_ZONE`: `UTC`
- `CELERY_TIMEZONE`: `America/Los_Angeles`
- `EMAIL_USE_TLS`: `False`

Local state changed for testing (not application source)
- A local, git-ignored `.env` (created by `just bootstrap`) was edited:
  - `NOM_DB_PORT=55432` and `NOM_REDIS_PORT=56379`, because the default host
    ports were already in use by other services.
  - `NOM_EMAIL_USE_TLS=false`, because MailCrab does not support STARTTLS.
- The database was populated by `just bootstrap` and then modified through
  Django Admin and a shell command (election state and a member profile; see
  Steps 6 and 7).
- The MailCrab SMTP host port is assigned dynamically by Docker Compose
  (`docker-compose.yml` publishes container port 1025 without a fixed host
  port).

Difference from the issue's environment: the reporter ran a deployed
convention site; this is a local development setup. The issue says the time
"currently seems to be UTC", which matches the `TIME_ZONE=UTC` setting here,
but I have not tested a deployment with a different `TIME_ZONE`.

## Expected behavior

From the issue: the timestamp should "either explicitly specify what time zone
it's in, or default to the time zone of the convention". The maintainer's
reply says it "should be in Pacific time".

## Steps to reproduce

Prerequisites from `docs/docs/dev/bootstrap.md`: Just, gsed, uv, and Docker
with the compose plugin.

1. Clone the repository and check out the tested commit:
   ```console
   git clone https://github.com/WorldconVotingSystems/nomnom.git
   cd nomnom
   git checkout 22d531f04da7748ac0daee7f5d7d8e59981d1f77
   ```
2. Bootstrap the environment. This creates `.env` if it is missing, starts the
   Docker services, runs migrations, and loads seed data (including The Yugo
   Awards election and an `admin` user), then stops the containers:
   ```console
   just bootstrap
   ```
   If the PostgreSQL or Redis host ports are already in use, bootstrap fails.
   In that case set unused ports in `.env` (I used `NOM_DB_PORT=55432` and
   `NOM_REDIS_PORT=56379`) and run `just bootstrap` again.
3. Before starting the worker, edit `.env` and add:
   ```
   NOM_EMAIL_USE_TLS=false
   ```
   MailCrab does not support STARTTLS. Without this, sending failed with
   `SMTPNotSupportedError: STARTTLS extension not supported by server.`
4. Start the web server (this also starts the Docker services):
   ```console
   just serve
   ```
   The site is at `http://localhost:8000/`.
5. In a second terminal, from the repository root, start a Celery worker.
   `DJANGO_SETTINGS_MODULE` is set explicitly because
   `src/nomnom/celery.py` defaults to `config.settings`, and `.env` is loaded
   explicitly:
   ```console
   DJANGO_SETTINGS_MODULE=nomnom_dev.settings uv run --env-file .env celery -A nomnom worker --loglevel=INFO --pool=solo
   ```
   I did not use `just worker`: its recipe requires `fd` and `entr` and does
   not set `DJANGO_SETTINGS_MODULE`.
6. Open the election in the browser (development-only seeded account):
   - Log in to the local NomNom application at `http://localhost:8000/` with
     username `admin`, password `admin`.
   - Open Admin Dashboard, then Nominate > Elections > The Yugo Awards.
   - Change State from Pre-Nomination to Nominating.
   - Leave the other fields unchanged and click Save.
   - Return to View Site and verify that Nominations are open.
7. Give the seeded `admin` user a nominating member profile. Run this from the
   repository root (the `admin` account already exists; no new user is
   created):
   ```console
   uv run --env-file .env manage.py shell -c 'from django.contrib.auth import get_user_model; from nomnom.nominate.models import NominatingMemberProfile; u = get_user_model().objects.get(username="admin"); p, created = NominatingMemberProfile.objects.get_or_create(user=u, defaults={"preferred_name": "Local Test Admin"}); print("Profile created:", created, "| recipient:", u.email)'
   ```
   Observed output in my run:
   ```
   Profile created: True | recipient: admin@site.com
   ```
   `get_or_create` also works if the profile already exists (it prints
   `Profile created: False` and changes nothing). This changes only the seeded
   local development database.
8. On the site, click the green "Click to Nominate" button to open the
   nomination form for The Yugo Awards, and in the Best Novel category enter:
   - Title: `AI301 Test Novel`
   - Author: `Test Author`
   - Publisher: `Test Publisher`
9. Click "Email my nominations to me". The page reports that an email will be
   sent, and the Celery worker logs execution of `send_ballot`.
10. Open the MailCrab web interface:
    ```console
    just mailcatcher
    ```
    Open the newest message and read its subject and first body line.

Note on the MailCrab port: do not hardcode an SMTP port in `.env`. Docker
Compose assigns the MailCrab host ports dynamically, so they can differ on
another machine or after the containers restart. In my successful run the
environment check reported `EMAIL_HOST=localhost` and `EMAIL_PORT=49673`, but
that value is specific to that run and should not be copied. To see the
current host ports:

```console
docker compose port mailcatcher 1025   # SMTP
docker compose port mailcatcher 1080   # web interface
```

`nomnom_dev/settings.py` runs `docker compose port mailcatcher 1025` when
Django loads and sets `NOM_EMAIL_PORT` from the result, so Django and the
Celery worker (started with `DJANGO_SETTINGS_MODULE=nomnom_dev.settings`, Step
5) pick up the current port automatically. `just mailcatcher` likewise looks
up the web port itself.

## Observed behavior (confirmed)

NomNom saved the nomination and showed a message that an email would be sent.
Celery executed `send_ballot`. MailCrab captured an email to
`admin@site.com`.

Captured subject (verbatim):

```
Your The Yugo Awards Nominations - Oct. 31, 2026, 6:54 a.m.
```

Captured body first line (verbatim):

```
Here are your The Yugo Awards nominations, as of Oct. 31, 2026, 6:54 a.m.
```

Both timestamps omit any time-zone identifier (no "UTC", "PDT", "PST", or
similar).

Comparison, computed by me: 6:54 a.m. UTC on Oct. 31, 2026 is
11:54 p.m. PDT on Oct. 30, 2026 (UTC-7). A reader who assumed Pacific time
would misread the timestamp by about seven hours and would see the wrong
date.

## Evidence

1. MailCrab screenshot, captured October 31, 2026, of the email above.
2. Log evidence that the generated value is UTC. This comes from an **earlier
   attempt** in which delivery failed on the STARTTLS error, not from the
   successful email above. The Celery failure logs showed:
   ```
   report_date: datetime(2026, 10, 31, 6, 46, 34, 887751, tzinfo=timezone.utc)
   formatted value: Oct. 31, 2026, 6:46 a.m.
   ```
   That run's timestamp (6:46 a.m.) differs from the successful email's
   (6:54 a.m.) because they were separate attempts. The UTC evidence and the
   captured email are therefore from different runs; I did not capture the
   `report_date` value for the successful run.

## What is confirmed and what is not

Confirmed by observation
- The nomination email subject and first body line show a time with no
  time-zone label (verbatim above).
- In an earlier attempt, `send_ballot` produced a UTC-aware `report_date`
  that formatted to the same style of unlabeled string.
- The displayed time matches UTC, not Pacific, for the successful email
  (6:54 a.m. versus 11:54 p.m. PDT on the previous day).

Hypothesis (not established)
- The likely cause is that `send_ballot` in
  `src/nomnom/nominate/tasks.py` creates `datetime.now(UTC)` and formats it
  with Django's `localize()` without converting to Pacific time or adding a
  time-zone label. I read the source but did not change it or test a fix, so
  this is an inference.
- Whether the email would show Pacific time under a different `TIME_ZONE`
  setting is untested.

Not tested
- Other versions or commits, a deployed environment, and the HTML versus
  plain-text email parts individually.

Relevant files read
- `src/nomnom/nominate/tasks.py` (`send_ballot`)
- `src/nomnom/nominate/views/nominate.py` (email task dispatch)
- `src/nomnom/nominate/templates/nominate/email/` (plain-text and HTML
  templates)

## Troubleshooting encountered

- Bootstrap first failed because the default PostgreSQL and Redis host ports
  were in use; changing them in `.env` fixed it.
- Running Celery without `DJANGO_SETTINGS_MODULE` and the `.env` variables
  failed; the command in Step 5 supplies both.
- Email delivery first failed with
  `SMTPNotSupportedError: STARTTLS extension not supported by server.`
  Setting `NOM_EMAIL_USE_TLS=false` in the local `.env` fixed it. This is a
  local MailCrab accommodation only.

## AI assistance

I used Claude Code to help read the repository, run the local environment, and
draft this report. I ran the steps and captured the outputs myself.
