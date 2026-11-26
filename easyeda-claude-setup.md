# Zprovoznění EasyEDA Pro + Claude Code

> **Pro uživatele:** Ulož tenhle soubor do prázdné složky, otevři v ní terminál, spusť `claude` a napiš:
> *„Přečti easyeda-claude-setup.md a proveď to. Po každé fázi mi napiš výsledek a počkej, až ti řeknu dál."*
>
> Budeš potřebovat: Windows (postup je psaný pro PowerShell), [EasyEDA Pro desktop](https://easyeda.com/page/download) a licenci Claude Code. Zbytek doinstaluje Claude.

---

## Pravidla pro agenta — přečti dřív, než začneš

1. **Nedůvěřuj tomuhle dokumentu, ověřuj.** Je psaný podle stavu z listopadu 2026. Všechno, co tu je uvedené jako fakt o cizím repozitáři, verzi nebo API, si napřed ověř v realitě. Když se realita liší, **řiď se realitou a rozdíl nahlas uživateli** — na konci je sekce, jak to zapsat.
2. **Před každou instalací softwaru** napiš, co instaluješ a proč, a počkej na souhlas.
3. **Nejdřív zjisti, co už je.** Nic nepřeinstalovávej zbytečně.
4. **Most poslouchá jen na localhostu a nemá autentizaci.** Spouští v editoru libovolný JavaScript. Nikdy neměň bind adresu, neotvírej port ve firewallu a nenavrhuj to.
5. **Kroky v GUI EasyEDA dělá uživatel.** Napiš mu je přesně a počkej na potvrzení.
6. **Když něco selže, diagnostikuj.** Neopakuj stejný příkaz znovu a znovu.
7. **Nikdy nenapiš, že je návrh hotový k výrobě.** Výrobní data až po kontrole člověkem.

---

## Fáze 1 — Prostředí

```powershell
node --version      # potřeba 18+
git --version
```

Chybí-li něco, nabídni `winget install --id OpenJS.NodeJS.LTS -e` nebo `--id Git.Git -e`.

Ověř, že je nainstalovaný **desktopový** klient EasyEDA Pro (typicky `%LOCALAPPDATA%\Programs\easyeda-pro`), ne webová Standard verze. Když chybí, pošli uživatele na <https://easyeda.com/page/download> a počkej.

Vypiš tabulku *komponenta / verze / OK?* a počkej na pokyn.

---

## Fáze 2 — API skill

```powershell
mkdir tools -Force; cd tools
git clone https://github.com/easyeda/easyeda-api-skill.git
cd easyeda-api-skill
npm install
```

**Teď si ověř, co v repozitáři doopravdy je** — struktura se mění a README za ní zaostává:

```powershell
npm pkg get scripts          # jaké skripty existují
Get-ChildItem                # jaké složky tam jsou
```

Rozhodni se podle toho, co vidíš:

- Je-li ve `scripts` build dokumentace (např. `build:docs`), spusť ho.
- **Není-li tam** a existuje složka s dokumentací (v listopadu 2026 to byla `references/`), je dokumentace už commitnutá a žádný build není potřeba.
- Ověř, kde je `SKILL.md` a na jaké cesty odkazuje. Tam, kde je platný `SKILL.md` i jeho dokumentace, míří junction v další fázi.

Registrace skillu (junction, aby se aktualizoval s repozitářem):

```powershell
New-Item -ItemType Junction -Path "$env:USERPROFILE\.claude\skills\easyeda-api" -Target "<absolutní cesta ke kořeni repa>"
```

Skill se načte při příštím startu Claude Code.

---

## Fáze 3 — Most

Spusť ho v **samostatném okně**, ne jako background task:

```powershell
Start-Process cmd -ArgumentList '/k','cd /d <cesta>\easyeda-api-skill && npm run server'
```

Najdi v repozitáři skript serveru (v listopadu 2026 `scripts/bridge-server.mjs`) a **ověř v jeho zdrojáku**, na jakou adresu a port se váže. Čekej `127.0.0.1` a rozsah portů kolem 49620–49629. Kdyby tam bylo cokoliv jiného než localhost, zastav se a řekni to uživateli.

Vytvoř uživateli `start-bridge.cmd`, aby to příště spustil dvojklikem.

---

## Fáze 4 — Rozšíření v EasyEDA (dělá uživatel)

**Nejdřív si ověř aktuální verzi.** Jsou dva mirrory a liší se:

- <https://ext.lceda.cn/item/oshwhub/run-api-gateway> — bývá aktuálnější
- <https://jlcext.com/item/oshwhub-official/run-api-gateway> — v listopadu 2026 zaostával o tři verze

Porovnej, co ukazují, a doporuč uživateli novější. **Nejčistší cesta je ale instalace z online záložky Extension Manageru přímo v klientovi** — ten si vytáhne verzi i jazykovou variantu, která k němu patří.

Napiš uživateli tyhle kroky a počkej na potvrzení:

1. **Advanced → Extension Manager**, najít *Run API Gateway* a nainstalovat (nebo *Import* staženého `.eext`)
2. Přepnout na seznam **Installed**, **kliknout na to rozšíření** a zapnout mu **oprávnění externí interakce** („Allow interact with external" / „External Interactions"). Je to až po kliknutí na položku, ne v importním dialogu. **Tohle lidi míjí nejčastěji.**
3. Volitelně zapnout zobrazení v horním menu
4. **API Gateway → Reconnect**

---

## Fáze 5 — Ověření spojení

```powershell
curl.exe http://localhost:49620/health
```

Hledáš `"edaConnected":true`. Když 49620 neodpovídá, zjisti skutečný port z banneru v okně mostu.

### Když se to nespojí

Postupuj v tomhle pořadí, dál jdi až když předchozí bod sedí:

| Co ověřit | Jak |
|---|---|
| Běží most vůbec? | `curl` na `/health` musí vrátit JSON. Když nevrátí nic, most spadl nebo se okno zavřelo. |
| Zkouší se rozšíření připojit? | Podívej se do **logu mostu**. Loguje každou příchozí konexi. **Prázdný log = rozšíření nekomunikuje vůbec**, nehledej port ani firewall. |
| Má rozšíření oprávnění? | Fáze 4, krok 2. Bez něj mu editor WebSocket nepustí. |
| Nedal uživatel Reconnect do mrtvého mostu? | Rozšíření se po marných pokusech zastaví a samo se nerozjede. **Vždy nejdřív most, teprve pak Reconnect.** |

**Slepá ulička:** `engines` v manifestu rozšíření může hlásit starší verzi EDA, než má klient. Není to problém — verze klienta je verze Electronového obalu, zatímco `engines.eda` se vztahuje k editoru načítanému z webu.

Testovací volání:

```powershell
curl.exe -X POST http://localhost:49620/execute -H "Content-Type: application/json" -d "{\"code\": \"return await eda.dmt_Project.getCurrentProjectInfo();\"}"
```

---

## Fáze 6 — První projekt

Projekt se dá založit i přes API (`dmt_Project.createProject()` vrací uuid a vytvoří Board se schématem i PCB). Ověřeno v listopadu 2026 — starší návody tvrdí, že to nefunguje. **Nech přesto uživatele rozhodnout**, jestli chce založit projekt sám v GUI; je to jeho pracovní prostor.

Navrhni spolu s ním něco malého — 5 až 10 součástek, např. LDO nebo step-down 5 V → 3,3 V s LED a dvěma konektory. Postupuj takhle a po každém kroku ukaž výsledek:

1. **Zadání** do `SPEC.md`. Ověř, že si požadavky neodporují (typicky proud versus dostupnost součástek), **dřív než začneš počítat**.
2. **Výpočty** do `DESIGN.md`. Hodnoty z datasheetu, ne z paměti. Co se nepodařilo ověřit, označ ⚠️.
3. **Součástky**: LCSC čísla a třída dílku (Basic/Extended).
4. **Schéma** → ERC → **přečíst netlist a zkontrolovat ho proti záměru**.
5. **PCB**: převod, rozmístění, routing, DRC.
6. Otevřená rizika do `REVIEW.md`.

### Co si pohlídat

- **Jednotky se liší.** PCB 1 mil, schéma 0,01 palce (10 mil). Záměna posune součástku desetkrát.
- **Vyrenderuj si schéma a podívej se na něj.** Kreslení poslepu stojí zbytečná kola. Hledej v API metodu na render oblasti a vyzkoušej si ji hned na začátku.
- **Netlist je kontrola, ne formalita.** Vodič vedený přes pin ho spojí, i když to na obrázku vypadá, že jen prochází kolem. Křížení dvou vodičů bez junction dotu naopak v pořádku je.
- **DRC chytá věci, které z obrázku vidět nejsou** — třeba součástku naimportovanou obráceně, jejíž pady si sednou na nulovou vzdálenost.
- **Smyčky ti nenahlásí nikdo.** Vzdálenost kondenzátoru od pouzdra změř **a vyhodnoť**. „Je to objemový, to stačí" není vyhodnocení.
- **Požádej uživatele o vizuální kontrolu a 3D náhled.** Mechanické kolize, výšky a konektory na hraně z API nevidíš.

---

## Pasti v API

Tohle se nedá odvodit z dokumentace. Když narazíš na jiné chování, je to změna verze — zapiš to podle poslední sekce.

| Past | Co s tím |
|---|---|
| **Přepočet je asynchronní** | Netlist i seznamy součástek vrací po změně prázdno nebo neúplná data. Stejné volání o chvíli později vrátí správná. **Nikdy nečti stav ve stejném skriptu, který ho mění.** |
| **Část funkcí vrací `undefined` i při úspěchu** | Např. vytvoření net labelu. Úspěch ověřuj přes netlist, ne podle návratové hodnoty. |
| **Úseky cest mají prohozené konce** | Mazání filtrované podle počátečního bodu část úseků mine. Filtruj na obě orientace. |
| **Rozlití mědi je po vytvoření prázdné** | Dokud nezavoláš přestavbu měděných oblastí, DRC hlásí všechny zemní pady jako nezapojené. |
| **Některé funkce nefungují na všech vrstvách** | Kreslení čar selhávalo na vrstvě obrysu desky, na měděných fungovalo. Když funkce hází chybu, zkus jinou primitivu, než ji prohlásíš za rozbitou. |
| **Dokumentace nemusí uvádět povinné parametry** | Když volání vrátí `PARAM_REQUIRED`, chybí něco, co v docs není. Přečti si text chyby, bývá v ní co. |

Když něco nefunguje podle dokumentace, **introspekce je rychlejší než hádání**:

```js
Object.keys(eda)                         // jaké jmenné prostory existují
Object.getOwnPropertyNames(
  Object.getPrototypeOf(eda.sch_Document))   // jaké metody má třída
```

---

## Volitelně — Freerouting

API vlastní autorouter nemá, routuje se přes Freerouting (export DSN → routing → import SES).

Stáhni poslední release z <https://github.com/freerouting/freerouting/releases>. Nabízejí se dvě varianty:

- **nativní CLI build pro Windows** — menší, nepotřebuje Javu, pro dávkové routování ideální
- **`.jar`** — potřebuje Javu 21 (`winget install --id EclipseAdoptium.Temurin.21.JDK -e`), umí GUI

Ukaž uživateli název a velikost souboru a počkej na souhlas. **Ověř kontrolní součet** proti `checksums` souboru v releasu, pokud tam je.

U malých desek bývá ruční routování kritických cest rychlejší a máš nad nimi kontrolu.

---

## Údržba tohohle dokumentu

Tenhle návod zastará. Když při provádění narazíš na rozdíl mezi tím, co je tu napsané, a realitou:

1. **Řiď se realitou**, ne dokumentem.
2. Zapiš rozdíl do souboru `ODCHYLKY.md` ve složce, odkud jsi návod načetl: co dokument tvrdí, co jsi našel, a datum.
3. Na konci práce uživateli shrň, kolik odchylek bylo a čeho se týkaly.
4. **Jsou-li víc než tři, nebo selhala-li kvůli nim některá fáze**, nabídni, že návod přepíšeš podle aktuálního stavu.

Zdroje, jejichž aktuálnost se vyplatí ověřit při každém běhu:

| Zdroj | Co ověřit |
|---|---|
| <https://github.com/easyeda/easyeda-api-skill> | struktura repozitáře a skripty v `package.json` |
| <https://ext.lceda.cn/item/oshwhub/run-api-gateway> | verze rozšíření proti druhému mirroru |
| <https://github.com/freerouting/freerouting/releases> | poslední release a dostupné varianty |

---

*Napsáno podle reálného běhu v listopadu 2026 na Windows 11, EasyEDA Pro 4.1.105, skill 1.1.86. Co se od té doby změnilo, zjistíš provedením.*
