# Leermomenten

## Leermoment 1

De verbinding met de database container wou niet in één keer lukken. Na wat debuggen van de database connectie tussen de
containers kwam ik erachter dat de app container
verbinding probeerde te maken met een andere postgres instantie. Dit komt doordat ik op mijn machine zelf PostgreSQL
geïnstalleerd heb staan, de app container probeerde hiermee te verbinden met credentials die niet bekend zijn binnen die
instantie. De oplossing was om voor de db container een andere externe poort te gebruiken, waarmee de app container een
verbinding zoekt.

App container:

```bash
docker run -d -p 3000:3000 -e DATABASE_URL=postgresql://dotnovi:devops123@host.docker.internal:5433/dotnovi --name dotnovi dotnovi:v1
```

Db container (credentials zijn waardeloos dus deze kunnen gecommit worden:

```bash
docker run -d --name dotnovi-db -e POSTGRES_USER=dotnovi -e POSTGRES_PASSWORD=devops123 -e POSTGRES_DB=dotnovi -p 5433:5432 postgres:18-alpine
```