# Les 2: Docker Containerisatie

## Opdrachtbeschrijving

Je hebt in les 1 kennisgemaakt met dotNOVI. De applicatie draait inmiddels op je eigen computer, maar hoe zorg je ervoor dat deze ook op andere computers kan draaien zonder dat je daar eerst Node.js en alle dependencies moet installeren?

Daar gaan we deze week mee aan de slag!

Je gaat dotNOVI **containeriseren met Docker**. Je schrijft een Dockerfile, bouwt een Docker-image en start de applicatie in een container. Daarna start je ook een PostgreSQL-container, zodat de applicatie gebruik kan maken van een database.

## Opdrachten

### Opdracht 1: Een Dockerfile schrijven

In deze opdracht maak je dotNOVI geschikt om in een Docker-container te draaien.

1. Zorg dat Docker geïnstalleerd is en controleer of het werkt:

   ```bash
   docker --version
   docker run hello-world
   ```

2. Als je de dotNOVI applicatie al op je computer hebt, kun je daarin verder werken. Is dit de eerste keer dat je met deze applicatie werkt? Clone dan deze repository naar je computer en push de code naar je eigen repository.

3. Maak in de root van het project een bestand met de naam `Dockerfile`. Zorg ervoor dat dit bestand:
   - `node:20-alpine` als base image gebruikt.
   - `/app` als werkdirectory instelt.
   - Eerst `package.json` en `package-lock.json` kopieert.
   - Met `npm ci --omit=dev` de production dependencies installeert.
   - De applicatiecode uit `src/` kopieert.
   - Poort `3000` beschikbaar stelt met `EXPOSE`.
   - De applicatie start met `npm start`.

4. Maak een `.dockerignore` waarin je minimaal `node_modules`, `.git` en `.env` uitsluit.

**Vraag:** Waarom is het verstandig om eerst `package.json` en `package-lock.json` te kopiëren en de dependencies te installeren, voordat je de applicatiecode kopieert?

### Opdracht 2: Een Docker-image bouwen en starten

Je gaat nu de Dockerfile gebruiken om dotNOVI in een container te draaien.

1. Bouw een Docker-image met de naam `dotnovi:v1`:

   ```bash
   docker build -t dotnovi:v1 .
   ```

2. Controleer met `docker images` of je image is aangemaakt.

3. Start een container:

   ```bash
   docker run -d -p 3000:3000 --name dotnovi dotnovi:v1
   ```

4. Open http://localhost:3000 en controleer of de applicatie bereikbaar is. Bekijk ook de livenesscheck op http://localhost:3000/health/live.

   De database is nog niet gekoppeld. Het is dus normaal dat sommige onderdelen niet werken.

5. Bekijk de logs van je container:

   ```bash
   docker logs dotnovi
   ```

6. Gebruik `docker exec -it dotnovi sh` om de container te openen. Bekijk welke bestanden erin staan.

7. Stop en verwijder de container:

   ```bash
   docker stop dotnovi
   docker rm dotnovi
   ```

**Vraag:** Wat is het verschil tussen een Docker-image en een Docker-container?

### Opdracht 3: PostgreSQL in een container draaien

dotNOVI gebruikt PostgreSQL om notities op te slaan. Je gaat nu een tweede container starten voor de database en deze verbinden met de applicatie.

1. Maak een Docker-netwerk zodat beide containers met elkaar kunnen communiceren:

   ```bash
   docker network create dotnovi-network
   ```

2. Start een PostgreSQL-container op dit netwerk:

   ```bash
   docker run -d \
     --name dotnovi-db \
     --network dotnovi-network \
     -e POSTGRES_USER=dotnovi \
     -e POSTGRES_PASSWORD=devops123 \
     -e POSTGRES_DB=dotnovi \
     postgres:18-alpine
   ```

   Dit wachtwoord is uitsluitend bedoeld voor de lokale oefening.

3. Maak de benodigde databasetabel aan met het meegeleverde SQL-bestand:

   ```bash
   docker exec -i dotnovi-db \
     psql -U dotnovi -d dotnovi < src/db/init.sql
   ```

4. Start de applicatie opnieuw en geef de databaseverbinding mee als environment variable:

   ```bash
   docker run -d \
     --name dotnovi \
     --network dotnovi-network \
     -p 3000:3000 \
     -e DATABASE_URL=postgresql://dotnovi:devops123@dotnovi-db:5432/dotnovi \
     dotnovi:v1
   ```

5. Open http://localhost:3000 en controleer of de applicatie nu met de database werkt. Controleer ook http://localhost:3000/health/ready. Deze endpoint moet HTTP 200 teruggeven als de databaseverbinding werkt.

6. Bekijk de logs als iets niet werkt. Stop en verwijder na afloop beide containers en het Docker-netwerk.

**Vraag:** Waarom gebruiken we in de databaseverbinding de hostnaam `dotnovi-db` in plaats van `localhost`?

### Opdracht 4: Je werk opleveren

1. Commit je `Dockerfile` en `.dockerignore` met een duidelijke commit message.
2. Push je wijzigingen naar GitHub.
3. Maak een Pull Request naar `main`.
4. Beschrijf in je PR wat je hebt gemaakt en hoe je hebt gecontroleerd dat de applicatie in Docker werkt.
5. Merge je PR.

## Volgende les

In les 3 ga je Docker Compose gebruiken om meerdere containers eenvoudiger samen te starten en te beheren. Ook ga je verder aan de slag met het optimaliseren en publiceren van Docker-images.

## Resources

- [Docker Get Started](https://docs.docker.com/get-started/)
- [Dockerfile Reference](https://docs.docker.com/reference/dockerfile/)
- [Docker Networking](https://docs.docker.com/engine/network/)