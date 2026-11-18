# Leermomenten

## Leermoment 1
Volume mounting i.c.m. local development is cool.
Dit heb ik op de volgende manier bereikt:
1. Dockerfile CMD aanpassen: `CMD ["npm", "run", "dev"]`. Uit de package.json blijkt dat het "dev" script een nodemon server start die filewijzigingen in de gaten houdt en opnieuw build wanneer wijzigingen zijn ontdekt.
2. Nieuwe docker image builden met een aparte tag (`dotnovi-dev:v1`).
3. Container met een volume mount op de src folder starten o.b.v. de `dotnovi-dev:v1` image: 
   ```bash
   docker run -it -v ./src:/app/src -p 3000:3000 dotnovi-dev:v1
   ```
   
> **Interessant:** de container stop wanneer deze met CTRL+C wordt beëindigd, handig voor development, dan blijven resources niet per ongeluk in gebruik.