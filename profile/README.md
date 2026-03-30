# Vårddatahubbens paketdistribution — teknisk dokumentation

**Organisation:** Kompetenscentrum Hälsodata (KCHD), SKR
**Plattform:** GitHub, organisation `kchd-se`
**Adress:** https://github.com/kchd-se
**Datum:** 2026-03-30
**Ansvarig:** Peder Hofman-Bang, samordnare KCHD

---

## 1. Översikt

Vårddatahubben (KCHD) distribuerar standardiserade beräkningspaket till Sveriges 21 hälso- och sjukvårdsregioner via GitHub. Hubben levererar kod och specifikationer, men kan även komma att leverera visualiseringar, benchmarks och analyser i framtiden. Regionerna kör koden mot sina egna datakällor i sina egna miljöer.

Varje paket är ett eget Git-repo under organisationen `kchd-se`. Alla paket följer samma struktur, samma versionshantering och samma branch-modell.

### Nuvarande paket

| Repo | Beskrivning | Version | Innehåll |
|------|-------------|---------|----------|
| `vantetid-katarakt` | Väntetidsberäkning katarakt (gråstarr) | 1.1.0 | VQL (Denodo) |
| `vantetid-katarakt-fhir` | FHIR-transformering av katarakt-resultat | 1.0.0 | VQL + C# (Firely SDK) |

FHIR-paketet har ett beroende: det förutsätter att resultatvyerna från `vantetid-katarakt` redan finns i regionens Denodo-miljö.

### Arbetsyta vs distributionspaket

Utveckling och experiment sker i ett separat repo: `pederhofmanbang/POC_KCHD`. Där finns Python-pipelines, testdata, demo-visualiseringar och annat som regioner inte behöver. När kod är testad och verifierad kopieras den till rätt distributionsrepo under `kchd-se/`.

Regioner interagerar bara med repon under `kchd-se/` — de behöver aldrig se arbetsytan.

---

## 2. Repostruktur

Varje repo följer samma grundmönster, anpassat efter pakettypen.

### vantetid-katarakt (VQL-paket)

```
vantetid-katarakt/
├── CLAUDE.md              Instruktioner för Claude Code (läses automatiskt)
├── README.md              Beskrivning, snabbstart, regiontabell
├── CHANGELOG.md           Versionshistorik (Keep a Changelog-format)
├── manifest.json          Maskinläsbar version, paketinfo, regionstatus
├── vql/
│   ├── 02_berakning/      Beräkningslogik — identisk alla regioner
│   ├── 04_verifiering/    Kontroller av beräkningsresultat
│   └── 05_kvalitet/       Kontroller av indatakvalitet
└── tests/                 SQL-testfrågor
```

### vantetid-katarakt-fhir (C# + VQL-paket)

```
vantetid-katarakt-fhir/
├── CLAUDE.md
├── README.md
├── CHANGELOG.md
├── manifest.json
├── vql/                   VQL-vyer för FHIR-mappning
├── csharp/
│   └── KchdFhirSerializer/   C#-projekt med Firely SDK
└── docs/                  FHIR-dokumentation
```

### Gemensamma filer i alla repon

| Fil | Syfte |
|-----|-------|
| `CLAUDE.md` | Universella instruktioner som Claude Code läser automatiskt — versionsregler, branch-modell, namnkonventioner. Identisk i alla repon. |
| `README.md` | Det första en region ser. Beskriver paketet, hur man kommer igång, vilka regioner som kör vilken version. |
| `CHANGELOG.md` | Alla ändringar dokumenterade per version, i kategorierna Tillagt/Ändrat/Fixat/Borttaget. |
| `manifest.json` | Maskinläsbar metadata: paketnamn, version, datum, pakettyp, beroenden, och vilken version varje region kör. |

---

## 3. Branch-modell

Varje repo har (minst) två typer av grenar:

### main — generisk mall

`main`-grenen innehåller den generiska versionen av paketet. I VQL-paket har basvyerna (`01_basvy/`) platshållarkolumner med kommentarer som förklarar vad regionen ska anpassa. En ny region utgår alltid från `main`.

### region/XXX — regionspecifik variant

Varje region som implementerar paketet får en egen gren, t.ex. `region/vgr`. Den enda skillnaden mot `main` är de regionanpassade basvyerna — all beräkningslogik, verifiering och kvalitetskontroll är identisk.

### Nuvarande grenar

| Repo | Gren | Beskrivning |
|------|------|-------------|
| `vantetid-katarakt` | `main` | Generisk mall |
| `vantetid-katarakt` | `region/vgr` | VGR:s anpassning mot sina källsystem |
| `vantetid-katarakt-fhir` | `main` | Generisk version |

### Så flödar ändringar

Buggfixar och ny funktionalitet görs alltid i `main` först. Sedan mergeas ändringen till berörda regiongrenar:

```
main (fix bugg) → region/vgr (merge main)
                → region/halland (merge main)
                → ...
```

Regionspecifika ändringar (t.ex. ett kolumnnamn i VGR:s basvy) görs direkt i regionens gren och påverkar inte `main` eller andra regioner.

---

## 4. Versionshantering

Alla paket följer Semantic Versioning (SemVer): `MAJOR.MINOR.PATCH`.

### Vad siffrorna betyder

| Ändring | Exempel | Vad hände | Regionen gör |
|---------|---------|-----------|--------------|
| PATCH | 1.1.0 → 1.1.1 | Buggfix i beräkningslogik | Uppgradera gärna. Inga ändringar krävs hos regionen. |
| MINOR | 1.1.x → 1.2.0 | Ny vy, ny kolumn, ny testfil | Valfritt att uppgradera. Inga ändringar krävs. |
| MAJOR | 1.x.x → 2.0.0 | Kolumn bytt namn eller borttagen i basvy | Regionen MÅSTE uppdatera sin basvy. Instruktioner medföljer i CHANGELOG. |

### Vad som uppdateras vid varje release

1. `CHANGELOG.md` — beskrivning av ändringen under rätt version och kategori
2. `manifest.json` — nytt versionsnummer och datum
3. Git commit med meddelande på formatet: `v1.1.1: Kort beskrivning`
4. Git tag: `v1.1.1`
5. GitHub Release med release notes (kopierat från CHANGELOG)

### Regeln som aldrig bryts

Versionsnummer bestäms alltid av KCHD — aldrig automatiskt av verktyg eller AI. Om ingen version anges i en prompt ska Claude Code fråga istället för att välja själv.

---

## 5. Hur en region hämtar och använder ett paket

### Steg 1: Klona

```bash
git clone https://github.com/kchd-se/vantetid-katarakt.git
```

Alternativt: ladda ner som ZIP via "Code" → "Download ZIP" på GitHub.

### Steg 2: Anpassa basvyer (VQL-paket)

Öppna filerna i `vql/01_basvy/` (om paketet har sådana) och ändra kolumnnamnen så de pekar mot regionens datakälla. Aliasnamnen (det som står efter AS) får inte ändras — beräkningsvyerna förväntar exakt dessa.

### Steg 3: Installera

Kör VQL-filerna i Denodo i nummerordning.

### Steg 4: Verifiera

Kör testfrågorna i `tests/`. Jämför resultat mot vantetider.se.

### Steg 5: Rapportera till KCHD

Meddela KCHD vilken version som driftsatts, resultat av verifiering, och eventuella avvikelser. KCHD uppdaterar `manifest.json` med regionens status.

---

## 6. Hur en region håller sig uppdaterad

### Prenumeration via GitHub Watch

1. Gå till repot på GitHub
2. Klicka "Watch" → "Custom" → bocka i "Releases"
3. Regionen får mejl vid varje ny version

### RSS-flöde

```
https://github.com/kchd-se/vantetid-katarakt/releases.atom
```

### Uppgradering

Om regionen klonade med Git:

```bash
cd vantetid-katarakt
git pull
```

Basvyerna i regionens gren påverkas inte vid PATCH- eller MINOR-uppgraderingar.

Vid MAJOR-uppgradering framgår det i CHANGELOG.md exakt vad som behöver ändras.

---

## 7. Regionspårning

`manifest.json` i varje repo innehåller ett `regions`-objekt som visar vilken version varje region kör:

```json
{
  "regions": {
    "vgr": {
      "name": "Västra Götalandsregionen",
      "deployed_version": "1.1.0",
      "deployed_date": "2026-03",
      "status": "verifiering",
      "platform": "Denodo 8.0"
    }
  }
}
```

Status kan vara: `planerad`, `installation`, `verifiering`, `driftsatt`.

Regionen meddelar KCHD vid driftsättning och uppgradering. KCHD uppdaterar manifest.json.

---

## 8. GitHub Releases

Varje officiell version publiceras som en GitHub Release med:

- Git-tagg (t.ex. `v1.1.0`)
- Titel (t.ex. "v1.1.0 — Verifiering och kvalitetskontroll")
- Release notes (kopierade från CHANGELOG.md)
- Eventuella binärer (t.ex. FHIR-paketets kompilerade C#-serialiserare)

Exempel: `vantetid-katarakt-fhir` har release v1.0.0 med en self-contained Windows x64-binär av FHIR-serialiseraren.

Regioner som "watchar" repot får automatiskt mejl vid ny release.

---

## 9. CLAUDE.md — automatisk styrning av Claude Code

Varje repo innehåller en fil `CLAUDE.md` i roten. Claude Code läser denna automatiskt vid varje session. Filen innehåller:

- Versionshanteringsregler (SemVer, CHANGELOG-format, tagg-rutin)
- Branch-modellens regler (main = generisk, region/XXX = specifik)
- Namnkonventioner för filer
- Regel att aldrig ändra version på eget initiativ
- Kvalitetschecklista innan release
- Att alias i resultatvyer är kontrakt som inte får ändras utan MAJOR-version

Filen är identisk i alla paketrepon. Det innebär att KCHD aldrig behöver förklara versionshantering, branch-modell eller namnkonventioner — Claude Code följer dem automatiskt.

---

## 10. Namnkonventioner

### Reponamn

Format: `vantetid-{vardomrade}` för beräkningspaket, med suffix `-fhir` för FHIR-transformering.

Exempel: `vantetid-katarakt`, `vantetid-katarakt-fhir`, `vantetid-hoft` (framtida).

### VQL-filer

| Prefix | Mapp | Beskrivning |
|--------|------|-------------|
| `ref_*` | `00_kodverk` | Referensdata och kodverk |
| `src_*` | `01_basvy` | Basvyer (det enda regionen anpassar) |
| `calc_*` | `02_berakning` | Beräkningslogik |
| `res_*` | `03_resultatvy` | Resultatvyer för BI/rapport |
| `dq_*` | `05_kvalitet` | Kvalitetskontroller |

### Grenar

- `main` — alltid den generiska mallen
- `region/{regionkod}` — regionspecifik variant (t.ex. `region/vgr`, `region/halland`)

---

## 11. Framtida utveckling

- **GitLab self-managed på svensk infrastruktur** — på sikt ersätter GitHub för att uppfylla eSam-krav och eliminera Schrems II-risk
- **Harbor OCI-registry** — för distribution av containeriserade verktyg (ETL-motorer, FHIR-validatorer)
- **Automatisk regionspårning** — dashboard som läser manifest.json från alla repon
- **Fler paket** — samma struktur och modell för nya vårdområden (höft, knä, etc.)
- **Signerade releaser** — Cosign-signaturer och SBOM för verifierbarhet
