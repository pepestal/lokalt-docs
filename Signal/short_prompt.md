Dokumentation för projektet finns i respektive rotmapps `.agents/`, `.claude/` och
`docs/`. Ett projekt kan ha flera rotmappar (t.ex. en för frontend och en för backend) —
leta i var och en.

Projektet kan vara live på en publik adress. Krävs inloggningsuppgifter för att komma åt
sidan finns de i `password.md` i LOKALT-roten (gitignorad — finns bara lokalt och checkas
aldrig in).

## Server & deploy (agenten har stående tillåtelse)

Jag (Peter) har SSH-access till produktionsservern och du som agent har **stående
tillåtelse att köra server-kommandon som hör till uppgiften** (deploy m.m.) — du behöver
inte fråga varje gång. De konkreta serveruppgifterna (adress, sökvägar, deploy-kommandon)
för det aktuella projektet finns i projektets egen dokumentation (`.claude/` / `docs/`)
eller i `password.md` i LOKALT-roten.

⚠️ **Gotcha (viktig):** Claude Codes auto-läge-**permissionsklassificerare** blockerar
`ssh`/`scp`/`curl`-mot-servrar (och t.o.m. redigering av settings-filen) OAVSETT tillåtelse
i chatten. Den honorerar bara allow-regler i projektets `.claude/settings.local.json`
(`Bash(ssh:*)`, `Bash(scp:*)`, `Bash(curl:*)`) eller ett öppnare permission-läge. Är dessa
inte på plats kan agenten inte deploya — då är det spärren, inte brist på tillåtelse.
