# ci-actions

Geteilte Composite Actions für die Deploy-Workflows aller Projekte (Rinhana, Hub, künftige
Projekte), die gegen turbo-svr deployen. Ersetzt den bisher in jedem Projekt kopierten
Boilerplate-Block (Intranet-Join, Infisical, GHCR-Login) durch einzelne, einmal gepflegte
Schritte.

Das Repo ist bewusst **public**, weil die Projekte unter verschiedenen GitHub-Konten/Orgs
laufen (`project-rinhana`, `eyyupa-hub`, ...) und der eingebaute `GITHUB_TOKEN` eines Workflows
keine privaten Repos anderer Owner auschecken kann. Es liegen hier keine Geheimnisse - nur
Schritt-Definitionen, die auf Secret-*Namen* verweisen. Die eigentlichen Werte kommen weiterhin
zur Laufzeit aus GitHub Secrets bzw. Infisical.

## Actions

| Action | Zweck |
|---|---|
| [`join-intranet`](./join-intranet) | Verbindet den Runner per NetBird mit dem privaten Netz |
| [`setup-infisical`](./setup-infisical) | Installiert die Infisical CLI |
| [`infisical-login`](./infisical-login) | Login per Machine Identity, gibt den Token als Output zurück |
| [`ghcr-login`](./ghcr-login) | `docker login` gegen ghcr.io |

`infisical-login` setzt voraus, dass `setup-infisical` vorher gelaufen ist.

## Beispiel

```yaml
- name: Join Intranet
  timeout-minutes: 5   # die NetBird-Action wartet sonst ohne Limit auf den Server
  uses: Allrounder-Devs/ci-actions/join-intranet@main
  with:
    setup-key: ${{ secrets.NETBIRD_SETUP_KEY }}
    management-url: ${{ secrets.NETBIRD_MANAGEMENT_URL }}

- name: Setup Infisical CLI
  uses: Allrounder-Devs/ci-actions/setup-infisical@main

- name: Infisical Login
  id: infisical
  uses: Allrounder-Devs/ci-actions/infisical-login@main
  with:
    client-id: ${{ secrets.INFISICAL_CLIENT_ID }}
    client-secret: ${{ secrets.INFISICAL_CLIENT_SECRET }}
    domain: https://infisical.turbo-svr.int.allrounder.dev

- name: GHCR Login
  uses: Allrounder-Devs/ci-actions/ghcr-login@main
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    actor: ${{ github.actor }}

# ... Build & Push, dann:

- name: Export runtime secrets
  run: |
    infisical export --token="${{ steps.infisical.outputs.token }}" \
      --domain="https://infisical.turbo-svr.int.allrounder.dev" \
      --projectId="<projekt-id>" --env="prod" --path="/" \
      --format=dotenv-export > /tmp/app.env
```

## Versionierung

Aktuell wird auf `@main` referenziert - solange nur wir selbst hier committen, ist das in
Ordnung. Sobald sich das ändert (mehr Mitwirkende, Vorsicht vor Breaking Changes), lieber auf
einen Release-Tag umstellen und den in den Projekten pinnen.

`join-intranet` bindet `Alemiz112/netbird-connect` selbst nicht per Tag, sondern per
Commit-SHA ein (Stand: `v1.0.2`) - ein Tag lässt sich nachträglich verschieben, ein SHA nicht.

## Voraussetzung: NetBird-Management-Server

`join-intranet` setzt eine laufende, selbst gehostete NetBird-Management-Instanz voraus. Die
gibt es auf turbo-svr aktuell noch **nicht** - das ist Teil des geplanten, aber noch nicht
begonnenen Umstiegs von Tailscale auf NetBird. Bis dahin bleibt der bestehende
`tailscale/github-action@v3`-Schritt in Rinhana/Hub unverändert; `join-intranet` erst
einbinden, wenn NetBird tatsächlich läuft.
