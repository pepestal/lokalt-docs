Du kommer nu att få en uppgift. Jag vill att du hanterar uppgiften i tre steg;

1. Undersök
För att uppgiften ska bli korrekt utförd behöver du först undersöka vilka system, regler gällande UI, API-routes som finns - beroende på uppgiftens karaktär.

Dokumentation för projektet finns i respektive rotmapps `.agents/`, `.claude/` och
`docs/`. Ett projekt kan ha flera rotmappar (t.ex. en för frontend och en för backend) —
leta i var och en. Finns en dokumentationskarta eller ett index (t.ex. `docs/README.md`
eller motsvarande) — börja där.

Projektet kan vara live på en publik adress. Krävs inloggningsuppgifter för att komma åt
sidan finns de i `password.md` i LOKALT-roten (gitignorad — finns bara lokalt och checkas
aldrig in).

2. Planering
Innan uppgiften genomförs vill jag att du överväger olika alternativa lösningar, och analyserar för- och nackdelar för dessa. Det är viktigt att genomförandet inte orsakar problem för någon annan del i projektet, samt att redan befintlig arkitektur används på ett optimalt sätt. Det är också viktigt att fil- och mappstruktur respekteras, och att planering görs vart eventuellt nya filer bäst placeras utifrån redan existerande arkitektur.

3. Genomförande
Efter utförd uppgift ska uppgiftens utfall testas där det är möjligt.

**Dokumentation och changelog måste hållas i synk med koden — annars uppstår
dokumentationsdrift, vilket jag vill undvika till varje pris.** Innan uppgiften kan anses
vara slutförd MÅSTE du därför:

- **Uppdatera relevant projektdokumentation** när du ändrat något som dokumentationen
  berör (arkitektur, API-routes, UI-regler, sökvägar, deploy-flöde, miljövariabler m.m.).
  Ändras verkligheten men inte dokumentationen har du skapat drift — det räknas som en
  ofullständig uppgift.
- **Uppdatera changelog-dokumentet** (t.ex. `docs/CHANGELOG.md`, eller projektets
  motsvarande ändringslogg) med en tydlig notering om vad som ändrats, när det är av
  relevans för projektet. Finns ingen changelog men förändringen är av den karaktären att
  den borde loggas — påtala det för mig.

Om du under uppgiftens gång skulle stöta på något som liknar en bugg, en onödig fil, onödig kod, dubbelroutes/dubbla uppsättning funktioner för samma syfte eller dylikt vill jag att du rapporterar detta till mig omedelbart.

## Terminal & shell (agenten har stående tillåtelse)

Du har **stående tillåtelse att köra alla terminal-/shell-kommandon som är relevanta för
att lösa uppgiften** — bygga, testa, installera beroenden, köra scripts, git-operationer
osv. Du behöver inte fråga om lov för varje enskilt kommando. Använd omdöme: destruktiva
eller svåråterställbara åtgärder (t.ex. radering av data, `git push --force`, ändringar mot
produktion) bekräftar du med mig först.

## Server & deploy (agenten har stående tillåtelse)

Jag (Peter) har SSH-access till produktionsservern via **nyckel som redan finns på datorn —
inget lösenord behövs**. Du som agent har **stående tillåtelse att köra server-kommandon
som hör till uppgiften** (SSH in, deploy m.m.) — du behöver inte fråga varje gång. De
konkreta serveruppgifterna (adress, sökvägar, deploy-kommandon) för det aktuella projektet
finns i projektets egen dokumentation (`.claude/` / `docs/`) eller i `password.md` i
LOKALT-roten.

⚠️ **Gotcha (viktig):** Claude Codes auto-läge-**permissionsklassificerare** blockerar
`ssh`/`scp`/`curl`-mot-servrar (och t.o.m. redigering av settings-filen) OAVSETT tillåtelse
i chatten. Den honorerar bara allow-regler i projektets `.claude/settings.local.json`
(`Bash(ssh:*)`, `Bash(scp:*)`, `Bash(curl:*)`) eller ett öppnare permission-läge. Är dessa
inte på plats kan agenten inte deploya — då är det spärren, inte brist på tillåtelse.
