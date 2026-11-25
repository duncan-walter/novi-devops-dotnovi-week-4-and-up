# Technische hulp B – GitHub Actions: publiceren en deployen

> **Bij Praktijk B, opdrachten B.1 en B.2** in [LES5.md](./LES5.md). Gebruik dit document als je hulp nodig hebt bij de structuur van de nieuwe `publish`-job of bij foutmeldingen. Het is geen tweede opdracht.

## Wat vind je hier?

In Praktijk A werkte een **handmatig gestarte workflow** voor image `v1`. In Praktijk B voeg je aan je **bestaande CI-workflow uit les 4** één job met de naam `publish` toe. Deze job gebruikt de werkende stappen uit Praktijk A, bouwt en pusht `dotnovi:v2` en werkt daarna **de bestaande Container App** bij.

De handmatige workflow uit Praktijk A mag blijven bestaan. Maak **geen derde workflow** en ook geen aparte `deploy`-job voor deze oefening.

## 1. Startpunt voor je `publish`-job

Onderstaand fragment plaats je onder `jobs:` van je **bestaande CI-workflow**. Het is bewust **onvolledig**: je vult de TODO-stappen in met de werkende commando's uit A.3 en de deploymentopdracht uit B.2.

```yaml
  publish:
    needs: [test, lint]   # Vervang door de echte namen van AL je verplichte CI-jobs
    if: github.ref == 'refs/heads/main'  # Pas aan als je OIDC-branch anders heet
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

      # TODO 1: Log Docker in bij ACR (zoals in A.3).
      # TODO 2: Bouw de volledige ACR-imageverwijzing met tag v2.
      # TODO 3: Publiceer dezelfde image naar ACR met docker push.
      # TODO 4: Werk ca-dotnovi-lab bij naar die image met az containerapp update --image.
```

### Waar moet je op letten?

- **`needs:` verwijst naar job-ID's, niet naar de getoonde namen van stappen.** Heb je één CI-job die alle tests en linting uitvoert? Verwijs dan alleen naar die job. Alle verplichte CI-jobs moeten succesvol zijn vóór `publish`.
- **Elke job draait op een eigen runner.** Daarom heeft `publish` opnieuw `actions/checkout`, OIDC-permissies en `azure/login@v3` nodig. Binnen `publish` hoef je na de ACR-push **niet nog een keer** bij Azure in te loggen om te deployen.
- **Branchbeveiliging:** de `if`-voorwaarde moet overeenkomen met de branch in je federated credential. De bestaande CI-workflow start al automatisch bij `push`.
- **Azure-rechten:** `id-github-dotnovi` heeft al `AcrPush` (A.2) en de rechten voor Container Apps uit je voorbereiding. Ken die niet opnieuw toe. De **system-assigned identity van de Container App** is uitsluitend voor het ophalen van de private image (`AcrPull`).
- **Eén imageverwijzing:** gebruik voor `docker build`, `docker push` en `az containerapp update --image` exact dezelfde volledige verwijzing: `<login-server>/dotnovi:v2`. Neem de echte login server over uit de Azure Portal.
- **Geen credentials in YAML:** bewaar geen registrywachtwoorden, admin-inloggegevens of client secrets in je workflow.

Gebruik bij het bijwerken van de bestaande Container App de [Microsoft-documentatie voor `az containerapp update`](https://learn.microsoft.com/en-us/cli/azure/containerapp#az-containerapp-update). De commando's worden op de **GitHub-hosted runner** uitgevoerd, niet op jouw laptop.

## 2. Als het niet werkt

Gebruik de controlepunten uit [B.3 in de hoofdopdracht](./LES5.md#b3--test-de-automatische-deployment) voor de eindcontrole. Deze tabel helpt als een stap faalt:

| Symptoom | Controle |
| --- | --- |
| `publish` start niet of wordt overgeslagen | `needs:`, echte CI-job-ID's, status van de tests en branchvoorwaarde |
| Azure-login mislukt | `id-token: write`, federated credential (repo en branch), client-, tenant- en subscription-ID |
| `docker push` wordt geweigerd | ACR-login en `AcrPush` voor `id-github-dotnovi` op de juiste registry |
| `az containerapp update` wordt geweigerd | `ca-dotnovi-lab`, `rg-dotnovi-lab`, subscription en Azure RBAC-rechten |
| Nieuwe revisie haalt de image niet op | System-assigned identity van de Container App, `AcrPull`, juiste login server en tag `v2` |
| Workflow slaagt maar oude versie blijft zichtbaar | Actieve revisie, verkeersverdeling, gebruikte image en zichtbare wijziging |

> **Let op:** `/version` kan in ronde 1 nog `commit: "unknown"` tonen. De koppeling aan de commit-SHA volgt in ronde 2.

**Resultaat:** je hebt één `publish`-job die alleen na succesvolle CI-controles de image publiceert én de bestaande Container App bijwerkt. De volledige functionele test voer je uit in B.3.

Voor meer achtergrond kun je de [Azure Login-documentatie](https://github.com/Azure/login) en de [Microsoft-documentatie over GitHub Actions met Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/github-actions) raadplegen.
