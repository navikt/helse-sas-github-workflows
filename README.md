# helse-sas-github-workflows

Gjenbrukbare GitHub Actions-workflows (`workflow_call`) for team TBD sine
applikasjoner. Workflowene lå tidligere i `navikt/helse-sp-forsikring` og er
flyttet hit slik at flere repoer kan dele dem, referere til dem med commit-SHA og
holde dem oppdatert med Dependabot.

## Workflows

| Workflow | Beskrivelse | Inputs | Outputs |
| --- | --- | --- | --- |
| `bygg-og-test-med-gradle.yml` | Bygger og tester med `./gradlew clean build`. | – | – |
| `bygg-modul-image-med-jib.yml` | Bygger et image med Jib, lager SBOM, signerer og skanner for hemmeligheter. | `modul` (påkrevd) | `image_ref` |
| `deploy.yml` | Deployer til Nais. | `CLUSTER`, `RESOURCE`, `WORKLOAD_IMAGE` (påkrevd), `VARS` (valgfri) | – |

Workflowene kjører i konteksten til den som kaller dem (`workflow_call`). Det
betyr at `actions/checkout` og `./gradlew` opererer på kall-repoet, ikke på dette
repoet.

## Bruk

Referer alltid med commit-SHA, med versjonen som kommentar, slik at Dependabot kan
holde referansen oppdatert:

```yaml
jobs:
  bygg-og-test:
    uses: navikt/helse-sas-github-workflows/.github/workflows/bygg-og-test-med-gradle.yml@<sha> # v1.0.0
    permissions:
      contents: read

  bygg-image:
    needs: bygg-og-test
    uses: navikt/helse-sas-github-workflows/.github/workflows/bygg-modul-image-med-jib.yml@<sha> # v1.0.0
    with:
      modul: min-modul
    permissions:
      actions: read
      contents: read
      id-token: write
      security-events: write

  deploy:
    needs: bygg-image
    uses: navikt/helse-sas-github-workflows/.github/workflows/deploy.yml@<sha> # v1.0.0
    with:
      CLUSTER: dev-gcp
      RESOURCE: .nais/app.yaml
      VARS: .nais/vars-dev.yaml
      WORKLOAD_IMAGE: ${{ needs.bygg-image.outputs.image_ref }}
    permissions:
      contents: read
      id-token: write
```

## Publisering og versjonering

`Release`-workflowen (`.github/workflows/release.yml`) kjører på push til `main`
når noe under `.github/workflows/` endres. Den:

1. finner høyeste eksisterende `vX.Y.Z`-tag og øker patch-nummeret,
2. oppretter en GitHub-release med den nye taggen på gjeldende commit, og
3. flytter major-aliaset (`vX`) til samme commit.

Semver-taggene gjør at Dependabot i repoene som bruker workflowene kan bumpe
SHA-referansen og oppdatere versjonskommentaren automatisk.

## Vedlikehold

`.github/dependabot.yml` holder actions som brukes inne i workflowene oppdatert.
Når en slik oppdatering merges til `main`, publiseres automatisk en ny versjon.
