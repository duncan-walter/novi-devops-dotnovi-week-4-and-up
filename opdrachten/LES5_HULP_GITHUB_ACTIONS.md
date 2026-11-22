# Technische hulp B – GitHub Actions uitbreiden met Azure-deployment

> **Bij Praktijk B, opdracht B2.** Deze instructie hoort bij [LES5.md](./LES5.md). Je hebt de OIDC-voorbereidingsopdracht na les 4 voltooid en een werkende dotNOVI-deployment uit Praktijk A.

## Wat ga je doen?

Breid je **bestaande CI-workflow** uit. Na succesvolle tests en linting bouwt GitHub Actions een nieuwe image, publiceert deze naar jouw Azure Container Registry (ACR) en werkt de bestaande Container App bij.

Je gebruikt de eerder aangemaakte identity `id-github-dotnovi`, niet de identity `id-acr-pull-dotnovi`. Deze laatste is alleen bedoeld om vanuit Container Apps images op te halen.

## 1. Geef de GitHub-identity pushrechten op ACR

1. Open jouw **Azure Container Registry**.
2. Ga naar **Access control (IAM) → Add → Add role assignment**.
3. Kies de rol **AcrPush**.
4. Selecteer bij Members de **managed identity** `id-github-dotnovi`.
5. Rond de roltoewijzing af **op de registry**, niet op de gehele subscription.

Deze instructie gaat uit van **RBAC Registry Permissions**, zoals ingesteld in Praktijk A. Bij een ABAC-registry is een andere repositorywriter-rol nodig.

> Je managed identity moet daarnaast al voldoende rechten hebben om de bestaande Container App bij te werken; dit heb je voorbereid na les 4. OIDC-login betekent niet automatisch dat je ook alle gewenste Azure-acties mag uitvoeren.

## 2. Vul de deploymentjob in

Onderstaande YAML is een **gedeeltelijk jobfragment**. Voeg het toe onder `jobs:` in je bestaande workflow. Neem niet opnieuw een volledige CI-pipeline over.

```yaml
  deploy:
    needs: [test, lint]       # Pas aan naar de echte namen van jouw CI-jobs
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest

    permissions:
      id-token: write
      contents: read

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Login to Azure via OIDC
        uses: azure/login@v3
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      # TODO 1: Authenticate Docker to your Azure Container Registry.
      # TODO 2: Build a Docker image and tag it as v2.
      # TODO 3: Push the v2 image to ACR.
      # TODO 4: Update your existing Azure Container App to use v2.
```

- Pas `needs` aan je bestaande workflow aan. Staan test en lint in dezelfde job, gebruik dan de naam van die ene job.
- De branch in de `if`-voorwaarde moet overeenkomen met de branch in je **federated credential**. Gebruik een andere naam als je geen `main` hebt.
- De GitHub Actions-secrets `AZURE_CLIENT_ID`, `AZURE_TENANT_ID` en `AZURE_SUBSCRIPTION_ID` heb je al tijdens de voorbereiding ingesteld.
- Plaats **geen wachtwoorden, tokens of registry admin-credentials** in de YAML.

## 3. Vul de vier TODO's zelf in

| Doel | Aanwijzing |
| --- | --- |
| Docker aanmelden bij ACR | Zoek de Azure CLI-opdracht `az acr login`. |
| Image bouwen | Gebruik `docker build` met de volledige image-URL en de tag `v2`. |
| Image publiceren | Gebruik `docker push`. |
| Container App bijwerken | Zoek `az containerapp update --image` en geef naam en resource group mee. |

De image heeft de vorm:

```text
<login-server>/<repository>:v2
```

Gebruik de **werkelijke login server** uit de Azure Portal; vul die niet op basis van een veronderstelde registrynaam in. Zorg dat de workflow `v2` gebruikt bij **zowel push als deploy**.

Voor deze eerste begeleide ronde mag je `v2` gebruiken. In ronde 2 wordt de tag afgeleid van de Git-commit-SHA.

## 4. Test de automatische deployment

1. Breng een kleine, herkenbare wijziging aan die **zonder database** zichtbaar of controleerbaar is, zonder bestaande tests onnodig te breken.
2. Commit en push naar de vertrouwde deploymentbranch.
3. Controleer in **GitHub → Actions** of de bestaande CI-controles slagen **vóór** de deploymentjob.
4. Controleer in ACR of de image `v2` is gepubliceerd.
5. Controleer in **Azure Container Apps → Revisions and replicas** welke image de nieuwe revisie gebruikt.
6. Open de HTTPS-URL en controleer `/health/live` en de zichtbare wijziging.

> Het endpoint `/version` kan in deze ronde nog `commit: "unknown"` tonen. De commit-SHA wordt pas in ronde 2 expliciet gekoppeld aan de image en runtimeconfiguratie.

## Als het niet werkt

| Symptoom | Controle |
| --- | --- |
| Deployjob wordt overgeslagen | `needs:`, jobnamen, branchvoorwaarde en voorafgaande CI-resultaten |
| Azure-login mislukt | Federated credential: exacte repository, branch, tenant en client ID |
| `docker push` is geweigerd | `AcrPush` op de juiste registry; kan roltoewijzing nog niet zijn doorgevoerd? |
| `az containerapp update` wordt geweigerd | Rechten op de Container App/resource group, naam en subscription |
| Revisie start niet | Juiste image-tag, ACR-pullrechten en de containerpoort/ingress |
| Workflow groen, oude versie zichtbaar | Geconfigureerde image/revisie en daadwerkelijke applicatieresponse |

**Resultaat:** een geslaagde codewijziging leidt automatisch tot het publiceren van image `v2` en het bijwerken van Azure Container Apps.

**Documentatie:** [Azure Login](https://github.com/Azure/login), [ACR-authenticatie](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-authentication), [GitHub Actions met Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/github-actions), [Container Apps CLI](https://learn.microsoft.com/en-us/cli/azure/containerapp).
