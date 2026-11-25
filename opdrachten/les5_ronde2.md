# Les 5 – Cloud Deployment met Microsoft Azure (Ronde 2 huiswerk)

Tijdens de les heb je een containerimage gepubliceerd, handmatig gedeployed en daarna automatische deployment ingericht. In deze ronde laat je zien dat je:

1. Zelfstandig een Azure-omgeving opnieuw kunt opbouwen;
2. Veilige toegang kunt inrichten en de gebruikte rechten kunt uitleggen;
3. Elke deployment kunt herleiden tot één Git-commit;
4. Kunt aantonen dat falende CI-controles publicatie en deployment blokkeren;
5. Een deployment kunt verifiëren en storingen systematisch onderzoeken.

Je gaat de Azure-infrastructuur opnieuw opbouwen, maar behoud en verbetert de bestaande CI/CD-workflow. Gebruik hiervoor de documentatie in plaats van het stappenplan uit ronde 1. Je hoeft geen Docker of Azure CLI op je laptop te installeren; de commando's draaien in GitHub Actions.

Aan het eind van deze opdracht heb je twee succesvolle deployments met verschillende commit-SHA's, een test waarbij falende CI publicatie blokkeert, en beknopte technische documentatie.

---

## Opdracht 1 – Bouw de Azure-infrastructuur opnieuw op

### 1.1 Ruim de oefenomgeving uit ronde 1 op

Controleer eerst of je resultaat van ronde 1 werkt. Verwijder daarna **alleen** de volgende oefenresources:

- de Azure Container App `ca-dotnovi-lab`;
- de Container Apps Environment `cae-dotnovi-lab`;
- je Azure Container Registry.

Behoud de Resource Group `rg-dotnovi-lab`, de GitHub Actions-identity `id-github-dotnovi`, de OIDC-federatie, je broncode en je bestaande CI/CD-workflow. Controleer of er nog logging- of monitoringresources van de oefening bestaan.

De system-assigned managed identity verdwijnt met de verwijderde Container App. Maak bij de nieuwe Container App opnieuw een system-assigned identity aan. Roltoewijzingen op de verwijderde ACR moet je bij de nieuwe registry opnieuw instellen.

### 1.2 Richt de omgeving zelfstandig opnieuw in

Maak de resources aan die nodig zijn om dotNOVI als HTTP-service te draaien. Zorg dat het voldoet aan de volgende eisen:
- Gebruik je eigen Azure for Students-subscription en herkenbare resourcenamen;
- Publiceer images naar een **private Azure Container Registry**; laat het registry-adminaccount uitgeschakeld;
- Host dotNOVI in Azure Container Apps, bereikbaar via HTTPS;
- Configureer _ingress_ en de _target port_ passend bij de applicatie;
- Kies onderbouwde, kostenbewuste instellingen voor CPU, geheugen en replicas;
- Richt nog geen PostgreSQL-database in. Dit volgt in een latere les.

## Opdracht 2 – Richt toegang veilig in
Gebruik de bestaande OIDC-koppeling uit de voorbereiding. Zorg dat:

- `id-github-dotnovi` images naar de nieuwe ACR kan publiceren en de opnieuw aangemaakte Container App kan bijwerken;
- de system-assigned identity van de nieuwe Container App de private image kan ophalen;
- iedere identiteit alleen de benodigde rollen heeft, op zo klein mogelijke scopes.

> _Let op:_ `AcrPush` en `AcrPull` gebruik je bij een registry met **RBAC Registry Permissions (zonder ABAC)**, zoals in ronde 1. Controleer het gekozen autorisatiemodel van je nieuwe ACR.

Neem onderstaande tabel over in `docs/azure-deployment.md` en vul de tabel met de roltoewijzingen die jij hebt ingericht. Beschrijf per roltoewijzing waarvoor deze nodig is, op welke Azure-resource de rol geldt en waarom je voor deze rechten en scope hebt gekozen. Voeg indien nodig rijen toe.

| Identiteit | Doel | Azure-rol | Scope | Waarom is deze rol nodig? |
|---|---|---|---|---|
| `id-github-dotnovi` | ... | ... | ... | ... |
| `id-github-dotnovi` | ... | ... | ... | ... |
| System-assigned identity van `ca-dotnovi-lab` | ... | ... | ... | ... |

## Opdracht 3 – Maak je bestaande CI/CD-pipeline herleidbaar

Je hebt al een werkende CI-workflow met een `publish`-job. Je verbetert deze workflow zodat de volgende keten ontstaat: _CI-controles → Image bouwen en publiceren → Bestaande Container App bijwerken → Resultaat verifiëren._

### 3.1 Behoud de kwaliteitsgrens

De `publish`-job mag pas starten nadat alle verplichte test- en lintjobs succesvol zijn afgerond. Behoud hiervoor de juiste afhankelijkheden met `needs:`. Publiceer en deploy alleen vanaf de afgesproken OIDC/deploymentbranch.

### 3.2 Gebruik een commit-SHA als image-tag

Vervang de vaste tag `v2` door de volledige Git-commit-SHA van de betreffende workflowrun. Gebruik dezelfde SHA bij het bouwen, publiceren en deployen van de image. Gebruik in deze opdracht geen andere tags, zoals `latest` of `v2`.

De imageverwijzing heeft de vorm:

```text
<jouw-acr-login-server>/dotnovi:<volledige-commit-sha>
```

### 3.3 Geef de SHA door aan de applicatie

Het endpoint `/version` leest de commit-SHA uit de environment variable `APP_COMMIT_SHA`. Zorg dat de deployment dezelfde volledige SHA instelt als:

- tag van de gepubliceerde Docker-image;
- waarde van `APP_COMMIT_SHA` in de Container App.

**Let op: een image-tag alleen stelt `APP_COMMIT_SHA` niet in. Onderzoek hoe je in GitHub Actions de commit-SHA verkrijgt en hoe je bij `az containerapp update` zowel de image als de environment variable bijwerkt.

Je hebt het goed gedaan wanneer de SHA van de workflowrun, de image-tag en de waarde die `/version` retourneert exact gelijk zijn.

## Opdracht 4 – Bewijs twee succesvolle deployments

### 4.1 Controleer de eerste deployment
Controleer na een succesvolle push:

1. Alle verplichte CI-controles zijn geslaagd vóór `publish`.
2. ACR bevat `dotnovi` met de juiste commit-SHA-tag.
3. De actieve Container Apps-revisie gebruikt die image.
4. `/version` retourneert HTTP 200 en de juiste SHA.
5. `/health/live` retourneert HTTP 200 via HTTPS.

`/health/ready` kan HTTP 503 teruggeven omdat PostgreSQL nog ontbreekt. Gebruik dit endpoint in deze ronde niet als verplichte readinessprobe. Leg in één zin uit waarom dat geen mislukte containerdeployment hoeft te betekenen en voeg dit toe aan `docs/azure-deployment.md`.

### 4.2 Controleer een tweede deployment

Maak een kleine zichtbare wijziging die geen database nodig heeft. Commit en push die naar de deploymentbranch. Controleer opnieuw de hele keten. Leg beide deployments vast in je documentatie:

| Controle | Deployment 1 | Deployment 2 |
| --- | --- | --- |
| Commit-SHA | Volledige SHA | Andere volledige SHA |
| Image-tag in ACR | Dezelfde SHA | Dezelfde SHA |
| `/version` | Dezelfde SHA | Dezelfde SHA |
| GitHub Actions-run | Link | Link |

Voeg bewijs toe van de actieve image/revisie en de geslaagde `/health/live`-controle. Twee screenshots zonder koppeling met de commit zijn onvoldoende.

## Opdracht 5 – Onderzoek fouten en blokkeringen

### 5.1 Toon aan dat falende CI publicatie blokkeert

1. Introduceer tijdelijk een veilige fout waardoor een bestaande test of lintcontrole faalt.
2. Commit en push de fout naar de deploymentbranch.
3. Controleer dat de CI-controle faalt en de `publish`-job wordt overgeslagen.
4. Controleer dat voor deze falende SHA géén image in ACR is gepubliceerd en dat de Container App op de laatst geslaagde versie blijft.
5. Herstel de fout en zorg dat de pipeline weer slaagt.

Voeg een link toe in jouw documentatie naar de mislukte workflowrun, samen met een korte toelichting waaruit blijkt dat publicatie en deployment zijn geblokkeerd.

### 5.2 Onderzoek de status van je deployment
Een succesvolle GitHub Actions-workflow betekent niet automatisch dat je applicatie correct draait. Onderzoek daarom de status van je Container App in Azure.

1. Open je Container App in de Azure Portal en onderzoek de **actieve revisie**. Controleer welke Docker-image draait, hoeveel replicas actief zijn en of er fouten of waarschuwingen zichtbaar zijn.

> **Let op:** bij een minimum van nul replicas kan de applicatie zijn teruggeschaald wanneer er geen verkeer is. Dit betekent niet automatisch dat er een probleem is.

3. Bekijk de console logs en system logs. Zoek naar informatie waarmee je kunt vaststellen of de applicatie correct is gestart en of Azure problemen heeft ondervonden bij het uitvoeren van de container.
4. Beantwoord de volgende vragen:
   - Welke informatie vertelt de revisiestatus je over de deployment, en wat kun je hieruit niet afleiden?
   - Hoe kun je met behulp van de logs onderscheid maken tussen een probleem in de applicatie en een probleem met de Azure-infrastructuur of containerconfiguratie?
   - Welke aanvullende controle zou je uitvoeren om vast te stellen dat de applicatie daadwerkelijk beschikbaar is voor gebruikers?

Beschrijf je bevindingen aan de hand van de daadwerkelijke revisiestatus en logs uit jouw Azure-omgeving. Onderbouw je antwoorden met relevante voorbeelden uit de getoonde informatie.

### 5.3 Analyseer één foutscenario

Kies één scenario: een mislukte ACR-image-pull, een verkeerde target port of een mislukte OIDC-login. Beschrijf op basis van de officiële documentatie:
1. Welke foutmelding of status je verwacht.
2. Welke informatie je als eerste onderzoekt en waarom.
3. Hoe je de vermoedelijke oorzaak bevestigt.
4. Welke aanpassing het probleem zou oplossen.

Je hoeft de storing niet daadwerkelijk te veroorzaken. Voeg de foutanalyse toe aan je documentatie.

---

### Definition of Done

- [ ] ACR, Container Apps Environment en Container App zijn zelfstandig opnieuw aangemaakt.
- [ ] dotNOVI draait via HTTPS met een private ACR-image en passende managed identities.
- [ ] De bestaande CI/CD-workflow publiceert en deployt uitsluitend na geslaagde controles.
- [ ] Image-tag, `APP_COMMIT_SHA` en `/version` komen overeen met de volledige commit-SHA.
- [ ] Twee verschillende commits zijn aantoonbaar succesvol gedeployed.
- [ ] Een falende CI-controle blokkeert de `publish`-job en laat de vorige versie draaien.
- [ ] Architectuur, rechten, deploymentbewijs en foutanalyse staan in `docs/azure-deployment.md`.

### Opruimen

Verwijder de oefenresources wanneer je ze niet meer nodig hebt. Controleer ook afzonderlijke logging- en monitoringresources. **Minimum replicas = 0 betekent niet dat alle Azure-resources kosteloos zijn!**

## Officiële documentatie

- [Azure Container Registry](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-intro)
- [Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/overview)
- [Azure CLI: Container App bijwerken](https://learn.microsoft.com/en-us/cli/azure/containerapp#az-containerapp-update)
- [Managed identity voor private ACR-images](https://learn.microsoft.com/en-us/azure/container-apps/managed-identity-image-pull)
- [GitHub Actions: jobs en afhankelijkheden](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-jobs)
- [Container Apps: revisies](https://learn.microsoft.com/en-us/azure/container-apps/revisions)
- [ACR: rollen en autorisatie](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-rbac-built-in-roles-overview)
