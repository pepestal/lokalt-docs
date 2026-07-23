Dokumentation för projektet finns på följande ställen:
För frontend (root mapp signal_frontend):
.agents/
.claude/
docs/

För backend (root mapp backend):
.agents/
.claude/
docs/

Projektet är live på adressen: https://signal.syntes.dev/
För att du ska komma åt sidan krävs inloggningsuppgifter. De finns i `password.md`
i LOKALT-roten (gitignorad — finns bara lokalt och checkas aldrig in).

## Server & deploy (agenten har stående tillåtelse)

Jag (Peter) har SSH-access till produktions-VPS:en och du som agent har **stående
tillåtelse att köra server-kommandon som hör till uppgiften** (deploy m.m.) — du behöver
inte fråga varje gång.

**VPS:** `root@65.109.143.130` (nyckelbaserad, inget lösenord). Allt körs i **Docker**
bakom **Caddy** (:443) + **Authelia**-login (därför ger `/api/...` `302` utifrån). Signal
ligger i `/root/apps/signal/{frontend,backend}`.

**Deploya frontend:**
```bash
ssh root@65.109.143.130
cd /root/apps/signal/frontend && git pull --ff-only && docker compose up -d --build
# verifiera: docker logs signal-frontend --tail 12   (ska visa: ✓ Ready)
```
Backend deployas analogt i `/root/apps/signal/backend` (egen docker-compose.yml).

⚠️ **Gotcha (viktig):** Claude Codes auto-läge-**permissionsklassificerare** blockerar
`ssh`/`scp`/`curl`-mot-servrar (och t.o.m. redigering av settings-filen) OAVSETT tillåtelse
i chatten. Den honorerar bara allow-regler i `.claude/settings.local.json`
(`Bash(ssh:*)`, `Bash(scp:*)`, `Bash(curl:*)`) eller ett öppnare permission-läge. Är dessa
inte på plats kan agenten inte deploya — då är det spärren, inte brist på tillåtelse.


