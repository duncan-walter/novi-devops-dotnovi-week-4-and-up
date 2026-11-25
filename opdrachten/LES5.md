# Les 5 – Cloud Deployment met Microsoft Azure

## Ronde 1 – Simpele deployment

In de vorige lessen heb je de dotNOVI-applicatie gecontaineriseerd en een CI-pipeline gemaakt met GitHub Actions. Nu ga je de applicatie deployen naar Microsoft Azure en ervoor zorgen dat nieuwe, succesvol gecontroleerde codewijzigingen automatisch worden uitgerold.

Je voert deze oefening in twee stappen uit:
1. **Praktijk A:** bouw en publiceer een Docker-image via GitHub Actions naar Azure Container Registry (ACR) en deploy deze daarna handmatig via de Azure Portal naar Azure Container Apps.
2. **Praktijk B:** breid je bestaande CI-pipeline uit zodat nieuwe versies automatisch worden gebouwd, gepubliceerd en gedeployed.

Deze eerste ronde draait om het werkend krijgen van de volledige keten. Je mag de hulpdocumenten en ondersteuning van de docent gebruiken. In de tweede ronde (huiswerk) bouw je de infrastructuur opnieuw op, met strengere eisen en minder begeleiding.

### Image bouwen via GitHub Actions
In een professionele CI/CD-omgeving worden Docker-images doorgaans automatisch gebouwd en gepubliceerd vanuit een pipeline. Daarmee is het buildproces reproduceerbaar en niet afhankelijk van de ontwikkelcomputer van één persoon.

Je kunt een image ook lokaal bouwen en handmatig publiceren met Docker en de Azure CLI. Dat is bijvoorbeeld bruikbaar voor ontwikkeling of troubleshooting, maar slaan we in deze opdracht bewust over. GitHub Actions bouwt en publiceert je image op een **GitHub-hosted runner**. Je hoeft daarvoor geen Azure CLI of containerengine lokaal te installeren. De runner voert de benodigde commando's wel uit.

## Wat heb je al voorbereid?
Als voorbereiding heb je gezorgd voor:
- Een actief Azure for Students-account en een werkende dotNOVI-repository.
- Een Dockerfile en werkende GitHub Actions CI-pipeline voor dotNOVI.
- Een Azure Resource Group `rg-dotnovi-lab`.
- Een user-assigned managed identity `id-github-dotnovi`, met een federated credential voor jouw GitHub-repository en deploymentbranch.
- De benodigde Azure-rol voor het bijwerken van Container Apps in je Resource Group.
- De GitHub Actions-secrets `AZURE_CLIENT_ID`, `AZURE_TENANT_ID` en `AZURE_SUBSCRIPTION_ID`.
- Een geslaagde OIDC-login-test vanuit GitHub Actions.

Werkt je OIDC-authenticatie niet? Vraag de docent om hulp. Gebruik geen alternatieve inlogmethode met opgeslagen wachtwoorden of client secrets om de OIDC-configuratie te omzeilen.

> **Let op:** in deze les richt je nog geen PostgreSQL-database in Azure in. dotNOVI kan als HTTP-service draaien zonder werkende databaseverbinding. Daarom is `/health/live` het controlepunt voor deze ronde; `/health/ready` kan terecht `HTTP 503` retourneren.

## Praktijk A
### A.1 – Maak een Azure Container Registry aan

Een registry slaat containerimages op zodat Azure Container Apps ze kan ophalen. Maak via de Azure Portal in je bestaande Resource Group een **Azure Container Registry (ACR)** aan:

| Instelling | Waarde |
| --- | --- |
| Resource Group | `rg-dotnovi-lab` |
| Registry name | Een wereldwijd unieke naam |
| Region | West Europe, indien beschikbaar |
| SKU | Basic |
| Role assignment permissions mode | RBAC Registry Permissions (zonder ABAC) |
| Admin user | Uitgeschakeld |

Noteer de **Login server** van je registry zoals die in Azure staat weergegeven.

### A.2 – Geef GitHub Actions toegang tot ACR
Je hebt vooraf een managed identity `id-github-dotnovi` gemaakt waarmee GitHub Actions via OIDC kan aanmelden bij Azure. Aanmelden betekent niet automatisch dat deze identiteit images naar je nieuwe registry mag publiceren.

Geef de identity daarom _AcrPush_-rechten op **uitsluitend je ACR-resource**:

1. Open je Azure Container Registry in de Azure Portal.
2. Ga naar **Access control (IAM) → Add → Add role assignment**.
3. Selecteer **AcrPush**.
4. Kies bij Members **Managed identity** en selecteer `id-github-dotnovi`.
5. Bevestig de roltoewijzing.

**Documentatie:** [Microsoft Learn – Azure Container Registry roles and permissions](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-rbac-built-in-roles-overview).

### A.3 – Bouw en publiceer je Docker-image via GitHub Actions
Bouw op een GitHub-hosted runner een image van dotNOVI met je bestaande Dockerfile en publiceer deze naar je eigen ACR. Gebruik voor deze eerste versie de tag `v1`. Je hoeft hiervoor geen Docker of Azure CLI op je eigen computer uit te voeren. Maak daarom een **aparte** GitHub Actions-workflow die je handmatig kunt starten.

#### Stap 1 – Maak een handmatig startbare workflow

Maak in `.github/workflows/` een nieuw workflowbestand aan. Zoek in de [GitHub-documentatie over het handmatig starten van workflows](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/manually-run-a-workflow) hoe je met `workflow_dispatch` een workflow handmatig uitvoerbaar maakt.

#### Stap 2 – Haal de broncode op

Configureer een job die draait op een GitHub-hosted Ubuntu-runner. Bekijk in de [documentatie van `actions/checkout`](https://github.com/actions/checkout#usage) hoe je de broncode van dotNOVI beschikbaar maakt op de runner, zodat de Dockerfile tijdens de build kan worden gebruikt.

#### Stap 3 – Authenticeer bij Azure en ACR
Tijdens de voorbereidingsopdracht heb je al een werkende OIDC-koppeling tussen GitHub Actions en Azure ingericht. Je kunt deze nu opnieuw gebruiken. Voeg de volgende permissies toe aan je job:

```yaml
permissions:
  id-token: write
  contents: read
```

Voeg vervolgens de Azure-loginstap toe aan je workflow:

```yaml
- name: Login to Azure
  uses: azure/login@v3
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

Zoals je weet, authenticeert GitHub Actions zich hiermee via de bestaande OIDC-koppeling bij Microsoft Azure. Je hoeft hiervoor dus geen nieuwe credentials aan te maken.

Vervolgens moet ook Docker toegang krijgen tot jouw Azure Container Registry. Zoek in de [Microsoft-documentatie over ACR-authenticatie](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-get-started-docker-cli#log-in-to-a-registry) hoe je dit met de Azure CLI doet. Let op: de Azure CLI en Docker worden hierbij op de GitHub-hosted runner uitgevoerd, **niet** op je eigen computer.

#### Stap 4 – Bouw de Docker-image
Voeg een stap toe waarmee je op de GitHub-runner een Docker-image bouwt vanuit de bestaande Dockerfile van dotNOVI.

Zoek in de [Docker-documentatie](https://docs.docker.com/reference/cli/docker/image/build/) over het bouwen van images hoe je een image bouwt en direct een naam en versie (_tag_) meegeeft.

Om de image later naar Azure Container Registry te kunnen publiceren, moet de image de juiste naam krijgen. Deze bestaat uit drie onderdelen:

```yaml
<registryadres>/<imagenaam>:<tag>
```

Bijvoorbeeld: `acrdotnovimaarten.azurecr.io/dotnovi:v1`

Hierbij is:
* `acrdotnovimaarten.azurecr.io` het adres van jouw Azure Container Registry (Login server).
* `dotnovi` de naam van de image (repository).
* `v1` de versie van de image (_tag_).

Je vindt de **Login server** op de overzichtspagina (Overview) van je Azure Container Registry in de Azure Portal. Gebruik voor deze opdracht `dotnovi` als repositorynaam en `v1` als tag.

#### Stap 5 – Publiceer de image naar ACR

Je hebt nu een Docker-image gebouwd op de GitHub-runner. Deze image is nog niet beschikbaar in Azure Container Registry. Voeg een stap toe aan je GitHub Actions-workflow waarmee je de image naar jouw ACR publiceert.

Zoek in de [Docker-documentatie](https://docs.docker.com/reference/cli/docker/image/push/) over het publiceren van images hoe je met `docker push` een image naar een container registry verstuurt.

Gebruik hierbij dezelfde **volledige imageverwijzing als bij het bouwen van de image in stap 4** (bijvoorbeeld `acrdotnovimaarten.azurecr.io/dotnovi:v1`). Wanneer de workflow succesvol is uitgevoerd, staat de image in jouw Azure Container Registry. Je kunt deze vervolgens gebruiken om een Azure Container App handmatig te deployen.

#### Controleer je resultaat

1. Commit en push je workflowbestand naar de branch waarvoor je OIDC-authenticatie hebt ingericht.
2. Open de Actions-tab van je GitHub-repository en start de workflow via Run workflow.
3. Controleer of alle stappen succesvol zijn afgerond.
4. Open je Azure Container Registry in de Azure Portal. Ga naar Services → Repositories en controleer of de repository `dotnovi` aanwezig is.
5. Open de repository en controleer of de tag `v1` beschikbaar is.

> **Let op:** om een workflow handmatig vanuit GitHub Actions te kunnen starten, moet het workflowbestand op de standaardbranch van de repository aanwezig zijn. Selecteer vervolgens de branch die overeenkomt met je voorbereide OIDC-koppeling.

### A.4 – Deploy naar Azure Container Apps
Je Docker-image staat nu in Azure Container Registry, maar de applicatie draait nog niet in Azure. Je gaat deze image handmatig deployen via de Azure Portal.

1. Maak een **Container Apps Environment** en een **Azure Container App** aan met de volgende instellingen. Begin bij het aanmaken zo nodig met de **publieke quickstart-image** van Azure. Je vervangt deze daarna door je private ACR-image, zodra je Container App en de bijbehorende system-assigned identity bestaan:

| Instelling | Waarde |
| --- | --- |
| Resource Group | `rg-dotnovi-lab` |
| Container Apps Environment | `cae-dotnovi-lab` |
| Container App | `ca-dotnovi-lab` |
| Image | Jouw ACR-image `dotnovi:v1` |
| Hosting | Consumption |
| Resources | Kleine CPU- en geheugenconfiguratie |
| Minimum replicas | 0 |
| Maximum replicas | 1 |
| Ingress | External, HTTP |
| Target port | De poort waarop Express in de container luistert |

2. Configureer de system-assigned managed identity van je Container App en vervang de eventuele quickstart-image via **Containers → Edit and deploy** door je eigen ACR-image `dotnovi:v1`. Kies **Managed identity → System assigned** als registry-authenticatie. De Azure Portal probeert de benodigde leesrechten (`AcrPull`) automatisch toe te kennen. Lukt het niet om de private image te selecteren of de identity te configureren? Gebruik dan [hulpdocument A](./LES5_HULP_ACR_PULL.md).

3. Controleer of de nieuwe revisie de image kan ophalen en succesvol start. Waar in de dotNOVI-configuratie zie je op welke poort de applicatie luistert, en waar configureer je deze als target port in Azure?

> **Let op:** configureer `/health/ready` in deze ronde niet als verplichte HTTP-readinessprobe. PostgreSQL is nog niet beschikbaar.

**Heb je problemen met `AcrPull`-rechten?** Controleer bij je Azure Container Registry onder **Access control (IAM) → Role assignments** of de managed identity van je Container App de rol **AcrPull** heeft. Ontbreekt deze rol, dan kun je die daar zelf toekennen, mits je voldoende rechten hebt. Lukt dat niet? Vraag de docent om hulp.

### A.5 – Controleer de deployment

Open de publieke HTTPS-URL van je Container App en controleer:

1. `/version` is bereikbaar en retourneert HTTP 200.
2. `/health/live` retourneert HTTP 200.
3. In Azure is zichtbaar dat je Container App de image met tag `v1` gebruikt.

Het is normaal dat `/health/ready` HTTP 503 retourneert. Ook de databaseafhankelijke routes werken nog niet volledig.

**Tussenresultaat A:** dotNOVI draait als HTTP-service op Azure Container Apps en is via HTTPS bereikbaar. Je hebt ervaren dat een image publiceren naar ACR niet hetzelfde is als een applicatie deployen naar Container Apps.

---

## Praktijk B – Automatiseer je deployment (25 minuten)

Je hebt zojuist een GitHub Actions-workflow gemaakt die je handmatig kunt starten. Hiermee heb je een Docker-image gepubliceerd naar ACR. Vervolgens heb je de applicatie handmatig gedeployed via de Azure Portal. Nu ga je de build-, publicatie- en deploymentstappen _automatiseren_.

Je hergebruikt je stappen uit de workflow van Praktijk A, maar integreert deze in jouw bestaande CI-workflow die je in de afgelopen paar lessen hebt gemaakt. Beide workflows gebruiken dezelfde branch waarvoor je OIDC-koppeling is ingericht. Het verschil is hoe ze worden gestart:

- De workflow uit Praktijk A start je handmatig met `workflow_dispatch`.
- Je uitgebreide CI-workflow uit Praktijk B start automatisch bij een `push`.

De gewenste volgorde in Praktijk B is: **Code pushen → CI-controles → Image bouwen en publiceren → Deployen naar Azure**

### B.1 – Breid je CI-pipeline uit met imagepublicatie
In de vorige les heb je een CI-workflow ingericht die automatisch start wanneer je code pusht. Je gaat deze bestaande workflow nu uitbreiden.

Loop je vast bij het inrichten van je publish-job? In [hulpdocument B](./LES5_HULP_GITHUB_ACTIONS.md) vind je een gedeeltelijk YAML-voorbeeld, uitleg over de afhankelijkheden tussen jobs en hulp bij veelvoorkomende foutmeldingen. Gebruik dit document ook bij B.2 als het automatisch deployen niet lukt.

1. Voeg aan je bestaande CI-workflow in `.github/workflows/` een nieuwe job met de naam `publish` toe.
2. Neem in deze job de stappen over uit je handmatige workflow van Praktijk A.
3. Zorg ervoor dat de `publish`-job alleen wordt uitgevoerd wanneer alle verplichte tests en kwaliteitscontroles succesvol zijn afgerond. Bekijk in de [GitHub-documentatie over jobs](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-jobs) hoe je met `needs:` deze afhankelijkheid instelt.
4. Publiceer de nieuwe image naar dezelfde Azure Container Registry, met repositorynaam `dotnovi` en tag **`v2`**.

> **Let op:** je gebruikt dezelfde OIDC-koppeling en `AcrPush`-rechten als in Praktijk A. Je hoeft deze niet opnieuw in Azure in te stellen. De nieuwe `publish`-job heeft **wel** opnieuw `actions/checkout`, de OIDC-permissies en de Azure-loginstap nodig, omdat elke job op een eigen runner wordt uitgevoerd.

### B.2 – Voeg automatische deployment toe
Je `publish`-job bouwt en publiceert nu de Docker-image `dotnovi:v2`. Je gaat deze job uitbreiden zodat de nieuwe image ook automatisch wordt gedeployed.

1. Voeg aan **dezelfde** `publish`-job, direct na de stap waarin je de image naar ACR pusht, een nieuwe deploymentstap toe.
2. Zoek in de [Microsoft-documentatie over Azure Container Apps](https://learn.microsoft.com/en-us/cli/azure/containerapp#az-containerapp-update) hoe je met `az containerapp update --image` de image van een bestaande Container App bijwerkt.
3. Configureer deze stap zodat de Container App `ca-dotnovi-lab` in Resource Group `rg-dotnovi-lab` wordt bijgewerkt naar jouw ACR-image `dotnovi:v2`.

> **Let op:** je gebruikt de **bestaande Azure-login** uit de `publish`-job. Je hoeft dus niet opnieuw te authenticeren. Je maakt ook geen nieuwe Container App aan: de applicatie die je in Praktijk A handmatig hebt gedeployed, wordt nu automatisch bijgewerkt.

### B.3 – Test de automatische deployment
Je CI/CD-pipeline is nu uitgebreid. Controleer of de volledige keten werkt:
1. Breng een kleine, herkenbare wijziging aan in dotNOVI, die zichtbaar is zonder de koppeling met de PostgreSQL-database.
2. Commit en push deze wijziging naar de branch die je tijdens de voorbereiding hebt gebruikt.
3. Open de **Actions**-tab in GitHub en volg de uitvoering van je CI/CD-workflow.

Controleer:
- [ ] De bestaande CI-controles zijn succesvol afgerond voordat de `publish`-job startte.
- [ ] De `publish`-job heeft de image `dotnovi:v2` naar ACR gepubliceerd.
- [ ] De deploymentstap is succesvol afgerond.
- [ ] Je bestaande Container App gebruikt de image `dotnovi:v2`.
- [ ] Je gewijzigde applicatie is via HTTPS bereikbaar en `/health/live` retourneert HTTP 200.

**Resultaat:** een succesvolle push leidt automatisch tot het controleren van je code, het bouwen en publiceren van een nieuwe Docker-image en het bijwerken van je bestaande Azure Container App.
