# CLAUDE.md: context voor dit project

Portfolio-website van Mawadda, beheerd door Adnan (GitHub: `fearlesssmind`). Communiceer in het **Nederlands**.
Repo: `fearlesssmind/PortfolioWebsiteMWD`, hoofdbranch **`master`**. Domein: **mawadda-alkhorchani.nl** (DNS-zone in eigen beheer).

## Opbouw van de repo

| Map | Wat |
|---|---|
| root | Backend: .NET 10, GraphQL (HotChocolate 15), Dapper + PostgreSQL (Npgsql). `Program.cs`, `Dockerfile` |
| `UI/` | Publieke site: React + Vite |
| `UIAdmin/` | Admin-site: React + Vite, login met Google via Firebase (project `portfoliowebsite-admin-7f818`) |
| `infra/main.bicep` | Azure-infrastructuur (zie hieronder) |
| `.github/workflows/` | `deploy.yml` (Azure), `custom-domains.yml` (domeinen koppelen), `ci.yml` (build-controle op PR's) |
| `DEPLOYMENT.md` | Uitleg van de Azure-opzet |

De database heeft 4 tabellen (Experiences, Education, Projects, AboutMe). De app maakt ze bij het opstarten zelf aan (`Database/DbContext.cs`). AboutMe wordt eenmalig gevuld uit `Database/AbtMeText.json`.

## Configuratie van de backend (omgevingsvariabelen)

| Variabele | Doel |
|---|---|
| `ConnectionStrings__ConnectionString` | Npgsql-connectionstring (`appsettings.json` staat niet in de repo) |
| `Auth__AdminEmails` | Komma-gescheiden Google-adressen die mogen wijzigen. Leeg + Production = alles geweigerd; leeg + Development = elke ingelogde gebruiker |
| `Auth__FirebaseProjectId` | Standaard `portfoliowebsite-admin-7f818` |
| `Cors__AllowedOrigins__0..n` | Toegestane frontends (standaard localhost:5000/3000) |
| `Storage__ConnectionString` / `Storage__Container` | Azure Blob voor uploads. **Leeg = uploads naar `wwwroot/imguploads`** (lokale schijf) |
| `PublicApiUrl` | Vervangt `http://localhost:5142` in de AboutMe-seed |
| `ASPNETCORE_FORWARDEDHEADERS_ENABLED=true` | Nodig achter een reverse proxy die TLS afhandelt |

Frontends: het API-adres komt uit `VITE_API_URL` tijdens de build (fallback `http://localhost:5142`). De container luistert op poort **8080** (`Dockerfile`, Production, non-root).

## Wat al gedaan is (PR #1 t/m #5, allemaal gemerged)

- **Beveiliging:** alle GraphQL-mutaties en `/api/img-upload` vereisen een Firebase ID-token van een adres uit `ADMIN_EMAILS` (`Auth/AdminAuth.cs`, `[HotChocolate.Authorization.Authorize(AdminAuth.Policy)]`). De admin-frontend stuurt de token mee (`UIAdmin/src/api.js`). Voorheen kon iedereen mutaties doen.
- De eigen HTTPS-check in `Program.cs` (die achter een proxy altijd een 400 gaf) is vervangen door forwarded headers. CORS komt uit de configuratie.
- Uploads: alleen afbeeldingen (jpg/png/gif/webp/avif), max. 5 MB. Gaan naar Blob, of naar de lokale schijf als er geen Blob is ingesteld.
- AboutMe-tekst van een JSON-bestand naar de database verplaatst.
- **Verbindingslek opgelost:** DB-verbindingen werden nooit gesloten. Nu overal `using var conn = ...`.
- Csproj opgeschoond (±4700 regels met verwijzingen naar `UI/node_modules`), met `DefaultItemExcludes` voor UI/UIAdmin/infra.
- `staticwebapp.config.json` in beide `public/`-mappen (SPA-fallback).
- Build-controle (`ci.yml`) is groen: backend, beide frontends en Bicep compileren.

## Azure-poging (vastgelopen)

**Opzet:** resource group `rg-mawadda` (metadata in West Europe). Static Web Apps `swa-mawadda-ui` en `swa-mawadda-admin` (Free, East US 2). Container Apps-omgeving `cae-mawadda` en app `ca-mawadda-api` (0–1 replica, 0,25 vCPU). PostgreSQL Flexible Server B1ms (Burstable, 32 GB, regionaam in de servernaam). Storage `stmawadda…` en Log Analytics `log-mawadda` (Sweden Central). De API-image staat op GHCR: `ghcr.io/fearlesssmind/portfoliowebsite-api` (public).

**Toegang:** Entra-tenant `mezelf888.onmicrosoft.com`, abonnement "Azure subscription 1" (**Pay-As-You-Go**). App-registratie `github-mawadda-deploy` met de rol Contributor op `rg-mawadda`, login via OIDC (federated credentials). GitHub-secrets: `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`, `POSTGRES_ADMIN_PASSWORD`, `ADMIN_EMAILS` (`deploy.yml` accepteert die als secret of als variabele). Budget: €30 per maand met e-mailwaarschuwingen.

**Opgeloste problemen, onderweg:**
1. De image ging naar de verkeerde GHCR-eigenaar. De workflow leidt de eigenaar nu af van `GITHUB_REPOSITORY_OWNER`.
2. `ADMIN_EMAILS` stond als secret in plaats van als variabele. Opgelost: beide worden geaccepteerd.
3. **AADSTS700213**: nieuwe GitHub-repo's sturen een OIDC-subject *met ID's* mee: `repo:fearlesssmind@50379781/PortfolioWebsiteMWD@1458235754:ref:refs/heads/master`. Daarvoor is een extra federated credential aangemaakt.
4. **West Europe** gaf `RequestDisallowedByAzure` (locationineligible: geen nieuwe klanten). Uitgeweken naar `swedencentral`, de Static Web Apps naar `eastus2` (te wijzigen met de repo-variabelen `AZURE_LOCATION`, `SWA_LOCATION` en `POSTGRES_LOCATION`).
5. Een mislukte PostgreSQL-poging laat een "spook"-resource achter met dezelfde naam. Daarom staat de regio nu in de naam (`uniqueString(resourceGroup().id, postgresLocation)`). Spookvermeldingen in de portal geven een 404 en kosten niets.

**Niet opgelost (blokkade):** PostgreSQL B1ms geeft **`SkuNotAvailable`** in Sweden Central én North Europe (laatste poging: run #7 op 15-11-2026). `az postgres flexible-server list-skus` meldt B1ms wel als beschikbaar, maar die lijst houdt geen rekening met beperkingen per abonnement. De oorzaak is Microsofts capaciteitsbeperking voor nieuwe klanten in Europa, niet het abonnementstype. Er staat een **supportaanvraag** open: ticket **2611150050000368** (Service and subscription limits → PostgreSQL Flexible Server → Region access → Sweden Central), ingediend op 15-11-2026. Daardoor bestaan de database en de API (`ca-mawadda-api`) nog niet. De overige Azure-resources bestaan wel en kosten vrijwel niets.

## Nieuwe richting (15-11-2026)

Adnan wil de site nu **op een eigen server hosten** in plaats van op Azure. Nog niet bekend: VPS of thuis, besturingssysteem, of Docker er al op staat. Plan: Docker Compose met API + PostgreSQL + gebouwde frontends, Caddy als reverse proxy (automatische HTTPS voor `www.`, `admin.` en `api.mawadda-alkhorchani.nl`), A-records naar het server-IP, en `admin.mawadda-alkhorchani.nl` als Authorized domain in Firebase. Uploads op een volume (`wwwroot/imguploads`).

**Opruimen in Azure** (alleen als Adnan dat bevestigt): de workflow "Deploy naar Azure" uitzetten in GitHub (elke push naar `master` start hem nu nog en faalt op de database), resource group `rg-mawadda` verwijderen, en ticket 2611150050000368 sluiten of laten lopen.

## Werkafspraken

- Wijzig `master` via een branch + pull request, niet rechtstreeks.
- De `POSTGRES_ADMIN_PASSWORD` mag geen `;`, `'` of `"` bevatten (die breken de connectionstring).
- De database begint leeg: content moet opnieuw worden ingevoerd via de admin-site, of overgezet met `pg_dump`.
- Lokaal: `run-dev.bat` of `docker-compose.yaml` (API op 5142→8080, `ASPNETCORE_ENVIRONMENT=Development`).
