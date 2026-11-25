# Les 5 – Cloud Deployment met Microsoft Azure

## Ronde 1 – Simpele deployment

In de vorige lessen heb je de dotNOVI-applicatie gecontaineriseerd en een CI-pipeline gemaakt met GitHub Actions. Nu ga je de applicatie deployen naar Microsoft Azure en ervoor zorgen dat nieuwe, succesvol gecontroleerde codewijzigingen automatisch worden uitgerold.

Je voert deze oefening in twee stappen uit:
1. **Praktijk A:** bouw en publiceer een Docker-image via GitHub Actions naar Azure Container Registry (ACR) en deploy deze daarna handmatig via de Azure Portal naar Azure Container Apps.
2. **Praktijk B:** breid je bestaande CI-pipeline uit zodat nieuwe versies automatisch worden gebouwd, gepubliceerd en gedeployed.

Deze eerste ronde draait om het werkend krijgen van de volledige keten. Je mag de hulpdocumenten en ondersteuning van de docent gebruiken. In de tweede ronde (huiswerk) bouw je de infrastructuur opnieuw op, met strengere eisen en minder begeleiding.

### Image bouwen via via GitHub Actions
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
Bouw op een GitHub-hosted runner een image van dotNOVI met je bestaande Dockerfile en publiceer deze naar je eigen ACR. Gebruik voor deze eerste versie de tag `v1`. Je hoeft hiervoor geen Docker of Azure CLI op je eigen computer uit te voeren. In plaats daarvan maak hiervoor een **aparte** GitHub Actions-workflow die je handmatig kunt starten.

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

De image staat nu in ACR, maar draait nog niet in Azure. De deployment configureer je **handmatig in de Azure Portal**.
* Maak een **Container Apps Environment** aan en deploy je gepubliceerde image met een nieuwe **Azure Container App**.
* Gebruik de volgende uitgangspunten:

| Instelling | Waarde |
| --- | --- |
| Resource Group | `rg-dotnovi-lab` |
| Container Apps Environment | `cae-dotnovi-lab` |
| Container App | `ca-dotnovi-lab` |
| Image | Jouw ACR-image met tag `v1` |
| Hosting | Consumption |
| Resources | Kleine CPU- en geheugenconfiguratie |
| Minimum replicas | 0 |
| Maximum replicas | 1 |
| Ingress | **External**, HTTP |
| Target port | De poort waarop Express in de container luistert |

Voor een **private ACR-image** heeft Container Apps leesrechten nodig. Gebruik hiervoor de aanvullende instructie:

**[Technische hulp A – Een private ACR-image gebruiken in Azure Container Apps](./LES5_HULP_ACR_PULL.md)**

Hierin configureer je een afzonderlijke managed identity met de benodigde pullrechten. Dat is een andere identiteit dan de identity waarmee GitHub Actions in Praktijk A de image naar ACR publiceert.

**Onderzoeksvraag:** waar in de dotNOVI-configuratie zie je op welke poort de applicatie luistert, en waar configureer je deze als target port in Azure?

> **Let op:** configureer `/health/ready` in deze ronde niet als verplichte HTTP-readinessprobe. PostgreSQL is nog niet beschikbaar.

### A.5 – Controleer de deployment

Open de publieke HTTPS-URL van je Container App en controleer:

1. `/version` is bereikbaar en retourneert HTTP 200.
2. `/health/live` retourneert HTTP 200.
3. In Azure is zichtbaar dat je Container App de image met tag `v1` gebruikt.

Het is normaal dat `/health/ready` HTTP 503 retourneert. Ook de databaseafhankelijke routes werken nog niet volledig.

**Tussenresultaat A:** dotNOVI draait als HTTP-service op Azure Container Apps en is via HTTPS bereikbaar. Je hebt ervaren dat **een image publiceren naar ACR niet hetzelfde is als een applicatie deployen naar Container Apps**.

---

## Praktijk B – Automatiseer je deployment (25 minuten)

In Praktijk A heb je GitHub Actions **handmatig gestart** om image `v1` te publiceren en daarna de image **via de Portal gedeployed**. Nu breng je imagepublicatie en deployment samen in je bestaande CI-pipeline.

### B.1 – Integreer imagepublicatie in je CI-pipeline

Gebruik de relevante stappen uit de handmatig gestarte workflow van Praktijk A om je **bestaande CI-workflow** uit te breiden.

Zorg ervoor dat:

1. De bestaande tests en kwaliteitscontroles blijven draaien.
2. Een image pas wordt gebouwd en gepubliceerd **nadat alle verplichte CI-controles zijn geslaagd**.
3. De image naar je eigen ACR wordt gepubliceerd met tag **`v2`**.

Je hoeft de CI-pipeline niet opnieuw te bouwen. Gebruik `needs:` om afhankelijkheden tussen jobs te configureren. Let op: de `AcrPush`-rechten heb je **al in Praktijk A** ingesteld.

### B.2 – Automatiseer de deployment naar Azure Container Apps

Breid de workflow uit zodat na succesvolle imagepublicatie ook de **bestaande Container App** wordt bijgewerkt.

De deployment moet:

1. Via OIDC authenticeren bij Azure.
2. De Container App `ca-dotnovi-lab` bijwerken.
3. De gepubliceerde image met tag `v2` gebruiken.
4. Alleen plaatsvinden nadat de verplichte CI-controles en imagepublicatie zijn geslaagd.
5. Alleen draaien vanaf de branch waarvoor je de OIDC-koppeling hebt ingericht.

Gebruik hiervoor **[Technische hulp B – GitHub Actions uitbreiden met Azure-deployment](./LES5_HULP_GITHUB_ACTIONS.md)**.

**Hints:** `needs:`, `azure/login@v3` en `az containerapp update --image`. De commando's draaien op de GitHub-hosted runner, niet op je laptop.

### B.3 – Test de automatische deployment

Breng een kleine, herkenbare wijziging aan die ook **zonder PostgreSQL** zichtbaar is en je bestaande tests niet onnodig breekt. Commit en push naar de afgesproken deploymentbranch.

Controleer:

1. Start de CI/CD-workflow automatisch?
2. Slagen de CI-controles **voordat** de image wordt gepubliceerd?
3. Staat image `v2` in je ACR?
4. Is de Container App automatisch bijgewerkt naar `v2`?
5. Draait de gewijzigde applicatie en retourneert `/health/live` HTTP 200?

In ronde 1 mag je `v2` gebruiken. Commit-SHA-tags en de koppeling met `APP_COMMIT_SHA` volgen in ronde 2.

**Tussenresultaat B:** een succesvolle codewijziging leidt automatisch tot het bouwen, publiceren en deployen van een nieuwe applicatieversie.

---

## Definition of Done – Ronde 1

- [ ] DotNOVI-image `v1` is via een **handmatig gestarte GitHub Actions-workflow** gepubliceerd naar ACR.
- [ ] Ik heb dotNOVI **handmatig via de Azure Portal** gedeployed.
- [ ] DotNOVI draait in Azure Container Apps en is bereikbaar via HTTPS.
- [ ] `/version` en `/health/live` reageren met HTTP 200.
- [ ] Container Apps kan de private image via een managed identity uit ACR ophalen.
- [ ] De bestaande CI-controles moeten slagen **voordat** de nieuwe image wordt gepubliceerd en gedeployed.
- [ ] GitHub Actions meldt zich met OIDC aan bij Azure.
- [ ] Een codewijziging start automatisch de workflow die image `v2` bouwt, publiceert en deployt.
- [ ] Je kunt in GitHub Actions en Azure aanwijzen welke nieuwe image wordt gebruikt.

In deze ronde hoef je nog geen uitgebreide architectuurdocumentatie of zelfstandig foutonderzoek op te leveren.

## Niet klaar binnen de lestijd?

De 65 minuten zijn bedoeld om de keten samen met de docent voor het eerst werkend te krijgen. Als iets niet lukt, noteer **welke stap faalt** en **welke foutmelding je ziet**. Rond de werkende keten zo nodig na de les af voordat je aan ronde 2 begint.

## Na de les – Ronde 2: Zelfstandig opnieuw opbouwen

**[Ga naar de volledige huiswerkopdracht: Ronde 2 – Bouw het zelfstandig en verantwoord](./LES5_RONDE2.md).**

In ronde 2 verwijder je de tijdelijke **Container App, Container Apps Environment en ACR**, en bouw je deze infrastructuur zelfstandig opnieuw op. Je mag de bestaande Resource Group, GitHub-identity, broncode en CI-workflow behouden.

Dan gelden aanvullende eisen:

- Herleidbare images met commit-SHA-tags in plaats van `v1`/`v2`.
- Correcte koppeling tussen gepubliceerde image, `APP_COMMIT_SHA` en `/version`.
- Beperkte toegangsrechten en veilige runtimeconfiguratie.
- Aantonen dat falende kwaliteitscontroles deployment blokkeren.
- Deploymentstatus, logs en een foutscenario onderzoeken.
- De gekozen architectuur en configuratie reproduceerbaar documenteren.

**Verwijder je oefenresources pas wanneer de werkende deployment uit ronde 1 is gecontroleerd.** Controleer ook aanvullende Azure-resources die kosten kunnen veroorzaken, zoals loggingresources.

## Officiële documentatie

- [Azure Container Registry](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-intro)
- [Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/overview)
- [GitHub Actions – Manually running a workflow](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/manually-run-a-workflow)
- [GitHub Actions met Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/github-actions)
- [Managed identity voor image pulls](https://learn.microsoft.com/en-us/azure/container-apps/managed-identity-image-pull)
