# Leermomenten

## Leermoment 1
`docker compose up -d --scale app=3` werkte niet met de inhoud van mijn huidige docker-compose. Dit kwam omdat de app service een `container_name` had gespecificeerd, dat leek mij eerder handig omdat de gegenereerde naam te omslachtig was. Daarnaast was de scaling niet mogelijk omdat voor de app service een host port was opgegeven. De host port moet worden verwijder om het `--scale` argument te kunnen gebruiken.

Voorbeeld `docker-compose.yaml` waarbij `--scale` wel kan worden gebruikt (niet gecommit):

```yaml
services:
  app:
    build: .
    ports:
      - "3000"
    environment:
      DATABASE_URL: postgresql://dotnovi:devops123@db:5432/dotnovi # local only, save to commit
  db:
    image: postgres:18-alpine # docs: https://hub.docker.com/_/postgres
    environment:
      POSTGRES_USER: dotnovi # local only, save to commit
      POSTGRES_PASSWORD: devops123 # local only, save to commit
      POSTGRES_DB: dotnovi # local only, save to commit
    volumes:
      - ./db-data:/var/lib/postgresql # Data
      - ./src/db/init.sql:/docker-entrypoint-initdb.d/init.sql # Init script
```