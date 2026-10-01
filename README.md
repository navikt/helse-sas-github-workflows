# sykepenger-github-workflows

Gjenbrukbare GitHub Actions-workflows (`workflow_call`) og actions for team TBD
sine applikasjoner. Workflowene lå tidligere i `navikt/helse-sp-forsikring` og er
flyttet hit slik at flere repoer kan dele dem, referere til dem med commit-SHA og
holde dem oppdatert med Dependabot.

## Workflows

| Workflow | Beskrivelse | Inputs | Outputs |
| --- | --- | --- | --- |
| `bygg-og-test-med-gradle.yml` | Bygger og tester med `./gradlew clean build`. | – | – |
| `bygg-modul-image-med-jib.yml` | Bygger en modul som et image med Jib, lager SBOM, signerer og skanner for hemmeligheter. | `modul` (påkrevd) | `image_ref` |
| `bygg-image-med-jib.yml` | Som over, men for enkeltmodulprosjekter. | – | `image_ref` |
| `deploy.yml` | Deployer til Nais med `nais/deploy/actions/deploy` v3. Støtter eksisterende Kubernetes-manifester og Handlebars-templating. | `CLUSTER`, `RESOURCE`, `WORKLOAD_IMAGE` (påkrevd), `VARS` (valgfri) | – |
| `deploy-v2.yml` | Deployer til Nais med `nais/setup` og `nais apply`. Krever manifester som bruker mixins. | `manifest`, `environment` (påkrevd), `image`, `extra_manifests` (valgfrie) | – |

Workflowene kjører i konteksten til den som kaller dem (`workflow_call`). Det
betyr at `actions/checkout` og `./gradlew` opererer på kall-repoet, ikke på dette
repoet.

`bygg-modul-image-med-jib.yml` kjører `./gradlew :<modul>:jib` og brukes av
multimodulprosjekter. `bygg-image-med-jib.yml` kjører `./gradlew jib` og
brukes av enkeltmodulprosjekter, der rotmodulen er den som blir til et image
(altså der `no.nav.helse.sas.sas-deployable` er lagt på rotprosjektet). Bortsett
fra hvilken modul som bygges er de to like.

## Actions

### `copilot-setup-steps`

Setter opp miljøet til Copilot-agenten. Actionen sjekker ut kall-repoet,
installerer Nav-tilpasninger og skills fra `navikt/helse-sas-meta` med
nav-pilot, og gjør valgte verktøy tilgjengelige. Den installerer ikke
prosjektavhengigheter. Det gjør agenten selv ved behov.

| Input | Standard | Beskrivelse |
| --- | --- | --- |
| `kotlin` | `false` | Installerer JDK og setter opp Gradle. |
| `node` | `false` | Installerer Node.js og pnpm. |
| `package-json` | `package.json` | Fila som bestemmer pnpm-versjonen (`packageManager`) og Node-versjonen (`engines.node`). |

Copilot kjører bare stegene i en jobb som heter `copilot-setup-steps` i repoets
egen `.github/workflows/copilot-setup-steps.yml`. Derfor er dette en composite
action og ikke en gjenbrukbar workflow. Et Gradle-prosjekt med frontend i en
undermappe ser slik ut:

```yaml
jobs:
  copilot-setup-steps:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    permissions:
      contents: read
    steps:
      - uses: navikt/sykepenger-github-workflows/.github/actions/copilot-setup-steps@<sha> # v1.0.0
        with:
          kotlin: true
          node: true
          package-json: frontend/package.json
```

## Bruk

Referer alltid med commit-SHA, med versjonen som kommentar, slik at Dependabot kan
holde referansen oppdatert:

```yaml
jobs:
  bygg-og-test:
    uses: navikt/sykepenger-github-workflows/.github/workflows/bygg-og-test-med-gradle.yml@<sha> # v1.0.0
    permissions:
      contents: read

  bygg-image:
    needs: bygg-og-test
    uses: navikt/sykepenger-github-workflows/.github/workflows/bygg-modul-image-med-jib.yml@<sha> # v1.0.0
    with:
      modul: min-modul
    permissions:
      actions: read
      contents: read
      id-token: write
      security-events: write

  deploy:
    needs: bygg-image
    uses: navikt/sykepenger-github-workflows/.github/workflows/deploy.yml@<sha> # v1.0.0
    with:
      CLUSTER: dev-gcp
      RESOURCE: .nais/app.yaml
      VARS: .nais/vars-dev.yaml
      WORKLOAD_IMAGE: ${{ needs.bygg-image.outputs.image_ref }}
    permissions:
      contents: read
      id-token: write
```

Med `deploy-v2.yml` ser deploy-jobben slik ut i stedet:

```yaml
  deploy:
    needs: bygg-image
    uses: navikt/sykepenger-github-workflows/.github/workflows/deploy-v2.yml@<sha> # v1.0.0
    with:
      manifest: .nais/app.yaml
      environment: dev-gcp
      image: ${{ needs.bygg-image.outputs.image_ref }}
    permissions:
      contents: read
      id-token: write
```

`extra_manifests` tar en kommaseparert liste med manifester som deployes før
basismanifestet, for eksempel `.nais/dev-db-policy.yaml`. De får ikke satt
`spec.image`. Lar du `image` stå tom, beholder `nais apply` imaget som kjører
nå, slik at du kan deploye manifestendringer uten å bygge på nytt. Uten `image`
kan `manifest` også være en ressurs uten image, for eksempel et Kafka-`Topic`,
og den kan da inneholde flere YAML-dokumenter så lenge den ikke har mixins.

## Publisering og versjonering

`Release`-workflowen (`.github/workflows/release.yml`) kjører på push til `main`
når noe under `.github/workflows/` eller `.github/actions/` endres. Den:

1. finner høyeste eksisterende `vX.Y.Z`-tag og øker patch-nummeret,
2. oppretter en GitHub-release med den nye taggen på gjeldende commit, og
3. flytter major-aliaset (`vX`) til samme commit.

Semver-taggene gjør at Dependabot i repoene som bruker workflowene kan bumpe
SHA-referansen og oppdatere versjonskommentaren automatisk.

## Vedlikehold

`.github/dependabot.yml` holder actions som brukes inne i workflowene og i
`.github/actions/*` oppdatert.
Når en slik oppdatering merges til `main`, publiseres automatisk en ny versjon.
