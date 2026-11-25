# Les 5 – Cloud Deployment met Microsoft Azure

## Ronde 2 – Bouw zelfstandig een herleidbare deploymentketen

**Uitvoering:** zelfstandig huiswerk na les 5  
**Voorkennis:** een werkende CI/CD-keten uit [ronde 1](./LES5.md)

### Wat leer je?

In ronde 1 heb je onder begeleiding een containerimage gepubliceerd, handmatig gedeployed en daarna automatische deployment ingericht. In deze ronde laat je zien dat je:

1. **zelfstandig** een Azure-omgeving opnieuw kunt opbouwen;
2. **veilige toegang** kunt inrichten en de gebruikte rechten kunt uitleggen;
3. elke deployment **kunt herleiden tot één Git-commit**;
4. kunt aantonen dat **falende CI-controles publicatie en deployment blokkeren**;
5. een deployment kunt **verifiëren en storingen systematisch onderzoeken**.

**Werkwijze:** bouw de **Azure-infrastructuur opnieuw op**, maar **behoud en verbeter je bestaande CI/CD-workflow**. Gebruik documentatie in plaats van het stappenplan uit ronde 1. Je hoeft geen Docker of Azure CLI op je laptop te installeren; de commando's draaien in GitHub Actions.

**Eindresultaat:** twee aantoonbaar succesvolle deployments met verschillende commit-SHA's, een test waarbij falende CI publicatie blokkeert, en beknopte technische documentatie.

---

## Opdracht 1 – Bouw de Azure-infrastructuur opnieuw op

### 1.1 Ruim de oefenomgeving uit ronde 1 op

Controleer eerst of je resultaat van ronde 1 werkt. Verwijder daarna **alleen** de volgende oefenresources:

- de Azure Container App `ca-dotnovi-lab`;
- de Container Apps Environment `cae-dotnovi-lab`;
- je Azure Container Registry.

**Behoud** de Resource Group `rg-dotnovi-lab`, de GitHub Actions-identity `id-github-dotnovi`, de OIDC-federatie, je broncode en je bestaande CI/CD-workflow. Controleer of er nog logging- of monitoringresources van de oefening bestaan.

De **system-assigned managed identity** verdwijnt met de verwijderde Container App. Maak bij de nieuwe Container App opnieuw een system-assigned identity aan. Roltoewijzingen op de verwijderde ACR moet je bij de nieuwe registry opnieuw instellen.

### 1.2 Richt de omgeving zelfstandig opnieuw in

Maak de resources aan die nodig zijn om dotNOVI als HTTP-service te draaien.

**Eisen:**

- Gebruik je eigen Azure for Students-subscription en herkenbare resourcenamen.
- Publiceer images naar een **private Azure Container Registry**; laat het registry-adminaccount uitgeschakeld.
- Host dotNOVI in **Azure Container Apps**, bereikbaar via HTTPS.
- Configureer ingress en de target port passend bij de applicatie.
- Kies onderbouwde, kostenbewuste instellingen voor CPU, geheugen en replicas.
- Richt **nog geen PostgreSQL-database** in. Die volgt in een latere les.

**Bewijs:** een werkende deployment en een compact architectuurdiagram met GitHub Actions, ACR, Container Apps en de identiteiten.

## Opdracht 2 – Richt toegang veilig in

Gebruik de bestaande OIDC-koppeling uit de voorbereiding. Zorg dat:

- `id-github-dotnovi` images naar **de nieuwe ACR** kan publiceren en de bestaande Container App kan bijwerken;
- de **system-assigned identity van de nieuwe Container App** de private image kan ophalen;
- iedere identiteit alleen de benodigde rollen heeft, op zo klein mogelijke scopes.

Let op: `AcrPush` en `AcrPull` gebruik je bij een registry met **RBAC Registry Permissions (zonder ABAC)**, zoals in ronde 1. Controleer het gekozen autorisatiemodel van je nieuwe ACR.

**Bewijs:** een korte tabel met per identiteit het doel, de Azure-rol en de scope. Leg in één zin per rol uit waarom die nodig is. Zet geen wachtwoorden of tokens in je repository.

## Opdracht 3 – Maak je bestaande CI/CD-pipeline herleidbaar

Je hebt al een werkende CI-workflow met een `publish`-job. **Maak geen nieuwe workflow.** Verbeter de bestaande keten:

**CI-controles → Image bouwen en publiceren → Bestaande Container App bijwerken → Resultaat verifiëren**

### 3.1 Behoud de kwaliteitsgrens

De `publish`-job mag pas starten nadat **alle verplichte test- en lintjobs** succesvol zijn afgerond. Behoud hiervoor de juiste afhankelijkheden met `needs:`. Publiceer en deploy alleen vanaf de afgesproken OIDC/deploymentbranch.

### 3.2 Gebruik een commit-SHA als image-tag

Vervang de vaste tag `v2` door de **volledige Git-commit-SHA van de betreffende workflowrun**. Gebruik dezelfde SHA bij het bouwen, publiceren en deployen van de image. Gebruik niet alleen `latest` of een andere veranderlijke tag.

De imageverwijzing heeft de vorm:

```text
<jouw-acr-login-server>/dotnovi/:<volledige-commit-sha>
```

### 3.3 Geef de SHA door aan de applicatie

Het endpoint `/version` leest de commit-SHA uit de environment variable `APP_COMMIT_SHA`.

Zorg dat de deployment **dezelfde volledige SHA** instelt als:

- tag van de gepubliceerde Docker-image;
- waarde van `APP_COMMIT_SHA` in de Container App.

**Een image-tag alleen stelt `APP_COMMIT_SHA` niet in.** Onderzoek hoe je in GitHub Actions de commit-SHA verkrijgt en hoe je bij `az containerapp update` zowel de image als de environment variable bijwerkt.

**Succescriterium:** de SHA van de workflowrun, de image-tag en de waarde die `/version` retourneert, zijn exact gelijk.

## Opdracht 4 – Bewijs twee succesvolle deployments

### 4.1 Controleer de eerste deployment

Controleer na een succesvolle push:

1. Alle verplichte CI-controles zijn geslaagd vóór `publish`.
2. ACR bevat `dotnovi` met de juiste commit-SHA-tag.
3. De actieve Container Apps-revisie gebruikt die image.
4. `/version` retourneert HTTP 200 en de juiste SHA.
5. `/health/live` retourneert HTTP 200 via HTTPS.

`/health/ready` kan **HTTP 503** teruggeven omdat PostgreSQL nog ontbreekt. Gebruik dit endpoint in deze ronde niet als verplichte readinessprobe. Leg in één zin uit waarom dat geen mislukte containerdeployment hoeft te betekenen.

### 4.2 Controleer een tweede deployment

Maak een kleine zichtbare wijziging die geen database nodig heeft. Commit en push die naar de deploymentbranch. Controleer opnieuw de hele keten.

**Bewijs:** leg beide deployments vast in je documentatie:

| Controle | Deployment 1 | Deployment 2 |
| --- | --- | --- |
| Commit-SHA | Volledige SHA | Andere volledige SHA |
| Image-tag in ACR | Dezelfde SHA | Dezelfde SHA |
| `/version` | Dezelfde SHA | Dezelfde SHA |
| GitHub Actions-run | Link | Link |

Voeg bewijs toe van de actieve image/revisie en de geslaagde `/health/live`-controle. Twee screenshots zonder koppeling met de commit zijn onvoldoende.

## Opdracht 5 – Onderzoek fouten en blokkeringen

### 5.1 Toon aan dat falende CI publicatie blokkeert

1. Introduceer tijdelijk een **veilige** fout waardoor een bestaande test of lintcontrole faalt.
2. Commit en push de fout naar de deploymentbranch.
3. Controleer dat de CI-controle faalt en de **`publish`-job wordt overgeslagen**.
4. Controleer dat voor deze falende SHA **geen image** in ACR is gepubliceerd en dat de Container App op de laatst geslaagde versie blijft.
5. Herstel de fout en zorg dat de pipeline weer slaagt.

**Bewijs:** een link naar de mislukte workflowrun en een korte toelichting waaruit blijkt dat publicatie en deployment zijn geblokkeerd.

### 5.2 Onderzoek revisies en logs

Zoek in de Azure Portal de **actieve revisie**, de **console logs** en de **system logs** van je Container App. Beschrijf kort waarvoor je elk van deze bronnen gebruikt.

### 5.3 Analyseer één foutscenario

Kies één scenario: een mislukte ACR-image-pull, een verkeerde target port of een mislukte OIDC-login.

Beschrijf op basis van de **relevante Azure/GitHub-documentatie** welke fout of status je verwacht, waar je de diagnose begint, hoe je de vermoedelijke oorzaak verifieert en hoe je die oplost. Je hoeft de werkende omgeving **niet expres kapot te maken**.

**Bewijs:** een beknopte, onderbouwde foutanalyse.

---

## Inleveren

Lever je werkende Azure-omgeving en je aangepaste GitHub Actions-workflow op. Maak in je repository het bestand `docs/azure-deployment.md` met uitsluitend:

1. **Architectuur:** een eenvoudig diagram en de gekozen Azure-resources.
2. **Toegang:** identiteiten, rollen, scopes en hun doel.
3. **Deploymentbewijs:** de vergelijking van twee succesvolle deployments, inclusief workflowlinks, SHA's en controles.
4. **Foutafhandeling:** de geblokkeerde `publish`-job, de gebruikte logbronnen en één onderbouwd foutscenario.

Houd de uitleg compact: beschrijf keuzes en bewijzen, geen stap-voor-stapverslag van alle Portal-klikken. Gebruik eventueel geschoonde screenshots of logfragmenten. Publiceer **geen secrets, tokens of wachtwoorden**.

### Definition of Done

- [ ] ACR, Container Apps Environment en Container App zijn zelfstandig opnieuw aangemaakt.
- [ ] dotNOVI draait via HTTPS met een private ACR-image en passende managed identities.
- [ ] De bestaande CI/CD-workflow publiceert en deployt uitsluitend na geslaagde controles.
- [ ] Image-tag, `APP_COMMIT_SHA` en `/version` komen overeen met de volledige commit-SHA.
- [ ] Twee verschillende commits zijn aantoonbaar succesvol gedeployed.
- [ ] Een falende CI-controle blokkeert de `publish`-job en laat de vorige versie draaien.
- [ ] Architectuur, rechten, deploymentbewijs en foutanalyse staan in `docs/azure-deployment.md`.

### Opruimen na beoordeling

Verwijder de oefenresources wanneer je ze niet meer nodig hebt. Controleer ook afzonderlijke logging- en monitoringresources. **Minimum replicas = 0 betekent niet dat alle Azure-resources kosteloos zijn.**

## Officiële documentatie

- [Azure Container Registry](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-intro)
- [Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/overview)
- [Azure CLI: Container App bijwerken](https://learn.microsoft.com/en-us/cli/azure/containerapp#az-containerapp-update)
- [Managed identity voor private ACR-images](https://learn.microsoft.com/en-us/azure/container-apps/managed-identity-image-pull)
- [GitHub Actions: jobs en afhankelijkheden](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-jobs)
- [Container Apps: revisies](https://learn.microsoft.com/en-us/azure/container-apps/revisions)
- [ACR: rollen en autorisatie](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-rbac-built-in-roles-overview)
