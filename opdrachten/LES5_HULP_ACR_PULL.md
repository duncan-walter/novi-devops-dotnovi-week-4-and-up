# Technische hulp A – Private ACR-image gebruiken in Azure Container Apps

> **Bij Praktijk A, opdracht A3.** Deze instructie hoort bij [LES5.md](./LES5.md). Ze gaat uit van een ACR met **RBAC Registry Permissions** (zonder ABAC).

## Waarom?

Een image in Azure Container Registry (ACR) is niet automatisch toegankelijk voor Azure Container Apps. Je geeft de Container App een eigen **user-assigned managed identity** met alleen pullrechten. Dit is **niet** dezelfde identiteit als `id-github-dotnovi` waarmee GitHub Actions inlogt.

## 1. Maak een managed identity aan

1. Open de [Azure Portal](https://portal.azure.com) en zoek **Managed Identities**.
2. Kies **Create** en selecteer je Azure for Students-subscription.
3. Resource group: `rg-dotnovi-lab`.
4. Naam: `id-acr-pull-dotnovi`; regio: **West Europe** (of dezelfde ondersteunde regio als je lab).
5. Maak de identity aan.

## 2. Geef de identity toegang tot de registry

1. Open je eigen **Azure Container Registry** in de Portal.
2. Kies **Access control (IAM) → Add → Add role assignment**.
3. Selecteer **AcrPull** en vervolgens als lidtype **Managed identity**.
4. Selecteer `id-acr-pull-dotnovi` en voltooi de roltoewijzing.

`AcrPull` staat het ophalen van images toe, niet het publiceren ervan. Wijs de rol toe op **de registry zelf**, niet op je hele subscription.

> Bij een registry met **RBAC Registry + ABAC Repository Permissions** is `AcrPull` niet de juiste rol. Gebruik in dat geval de passende repositoryreader-rol. Voor ronde 1 gebruik je de in A1 voorgeschreven **RBAC Registry Permissions**.

## 3. Koppel de identity aan je Container App

1. Als je nog geen Container App hebt, maak er eerst één in `cae-dotnovi-lab` met een **tijdelijke publieke image**; configureer de overige basisinstellingen zoals in Praktijk A.
2. Open de Container App en ga naar **Identity → User assigned → Add**.
3. Selecteer `id-acr-pull-dotnovi` en sla de wijziging op.

## 4. Gebruik de private ACR-image

1. Open de configuratie van de container (via **Containers** of **Revisions and replicas → Create new revision**, afhankelijk van de Portal-versie).
2. Selecteer je **Azure Container Registry** en de gepubliceerde dotNOVI-image met tag `v1`.
3. Kies **Managed identity** voor registry-authenticatie en selecteer `id-acr-pull-dotnovi`.
4. Sla de configuratie op, maak zo nodig een nieuwe revisie en controleer dat deze start.

> Sommige Portal-schermen bieden de koppeling met de managed identity al tijdens het aanmaken van een Container App aan. Gebruik dan dezelfde identity en rechten; de volgorde van de schermen kan afwijken.

## 5. Controleer de image pull

Controleer bij **Revisions and replicas** of je nieuwe revisie start en of de juiste image met tag `v1` is geselecteerd. Bekijk bij een fout de **system logs**.

Controleer bij een fout achtereenvolgens:

- Kloppen de exacte ACR-loginserver, repository en tag?
- Is de managed identity aan de Container App gekoppeld?
- Heeft diezelfde identity `AcrPull` op **de juiste registry**?
- Staat in ACR onder **Settings → Properties** het gebruik van **ARM audience tokens** voor authenticatie toe? Dit kan nodig zijn voor managed-identity image pulls.
- Heeft Azure voldoende tijd gehad om een nieuwe roltoewijzing te verwerken?

**Resultaat:** de Container App haalt de private image uit ACR op zonder dat je een registrywachtwoord opslaat.

**Documentatie:** [Microsoft Learn – Managed identity image pull](https://learn.microsoft.com/en-us/azure/container-apps/managed-identity-image-pull).
