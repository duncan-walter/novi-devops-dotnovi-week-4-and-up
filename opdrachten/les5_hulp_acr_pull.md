# Technische hulp A – Een private ACR-image gebruiken in Azure Container Apps

> **Bij Praktijk A, opdracht A.4** in [LES5.md](./LES5.md). Gebruik dit document als je hulp nodig hebt bij de private image, de managed identity of de image pull. De opdracht gebruikt een ACR met **RBAC Registry Permissions (zonder ABAC)**.

## Wat ga je hier doen?

Je image `dotnovi:v1` staat in je eigen private Azure Container Registry (ACR). Azure Container Apps moet deze image kunnen ophalen, maar de ACR-adminaccount blijft uitgeschakeld. Je configureert daarom een **system-assigned managed identity** en laat die identity de image lezen met `AcrPull`-rechten.

Dit is **niet** de identiteit `id-github-dotnovi`: die gebruikt GitHub Actions om images te publiceren (`AcrPush`). Een aparte user-assigned pull-identity aanmaken is voor deze oefening niet nodig.

## 1. Maak eerst de Container App aan

1. Open de [Azure Portal](https://portal.azure.com) en maak een Container App met de naam `ca-dotnovi-lab` in Resource Group `rg-dotnovi-lab`.
2. Selecteer je Container Apps Environment `cae-dotnovi-lab` (maak die aan als deze nog niet bestaat).
3. Kies bij de containerimage voorlopig de **publieke quickstart-image** (`Simple hello world container`). Zo kan de Container App worden aangemaakt voordat de private registry-authenticatie is ingesteld.
4. Gebruik de overige instellingen uit [opdracht A.4](./LES5.md#a4--deploy-naar-azure-container-apps), waaronder externe ingress en de juiste target port zodra je dotNOVI gebruikt.

## 2. Vervang de quickstart-image door dotNOVI

1. Open `ca-dotnovi-lab` in de Azure Portal en ga naar **Containers → Edit and deploy**.
2. Open de containerconfiguratie en kies als **Image source**: **Azure Container Registry**.
3. Selecteer je eigen registry. Gebruik repository `dotnovi` en tag `v1`.
4. Kies bij **Authentication type** voor **Managed identity** en bij **Managed identity** voor **System assigned**.
5. Sla de configuratie op en maak de nieuwe revisie aan.

De system-assigned identity hoort bij deze Container App. De Portal probeert bij het aanmaken van de revisie automatisch de benodigde rol `AcrPull` op jouw ACR toe te kennen. De precieze Portal-benamingen kunnen per versie iets verschillen.

## 3. Controleer de rechten als de image niet wordt opgehaald

1. Open je Azure Container Registry en ga naar **Access control (IAM) → Role assignments**.
2. Controleer of de **system-assigned managed identity van `ca-dotnovi-lab`** de rol `AcrPull` op **de registry** heeft.
3. Ontbreekt de rol? Kies **Add → Add role assignment**, selecteer `AcrPull` en wijs de rol toe aan de managed identity van `ca-dotnovi-lab`. Hiervoor moet jouw Azure-account bevoegd zijn om rollen toe te kennen.
4. Controleer ook bij de Container App onder **Identity → System assigned** of de identity actief is. Nieuwe roltoewijzingen kunnen enige tijd nodig hebben om door te werken.

Gebruik niet `id-github-dotnovi` voor deze pullrechten: daarmee zou je de twee verantwoordelijkheden onnodig combineren.

## 4. Controleer de revisie

Open bij je Container App **Revisions and replicas** (of **Revision management**) en controleer of de revisie met `dotnovi:v1` start. Bekijk bij problemen de **system logs**.

| Symptoom | Wat controleer je? |
| --- | --- |
| Image kan niet worden opgehaald | Exacte **Login server**, repository `dotnovi` en tag `v1`; `AcrPull` voor de juiste identity |
| Registry wordt geweigerd ondanks `AcrPull` | Of ACR **ARM audience tokens** toestaat voor managed-identity-authenticatie; zie de Microsoft-documentatie hieronder |
| Nieuwe revisie start niet | Containerlogs, containerpoort en ingestelde ingress/target port |
| Je ziet nog de voorbeeldapp | Of de nieuwe revisie actief is en verkeer ontvangt |

> **Let op:** je hoeft voor deze opdracht geen Azure CLI of Docker op je eigen computer te installeren.

**Resultaat:** je Container App kan de private dotNOVI-image met tag `v1` ophalen zonder registrywachtwoorden op te slaan.

Raadpleeg voor Portal-stappen en aanvullende problemen de [Microsoft-documentatie over private ACR-images en managed identities](https://learn.microsoft.com/en-us/azure/container-apps/managed-identity-image-pull).
