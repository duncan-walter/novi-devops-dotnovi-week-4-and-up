# Les 5 – Cloud Deployment met Microsoft Azure

## Ronde 1 – Maak het werkend

**Tijdens de les: 65 minuten praktijkwerk**  
**Praktijk A:** 40 minuten · **Praktijk B:** 25 minuten

In de vorige lessen heb je de **dotNOVI-applicatie** gecontaineriseerd en een CI-pipeline gemaakt met GitHub Actions. Nu ga je de applicatie deployen naar **Microsoft Azure** en ervoor zorgen dat nieuwe, succesvol gecontroleerde codewijzigingen automatisch worden uitgerold.

Je voert deze oefening in twee stappen uit:

- **Praktijk A:** publiceer een Docker-image naar Azure Container Registry (ACR) en deploy deze handmatig naar Azure Container Apps.
- **Praktijk B:** breid je bestaande GitHub Actions-workflow uit zodat nieuwe versies automatisch worden gepubliceerd en gedeployed.

Deze eerste ronde draait om het **werkend krijgen van de volledige keten**. Je mag de hulpdocumenten en ondersteuning van de docent gebruiken. In **ronde 2 (huiswerk)** bouw je de infrastructuur opnieuw op, met strengere eisen en minder begeleiding.

## Wat heb je al voorbereid?

Na les 4 heb je als voorbereiding:

- Een actief **Azure for Students-account** en een werkende dotNOVI-repository.
- Een Dockerfile en werkende GitHub Actions **CI-pipeline** voor dotNOVI.
- Een Azure Resource Group `rg-dotnovi-lab`.
- Een user-assigned managed identity `id-github-dotnovi`, met een federated credential voor jouw GitHub-repository en deploymentbranch.
- De benodigde Azure-rol voor het bijwerken van Container Apps in je Resource Group.
- De GitHub Actions-secrets `AZURE_CLIENT_ID`, `AZURE_TENANT_ID` en `AZURE_SUBSCRIPTION_ID`.
- Een **geslaagde OIDC-login-test** vanuit GitHub Actions.

Als je voorbereiding niet werkt, meld dit aan de docent voordat je begint. Gebruik geen langdurige Azure-wachtwoorden als snelle vervanging.

> **Database:** in deze les richt je nog geen PostgreSQL-database in Azure in. dotNOVI kan als HTTP-service draaien zonder werkende databaseverbinding. Daarom is `/health/live` het controlepunt voor deze ronde; `/health/ready` kan terecht **HTTP 503** retourneren.

---

## Praktijk A – Eerste clouddeployment (40 minuten)

### A1 – Maak een Azure Container Registry aan

Een registry slaat containerimages op zodat Azure Container Apps ze kan ophalen.

Maak via de Azure Portal in je bestaande Resource Group een **Azure Container Registry (ACR)** aan:

| Instelling | Waarde |
| --- | --- |
| Resource Group | `rg-dotnovi-lab` |
| Registry name | Een **wereldwijd unieke** naam |
| Region | West Europe, indien beschikbaar |
| SKU | Basic |
| Role assignment permissions mode | **RBAC Registry Permissions** (zonder ABAC) |
| Admin user | Uitgeschakeld |

Noteer de **Login server** van je registry zoals die in Azure staat weergegeven.

### A2 – Publiceer je Docker-image

Gebruik je bestaande Dockerfile om een image van dotNOVI te bouwen en naar je eigen ACR te publiceren.

Voor de eerste deployment gebruik je de tag **`v1`**.

1. Bouw de image.
2. Meld Docker aan bij ACR via Azure CLI.
3. Geef de image de volledige naam die bij jouw registry hoort.
4. Publiceer de image.
5. Controleer in ACR of de repository en tag `v1` beschikbaar zijn.

**Hints:** `az acr login`, `docker build`, `docker tag`, `docker push`.

Een imageverwijzing heeft de vorm:

```text
<acr-login-server>/<repository>:v1
```

### A3 – Deploy naar Azure Container Apps

Maak een **Container Apps Environment** aan en deploy je gepubliceerde image met een nieuwe **Azure Container App**.

Gebruik de volgende uitgangspunten:

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

Hierin configureer je een afzonderlijke managed identity met de benodigde pullrechten. Dat is een andere identiteit dan de identity waarmee GitHub Actions straks naar Azure deployt.

**Onderzoeksvraag:** waar in de dotNOVI-configuratie zie je op welke poort de applicatie luistert, en waar configureer je deze als target port in Azure?

> **Let op:** configureer `/health/ready` in deze ronde niet als verplichte HTTP-readinessprobe. PostgreSQL is nog niet beschikbaar.

### A4 – Controleer de deployment

Open de publieke HTTPS-URL van je Container App en controleer:

1. `/version` is bereikbaar en retourneert HTTP 200.
2. `/health/live` retourneert HTTP 200.
3. In Azure is zichtbaar dat je Container App de image met tag `v1` gebruikt.

Het is normaal dat `/health/ready` HTTP 503 retourneert. Ook de databaseafhankelijke routes werken nog niet volledig.

**Tussenresultaat A:** dotNOVI draait als HTTP-service op Azure Container Apps en is via HTTPS bereikbaar.

---

## Praktijk B – Automatiseer je deployment (25 minuten)

Je hebt de image handmatig gepubliceerd en gedeployed. Nu ga je die handelingen opnemen in je bestaande GitHub Actions-workflow.

### B1 – Geef GitHub Actions pushrechten op ACR

Je hebt al een managed identity `id-github-dotnovi` waarmee GitHub Actions via OIDC kan aanmelden bij Azure. Die identity heeft nog niet automatisch rechten om images naar je nieuwe registry te **pushen**.

Geef `id-github-dotnovi` de rol **AcrPush** op **jouw ACR-resource** via **Access control (IAM)**. Gebruik geen onnodig brede rollen op je subscription.

De Container Apps-pullidentity uit Praktijk A blijft apart bestaan.

### B2 – Breid de CI-workflow uit

Voeg aan je **bestaande CI-workflow** een deploymentjob toe die:

1. Alleen start wanneer alle verplichte CI-controles succesvol zijn.
2. Via OIDC authenticeert bij Azure.
3. Een nieuwe Docker-image bouwt en onder tag **`v2`** publiceert naar ACR.
4. Je bestaande Container App bijwerkt naar de image met tag **`v2`**.

Je hoeft de CI-pipeline niet opnieuw te bouwen. Gebruik de YAML-fragmenten uit de les en de volgende handout:

**[Technische hulp B – GitHub Actions uitbreiden met Azure-deployment](./LES5_HULP_GITHUB_ACTIONS.md)**

**Hints:** `needs:`, `azure/login@v3`, `az acr login`, `docker build`, `docker push` en `az containerapp update --image`.

Controleer dat de deploymentjob alleen op de juiste branch draait: dezelfde branch die je in de OIDC-voorbereiding hebt ingesteld.

### B3 – Test de automatische deployment

Voer een kleine, herkenbare wijziging door die zonder PostgreSQL te controleren is en die je bestaande tests niet onnodig breekt.

Commit en push naar de afgesproken deploymentbranch. Controleer vervolgens:

1. Start de workflow automatisch?
2. Slagen de CI-controles **voordat** de deploymentjob start?
3. Staat image `v2` in je ACR?
4. Is Azure Container Apps bijgewerkt naar die image?
5. Draait de gewijzigde applicatie en reageert `/health/live` met HTTP 200?

In deze eerste ronde mag je de vaste tag `v2` gebruiken. Het gebruik van commit-SHA-tags en de koppeling met `APP_COMMIT_SHA` komen in ronde 2 aan bod.

**Tussenresultaat B:** een geslaagde codewijziging leidt automatisch tot een nieuwe deployment naar Azure.

---

## Definition of Done – Ronde 1

- [ ] DotNOVI-image `v1` is gepubliceerd naar ACR.
- [ ] DotNOVI draait in Azure Container Apps en is bereikbaar via HTTPS.
- [ ] `/version` en `/health/live` reageren met HTTP 200.
- [ ] Container Apps kan de private image via een managed identity uit ACR ophalen.
- [ ] De bestaande CI-controles blokkeren de deploymentjob als ze mislukken.
- [ ] GitHub Actions meldt zich met OIDC aan bij Azure.
- [ ] Een nieuwe image `v2` wordt automatisch gepubliceerd en gedeployed.
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
- [GitHub Actions met Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/github-actions)
- [Managed identity voor image pulls](https://learn.microsoft.com/en-us/azure/container-apps/managed-identity-image-pull)
