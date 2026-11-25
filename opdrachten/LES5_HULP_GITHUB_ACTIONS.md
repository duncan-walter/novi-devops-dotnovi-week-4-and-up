# Technische hulp B – GitHub Actions uitbreiden met Azure-deployment

> **Bij Praktijk B, opdracht B.2.** Deze instructie hoort bij [LES5.md](./LES5.md). Je hebt de OIDC-voorbereidingsopdracht na les 4 voltooid en een werkende dotNOVI-deployment uit Praktijk A.

## Wat ga je doen?

In Praktijk A heb je al een **handmatig gestarte GitHub Actions-workflow** gemaakt die image `v1` naar Azure Container Registry (ACR) publiceert. Breid nu je **bestaande CI-workflow** uit en **hergebruik de werkende publicatiestappen**. Na succesvolle tests en linting publiceert GitHub Actions image `v2` en werkt de bestaande Container App bij.

Je gebruikt de eerder aangemaakte identity `id-github-dotnovi`, niet de identity `id-acr-pull-dotnovi`. Deze laatste is alleen bedoeld om vanuit Container Apps images op te halen.

## 1. Hergebruik je publicatieworkflow uit Praktijk A

Je hebt `id-github-dotnovi` in Praktijk **A.2** al `AcrPush` gegeven op jouw ACR. **Ken deze rechten niet opnieuw toe.**

Neem de werkende authenticatie-, build- en publishstappen uit de handmatig gestarte workflow van **A.3** over in je bestaande CI-workflow. Publiceer in deze ronde tag **`v2`** in plaats van `v1`.

Zorg dat de publicatiejob afhankelijk is van **alle verplichte test- en lintcontroles**. De aparte handmatig te starten workflow kan blijven bestaan, maar gebruik voor de automatische deployment je bestaande CI-workflow.

De OIDC-identiteit moet voldoende rechten hebben om ook de bestaande Container App bij te werken. Die rechten heb je na les 4 voorbereid.

## 2. Vul de deploymentjob in

Onderstaande YAML is een **gedeeltelijk voorbeeld** van een job die publiceren en deployen combineert. Voeg de relevante stappen toe onder `jobs:` in je bestaande CI-workflow of gebruik afzonderlijke afhankelijke publish- en deployjobs. Maak niet opnieuw een CI-pipeline vanaf nul.

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

      # TODO 1: Hergebruik de ACR-login uit Praktijk A.3.
      # TODO 2: Hergebruik de Docker-build uit Praktijk A.3 (nu met tag v2).
      # TODO 3: Hergebruik de ACR-push uit Praktijk A.3 (nu met tag v2).
      # TODO 4: Werk de bestaande Container App bij naar de gepubliceerde image v2.
```

- Pas `needs` aan je bestaande workflow aan. Staan test en lint in dezelfde job, gebruik dan de naam van die ene job. Publicatie mag niet beginnen vóórdat deze controles slagen.
- De branch in de `if`-voorwaarde moet overeenkomen met de branch in je **federated credential**. Gebruik een andere naam als je geen `main` hebt.
- De GitHub Actions-secrets `AZURE_CLIENT_ID`, `AZURE_TENANT_ID` en `AZURE_SUBSCRIPTION_ID` heb je al tijdens de voorbereiding ingesteld.
- Plaats **geen wachtwoorden, tokens of registry admin-credentials** in de YAML.

## 3. Maak publiceren én deployen automatisch

| Doel | Aanwijzing |
| --- | --- |
| Docker aanmelden bij ACR | Hergebruik de werkende `az acr login`-stap uit Praktijk A.3. |
| Image bouwen | Hergebruik je `docker build`-stap en geef de image tag `v2`. |
| Image publiceren | Hergebruik `docker push` uit Praktijk A.3. |
| Container App bijwerken | Zoek `az containerapp update --image`; geef de app-naam, Resource Group en de volledige image-URL mee. |

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
