# Leermomenten

## Leermoment 1
Het is mogelijk om via docker compose in `docker-compose.yaml` een environment file door te geven. Dit zorgt ervoor dat de inhoud van deze file op de host in de environment variabelen van de container terecht komt.

Wellicht handig van local development?

Voorbeeld `docker-compose.yaml`:
```yaml
services:
  app:
    build: .
    env_file:
      - .env
    ports:
      - "3000:3000"
```