# Test 1 — Jde z Claude něco do editoru?

> **Pro uživatele:** Nejkratší možný test. Ověří, že řetěz Claude → most → EasyEDA funguje, a nic nenavrhuje. Spotřeba je minimální.
>
> Než začneš: most musí běžet a v EasyEDA musí být **otevřený nějaký projekt s otevřeným listem schématu**. Stačí prázdný.
>
> Pak napiš: *„Přečti easyeda-test-1-spojeni.md a proveď to."*

---

## Co má agent udělat

Projdi body popořadě. Po každém napiš výsledek. Když některý selže, **zastav se** a přejdi na diagnostiku dole.

### 1. Žije most?

```powershell
curl.exe http://localhost:49620/health
```

Čekáš JSON s `"edaConnected":true`. Když se neozve nic, most neběží. Když je tam `false`, rozšíření v EasyEDA není připojené.

Most si bere první volný port z rozsahu 49620–49629. Když 49620 mlčí a víš, že běží, zkus další porty nebo se podívej do jeho okna, jaký port vypsal.

### 2. Vidíš dovnitř projektu?

```js
return await eda.dmt_Project.getCurrentProjectInfo();
```

Čekáš objekt s názvem projektu a seznamem boardů. Vrátí-li `null`, uživatel nemá otevřený projekt — požádej ho, ať ho otevře.

### 3. Dokážeš něco umístit?

Najdi si v knihovně libovolný rezistor a umísti ho doprostřed listu. **Jednotky schématu jsou 0,01 palce**, ne mil.

```js
const d = await eda.lib_Device.getByLcscIds('C23231');   // 68k 0603
const c = await eda.sch_PrimitiveComponent.create(d, 500, 400);
return { umisteno: !!c, id: c?.primitiveId, x: c?.x, y: c?.y };
```

Nenajde-li se to LCSC číslo, vezmi jakýkoliv jiný rezistor — na tomhle testu nezáleží který.

**Požádej uživatele, ať se podívá na plátno.** Má tam vidět rezistor. Tohle je ten vlastní test: data prošla celou cestou a projevila se na obrazovce.

### 4. Přečteš si, co jsi udělal?

```js
const all = await eda.sch_PrimitiveComponent.getAll();
return all.filter(c => c.componentType === 'part')
          .map(c => `${c.designator} @${c.x},${c.y}`);
```

Čtení zpátky je stejně důležité jako zápis. Bez něj si návrh nezkontroluješ.

### 5. Ukliď

Smaž, co jsi umístil, ať uživateli nezůstane v projektu nepořádek.

```js
const all = await eda.sch_PrimitiveComponent.getAll();
const ids = all.filter(c => c.componentType === 'part').map(c => c.primitiveId);
return await eda.sch_PrimitiveComponent.delete(ids);
```

⚠️ Tohle smaže **všechny** součástky na aktivním listu. Pouštěj to jen na prázdném testovacím projektu a napřed se uživatele zeptej.

---

## Vyhodnocení

Projde-li všech pět bodů, řetěz funguje a můžeš jít na **Test 2**.

### Když to selže

| Kde | Co to znamená |
|---|---|
| Bod 1, žádná odpověď | Most neběží nebo se jeho okno zavřelo. Spusť ho znovu. |
| Bod 1, `edaConnected:false` | Rozšíření není připojené. **Nejdřív se ujisti, že most běží, a teprve pak dej v EasyEDA API Gateway → Reconnect.** Opačné pořadí rozšíření zasekne. |
| Bod 2 vrátí `null` | Uživatel nemá otevřený projekt. |
| Bod 3 selže s hláškou o aktivaci listu | Je aktivní špatný dokument. Nech uživatele kliknout na list schématu. |
| Bod 3 projde, ale na plátně nic není | Aktivní je jiný list, než do kterého se zapsalo. |

Podrobnější diagnostika je v `easyeda-claude-setup.md`, sekce *Fáze 5*.

---

## Poznámka ke spotřebě

Tenhle test je levný, jde o pár volání. Test 2 a jakýkoliv skutečný návrh jsou **řádově dražší**. Než se do nich pustíš, přečti si varování na začátku Testu 2.
