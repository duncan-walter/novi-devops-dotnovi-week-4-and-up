# Leermomenten

## Leermoment 1
Tijdens het uitvoeren van de opdracht moest geprobeerd worden om een `POST` request uit te voeren om een nieuwe note aan te maken. Dit lukte natuurlijk niet omdat de notes tabel nog niet bestond.

Na wat zoekwerk had ik [de documentatie](https://hub.docker.com/_/postgres) van de PostgreSQL docker image gevonden. Hierin staat vermeld dat sql bestanden en scripts tijdens de initialisatie van de container worden uitgevoerd als deze in de `/docker-entrypoint-initdb.d/` folder in de container zijn geplaatst.

Aangezien de repository al een [`init.sql`](/src/db/init.sql) bevat was de oplossing eenvoudig: voeg een volume toe die het sql-bestand bind aan `/docker-entrypoint-initdb.d/init.sql`.

> **Let op:** sql-bestanden en scripts in `/docker-entrypoint-initdb.d/` worden enkel uitgevoerd wanneer de [`/db-data`](/db-data) folder (host) en dus `/var/lib/postgresql` (container) niet aanwezig zijn. Lokaal seeding nodig? Verwijder dan de db-data folder.

## Leermoment 2
Single quotes werken niet in cmd (Windows) en double quotes moeten met een backslash escaped worden. Dit geeft voor het voorbeeld request van de opdracht het volgende curl-commando:

```bash
curl -X POST http://localhost:3000/api/notes -H "Content-Type: application/json" -d "{\"title\":\"Compose test\",\"content\":\"Het werkt!\"}"
```