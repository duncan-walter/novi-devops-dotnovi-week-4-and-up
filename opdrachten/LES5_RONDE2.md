# Les 5 – Cloud Deployment met Microsoft Azure

## Ronde 2 – Bouw het zelfstandig en verantwoord

**Uitvoering:** zelfstandig, na les 5  
**Voorkennis:** de werkende deployment uit [ronde 1](./LES5.md)

### Situatie

In ronde 1 heb je dotNOVI naar Azure gedeployed: je hebt een image gepubliceerd naar Azure Container Registry (ACR), een Container App geconfigureerd en GitHub Actions uitgebreid met automatische deployment.

Nu begin je opnieuw. Verwijder de tijdelijke Azure-deploymentomgeving en bouw de infrastructuur zelfstandig opnieuw op. Dit keer krijg je **geen gedetailleerd stappenplan**. Je gebruikt documentatie en onderbouwt zelf je keuzes.

**Einddoel:** een veilige, controleerbare, herleidbare en volledig geautomatiseerde deploymentketen.

### Leerdoelen

Na afloop kun je:

- Zelfstandig een containerapplicatie hosten in Azure.
- CI uitbreiden tot CI/CD met verplichte kwaliteitscontroles.
- Een draaiende applicatie terugvoeren op een specifieke Git-commit.
- Veilige authenticatie en passende autorisatie inrichten.
- Aantonen dat falende kwaliteitscontroles deployment voorkomen.
- Een deployment controleren en technische problemen systematisch onderzoeken.

---

## Opdracht 1 – Bouw je Azure-infrastructuur opnieuw op

### 1.1 Verwijder de tijdelijke oefeninfrastructuur

Verwijder de volgende resources uit ronde 1:

- Azure Container App;
- Container Apps Environment;
- Azure Container Registry.

Je mag je **Resource Group**, de bestaande GitHub Actions-identiteit, de ACR-pullidentiteit, je broncode en je CI-workflow behouden. Controleer ook eventueel aangemaakte logging- of monitoringresources. Verwijder geen resources die niet bij deze oefening horen.

**Let op:** verwijder de oefenomgeving pas nadat je het resultaat van ronde 1 hebt gecontroleerd.

### 1.2 Richt de infrastructuur opnieuw in

Maak zelfstandig de resources aan die nodig zijn om dotNOVI op Azure Container Apps te hosten.

**Eisen:**

1. Alle resources staan in je eigen Azure for Students-subscription.
2. De resources zijn logisch gegroepeerd en herkenbaar benoemd.
3. Docker-images worden opgeslagen in **Azure Container Registry**.
4. dotNOVI draait op **Azure Container Apps** en is via HTTPS bereikbaar.
5. Ingress en target port sluiten aan bij de containerconfiguratie.
6. CPU, geheugen en schaalinstellingen zijn passend en kostenbewust gekozen.
7. De registry blijft privé; gebruik geen registry-adminaccount voor runtime-authenticatie.

**Nog niet nodig:** Azure Database for PostgreSQL. Dit volgt in een latere opdracht.

**Op te leveren:** een werkende Azure-omgeving en een architectuurdiagram van de gebruikte resources en verbindingen.

---

## Opdracht 2 – Configureer veilige authenticatie en autorisatie

Je deploymentketen gebruikt verschillende identiteiten. Geef iedere identiteit uitsluitend de rechten die nodig zijn voor de betreffende taak.

### 2.1 GitHub Actions → Azure

Gebruik je bestaande OIDC-federatie. Zorg dat GitHub Actions:

- Zonder langdurig Azure-wachtwoord inlogt;
- Docker-images naar ACR kan publiceren;
- De bestaande Container App kan bijwerken;
- Alleen de noodzakelijke rechten krijgt op passende scopes.

### 2.2 Azure Container Apps → ACR

Gebruik een managed identity waarmee Container Apps de private image uit ACR kan ophalen. Geef deze identiteit alleen de noodzakelijke pull-/leesrechten.

### 2.3 Onderbouw je keuzes

Leg per identiteit vast:

- Welk systeem gebruikt haar?
- Waarvoor is de identiteit nodig?
- Welke Azure-rol(len) zijn toegewezen?
- Op welke resource of scope gelden deze rechten?
- Waarom zijn deze rechten toereikend?

Houd rekening met het autorisatiemodel van ACR: RBAC-only en ABAC-enabled registries hebben verschillende push- en pullrollen.

**Op te leveren:** veilige en werkende identiteiten, plus een korte onderbouwing van rollen en scopes. Zet geen geheimen of tokens in de repository.

---

## Opdracht 3 – Bouw een volledige CI/CD-pipeline

Breid je bestaande GitHub Actions-workflow uit tot de keten:

**Build → Test → Publish → Deploy → Verify**

### 3.1 Kwaliteitscontroles

Je bestaande test- en lintcontroles moeten slagen voordat een image wordt gepubliceerd of gedeployed. Configureer de jobafhankelijkheden zo dat falende kwaliteitscontroles de deployment blokkeren.

### 3.2 Unieke imageversies

Gebruik bij iedere succesvolle deployment een Docker-image-tag gebaseerd op de **Git-commit-SHA**. Gebruik niet uitsluitend een veranderlijke tag zoals `latest`.

### 3.3 Automatisch publiceren

Na succesvolle CI-controles bouwt GitHub Actions een Docker-image en publiceert deze naar ACR, getagd met de commit-SHA van de betreffende workflowrun.

### 3.4 Automatisch deployen

Laat GitHub Actions de bestaande Container App bijwerken naar de gepubliceerde image. De workflow moet de juiste subscription, Resource Group, applicatie en registry gebruiken. De automatische deployment draait uitsluitend voor de afgesproken deploymentbranch.

### 3.5 Commit-SHA in de applicatie

dotNOVI bevat `/version`, dat onder meer de commit-SHA uit `APP_COMMIT_SHA` teruggeeft.

Zorg dat de applicatie de **juiste** SHA ontvangt, bijvoorbeeld via een environment variable in de Container App. **Een SHA-tag op de Docker-image alleen is niet voldoende** om `APP_COMMIT_SHA` in het proces te zetten.

**Op te leveren:** een werkende GitHub Actions-workflow die na geslaagde controles automatisch een herleidbare image publiceert en deployt.

---

## Opdracht 4 – Verifieer je deployment

Een groene workflow garandeert nog niet dat de juiste applicatie draait.

### 4.1 Controleer de hele keten

Toon aan dat:

1. Alle verplichte CI-jobs succesvol zijn afgerond.
2. De image met de verwachte SHA in ACR staat.
3. Azure Container Apps die image in de actieve revisie gebruikt.
4. dotNOVI via HTTPS bereikbaar is.
5. `/version` de verwachte commit-SHA teruggeeft.
6. `/health/live` HTTP 200 teruggeeft.

De **workflowcommit, image-tag en SHA uit `/version` moeten overeenkomen**.

### 4.2 Controleer de gezondheid

| Endpoint | Verwacht | Betekenis |
| --- | --- | --- |
| `/version` | HTTP 200 | Versie-informatie is bereikbaar |
| `/health/live` | HTTP 200 | Express verwerkt HTTP-verzoeken |
| `/health/ready` | HTTP 503 | PostgreSQL is nog niet beschikbaar |

Leg uit waarom `/health/ready` in deze opdracht HTTP 503 kan geven terwijl de containerdeployment wel is geslaagd. Gebruik `/health/ready` nu **niet** als verplichte HTTP-readinessprobe; dat zou de ontbrekende database ten onrechte als deploymentfout laten behandelen.

### 4.3 Controleer een tweede succesvolle deployment

Maak een kleine nieuwe codewijziging en push deze naar de deploymentbranch. Laat zien dat:

- De pipeline opnieuw start en slaagt;
- Een **andere** commit-SHA-tag in ACR verschijnt;
- Container Apps de nieuwe image deployt;
- `/version` de nieuwe SHA teruggeeft.

**Op te leveren:** bewijs van **twee opeenvolgende succesvolle deployments** met verschillende commit-SHA's.

---

## Opdracht 5 – Onderzoek een mislukte deployment

### 5.1 Bewijs dat kwaliteitscontroles deployment blokkeren

Introduceer tijdelijk een kleine, veilige fout waardoor een bestaande test of lintcontrole faalt. Commit en push deze naar de deploymentbranch.

Controleer dat:

1. De bedoelde kwaliteitscontrole faalt;
2. De deploymentjob wordt overgeslagen;
3. Er geen nieuwe image voor deze falende commit wordt gepubliceerd;
4. De Container App de laatst succesvol gedeployde versie blijft gebruiken.

Herstel daarna de fout en controleer dat de pipeline opnieuw succesvol kan deployen. Veroorzaak geen destructieve wijzigingen of dataverlies.

### 5.2 Onderzoek logs en revisies

Zoek in de Azure Portal op:

- De actieve revisie en containerimage;
- De **console logs** van de applicatie;
- De **system logs** van Container Apps.

Beschrijf kort welke informatie je in de verschillende logtypen kunt vinden.

### 5.3 Analyseer een foutscenario

Kies één scenario:

- Container Apps kan een private image niet uit ACR ophalen;
- De applicatie is onbereikbaar door een verkeerde target port;
- GitHub Actions kan niet authenticeren bij Azure.

Beschrijf:

1. Welk symptoom je zou verwachten;
2. Waar je met onderzoeken begint;
3. Welke logs en statusinformatie relevant zijn;
4. Wat een waarschijnlijke oorzaak is;
5. Hoe je die oorzaak zou verifiëren en het probleem zou oplossen.

Je hoeft hiervoor geen defect in je werkende cloudomgeving te veroorzaken.

**Op te leveren:** bewijs van een geblokkeerde deployment en een korte, onderbouwde troubleshootinganalyse.

---

## Eindresultaat en inleveren

Je levert op:

- [ ] Werkende dotNOVI-deployment met private image in ACR.
- [ ] Architectuurdiagram van de deploymentomgeving.
- [ ] GitHub Actions-workflow met verplichte CI-controles en automatische CD.
- [ ] Unieke image-tags op basis van commit-SHA.
- [ ] Correct ingestelde `APP_COMMIT_SHA` en overeenkomende `/version`-response.
- [ ] Werkende `/health/live`-controle via HTTPS.
- [ ] OIDC-authenticatie en managed identities met onderbouwde rechten.
- [ ] Bewijs van twee succesvolle opeenvolgende deployments.
- [ ] Bewijs dat falende CI-controles deployment blokkeren.
- [ ] Korte analyse van een deploymentprobleem.

### Technische documentatie

Maak in je repository `docs/azure-deployment.md` aan met:

1. **Architectuur:** diagram van GitHub Actions, ACR, Container Apps en de gebruikte identiteiten.
2. **Configuratie:** belangrijkste Azure-resources, instellingen en keuzes.
3. **Beveiliging:** gebruikte identiteiten, rollen en scopes.
4. **Verificatie:** links naar succesvolle workflowruns, commit-SHA's en bewijs van image, revisie en `/version`.
5. **Foutafhandeling:** de falende kwaliteitscontrole en je troubleshootinganalyse.

Gebruik eventueel screenshots en relevante (geschoonde) logfragmenten; publiceer **geen tokens, wachtwoorden of andere gevoelige gegevens**.

### Afronding en opruimen

Als de opdracht is beoordeeld of je de oefenomgeving niet meer nodig hebt, verwijder je de aangemaakte Azure-resources. Controleer ook eventuele logging- en monitoringresources. Minimum replicas op 0 betekent niet automatisch dat al je Azure-resources kosteloos zijn.

## Officiële documentatie

- [Azure Container Registry](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-intro)
- [Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/overview)
- [GitHub Actions met Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/github-actions)
- [Managed identity voor private ACR-images](https://learn.microsoft.com/en-us/azure/container-apps/managed-identity-image-pull)
- [Revisions in Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/revisions)
- [Health probes in Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/health-probes)
- [Azure Container Registry RBAC en ABAC](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-rbac-abac-repository-permissions)
