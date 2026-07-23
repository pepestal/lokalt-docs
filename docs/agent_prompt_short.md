Dokumentation för projektet finns i respektive rotmapps `.agents/`, `.claude/` och
`docs/`. Ett projekt kan ha flera rotmappar (t.ex. en för frontend och en för backend) —
leta i var och en. Finns en dokumentationskarta eller ett index (t.ex. `docs/README.md`)
— börja där.

Projektet kan vara live på en publik adress. Krävs inloggningsuppgifter för att komma åt
sidan finns de i `password.md` i LOKALT-roten (gitignorad — finns bara lokalt och checkas
aldrig in).

## Dokumentation & changelog (måste hållas i synk — undvik drift)

Innan en uppgift är slutförd MÅSTE relevant projektdokumentation uppdateras när något
ändrats som den berör, och changelog-dokumentet (`docs/CHANGELOG.md` el. motsvarande)
noteras när ändringen är av relevans. Ändras verkligheten men inte dokumentationen uppstår
dokumentationsdrift — det räknas som en ofullständig uppgift.

## Terminal & shell (agenten har stående tillåtelse)

Du har stående tillåtelse att köra alla terminal-/shell-kommandon som är relevanta för
uppgiften (bygga, testa, installera, scripts, git m.m.) — fråga inte per kommando.
Destruktiva/svåråterställbara åtgärder bekräftas dock med mig först.

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
