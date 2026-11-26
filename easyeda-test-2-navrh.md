# Test 2 — Celý návrh na hotovém zadání

> **Pro uživatele:** Ověří celou cestu od schématu po čisté DRC. Zadání je už hotové, takže se nic nevymýšlí ani nerešeršuje — agent jen provede návrh.
>
> Než začneš: projdi **Test 1**. A přečti si varování o spotřebě hned pod tímhle odstavcem.
>
> Pak v EasyEDA **ručně založ prázdný projekt**, otevři v něm list schématu a napiš:
> *„Přečti easyeda-test-2-navrh.md a proveď to. Po každé fázi se zastav."*

> [!WARNING]
> **Tenhle test spotřebuje znatelné množství tokenů.** Je to kompletní návrh desky, ne dotaz. U velkých nebo opakovaných pokusů můžeš vyčerpat denní nebo týdenní limit. Pouštěj to, když víš, že limit zrovna nepotřebuješ na něco jiného.

---

## Pravidla pro agenta

1. **Nerešerši.** Zadání i součástky jsou dané níž. Nehledej alternativy, nepočítej topologii znovu, nenavrhuj vylepšení. Cílem je ověřit nástroj, ne odladit desku.
2. **Po každé fázi se zastav** a ukaž výsledek.
3. Co se nepodařilo ověřit, označ ⚠️ a zapiš do `REVIEW.md`.
4. **Nikdy nenapiš, že je deska hotová k výrobě.** Není a nebude.

---

## Zadání

Snižující měnič **5 V → 3,3 V**, zátěž do 1 A, napájení i výstup přes pinheadery 2,54 mm, indikační LED na výstupu. Dvouvrstvá deska, rozměr si zvol podle rozmístění — samotný induktor má kolem 10 mm, ověřený běh skončil na zhruba **32 × 15 mm**.

### Osazení

| Ozn. | Součástka | LCSC | Pouzdro |
|---|---|---|---|
| U1 | Silergy SY8089AAAC, synchronní buck | `C78988` | SOT-23-5 |
| L1 | 4,7 µH, Isat ≥ 1,5 A, **DCR < 50 mΩ** | — | vyber z knihovny |
| C1, C3 | 22 µF X5R 25 V | `C45783` | 0805 |
| C2 | 100 nF X7R | `C1525` | 0402 |
| R1 | 68 kΩ 1 % | `C23231` | 0603 |
| R2 | 15 kΩ 1 % | `C22809` | 0603 |
| R3 | 680 Ω 1 % | `C23228` | 0603 |
| D1 | LED červená | `C2286` | 0603 |
| J1, J2 | pinheader 1×2, 2,54 mm | `C492401` | THT |

### Zapojení

```
J1.1 ─┬─ C1 ─┬─ C2 ─┬─ U1.IN(4)        U1.LX(3) ── L1 ─┬─ C3 ─┬─ R1 ─┬─ R2 ─ GND
      │      │      │                                  │      │      │
      └──────┴──────┴─ U1.EN(1)                        │      │      └─ U1.FB(5)
                                                       │      │
GND: U1.GND(2), C1, C2, C3, R2, D1.K, J1.2, J2.2       └─ J2.1└─ R3 ─ D1.A
```

Nety: `VIN`, `LX`, `3V3`, `FB`, `LED_A`, `GND`. Dělič 68k/15k dává `Vout = 0,6 × (1 + 68/15) = 3,32 V`.

### Když LCSC číslo neexistuje

Katalog se mění. Nenajdeš-li dílek, vezmi **jiný se stejnou hodnotou, pouzdrem a třídou** a napiš, co jsi zaměnil. U induktoru drž podmínku `DCR < 50 mΩ`, tu žádá datasheet regulátoru.

**Pozor:** `getByLcscIds()` může vrátit obecný dílek bez výrobce. Podle pole `manufacturerId` se to **nepozná** — bývá prázdné i u správných dílků. Spolehlivý signál je až **ERC: varování `ATTRMATCHCODE`** u té součástky. Dostaneš-li ho, dohledej dílek podle jména výrobní řady přes `lib_Device.search()` a součástku přeosaď.

---

## Fáze A — Schéma

Umísti součástky a propoj je. Jednotky schématu jsou **0,01 palce** (10 mil).

Jak kreslit, aby to byl výkres a ne jen nosič netlistu:

- Hlavní signálová cesta na **jedné vodorovné řadě**, všechny zemní symboly na **jedné společné řadě** pod ní.
- **Zpětnou vazbu kresli vodičem**, klidně oklikou pod obvodem. Je to funkčně zajímavý net a vykreslená smyčka ukazuje, jak se obvod reguluje.
- **Enable pin připoj štítkem** co nejblíž pouzdru. Je natrvalo na napájení, nikdo ho neměří ani neovládá, vodič přes půl schématu by jen matl.
- Vstupní a výstupní konektor na **stejné vodorovné ose**.

### Kontrola fáze A

1. **Vyrenderuj si schéma a podívej se na něj.** Nehádej ze souřadnic. Hledej v API metodu na render oblasti; vrací obrázek.
2. **Přečti netlist** a porovnej ho se zapojením výš. Každý net musí mít přesně ty piny, co má mít.
3. Spusť **ERC**. Má být bez chyb.
4. Zapiš netlist do `CONNECTIONS.md`.

> [!CAUTION]
> Vodič vedený **přes pin** se s ním spojí, i když to na obrázku vypadá, že jen prochází kolem. Křížení dvou vodičů bez junction dotu naopak v pořádku je. Tohle odhalí jedině netlist, ne pohled na obrázek.

---

## Fáze B — PCB

Převeď schéma do PCB, vytvoř obrys desky a rozmísti footprinty. Jednotky PCB jsou **mil**.

> [!CAUTION]
> **Import ze schématu vrací `true`, i když nic neimportoval.** Stalo se to v ověřeném běhu: zavolal jsem ho, dostal `true` a na desce bylo nula součástek. Pomohlo až znovu uložit schéma a import zopakovat. **Nevěř návratové hodnotě, spočítej součástky na desce.** Je-li jich nula, aktivuj list schématu, ulož ho a importuj znovu.

Rozmísti podle těchhle pravidel, pořadí je podle důležitosti:

1. **C2 co nejblíž k napájecímu pinu U1.** Vstupní smyčka je u spínaného zdroje to hlavní.
2. **C1 hned vedle C2**, ne na druhém konci desky. Teče jím objemové zvlnění a dlouhá cesta k němu udělá smyčku, kterou ti žádný nástroj nenahlásí.
3. **Induktor hned u LX padu**, cesta krátká.
4. **Dělič R1/R2 dál od LX uzlu.** Datasheet to žádá doslova.
5. Konektory na protilehlé hrany.

### Kontrola fáze B

Spusť **DRC** ještě před routováním. Chytá věci, které z obrázku vidět nejsou — například součástku naimportovanou obráceně, jejíž pady si sednou na nulovou vzdálenost. Vzdálenosti `0 mil` mezi pady různých netů jsou zkrat.

Nech si od uživatele odsouhlasit rozmístění, než začneš routovat.

---

## Fáze C — Routing a zem

- **Napájecí cesty udělej tak široké, jak to místo dovolí.** Minimum pro 1 A je ~0,3 mm, ale to je bod, kdy se cesta neupeče, ne cíl. Jdi na 0,5–0,8 mm, kde je místo.
- **LX krátký a široký.**
- **Zpětnou vazbu veď dál od LX.**
- **Zem udělej rozlitím mědi na obou vrstvách**, ne cestami, a **sešij vrstvy prokovy** — hlavně u zemního padu regulátoru, po obvodu desky a kolem vstupní smyčky.

> [!CAUTION]
> **Na desce se cesty dvou netů nesmí křížit na jedné vrstvě.** Ve schématu je křížení bez junction dotu v pořádku, na desce je to měď na mědi a DRC to hlásí jako nulovou vzdálenost. Nedá-li se křížení vyhnout, přeskoč to **dvojicí prokovů a krátkým úsekem na druhé vrstvě**.
>
> **Prokov musí vzniknout dřív než cesta, která do něj ústí.** Když jsem v ověřeném běhu nejdřív nakreslil cesty a prokovy doplnil potom, DRC hlásilo ten úsek jako nezapojený, přestože souřadnice seděly na setinu. Po vytvoření prokovů jako prvních a dokreslení cest do jejich středů bylo DRC čisté.

> [!NOTE]
> Rozlití je po vytvoření prázdné, dokud nezavoláš přestavbu měděných oblastí. Bez toho DRC hlásí všechny zemní pady jako nezapojené a vypadá to jako chyba návrhu.

### Kontrola fáze C

**DRC musí projít bez chyb.** Potom:

- Ukaž uživateli obrázek desky.
- Požádej ho o **3D náhled** — mechanické kolize, výšky a konektory na hraně z API nevidíš.
- Zapiš do `REVIEW.md`, co zůstalo neověřené.

---

## Co je potřeba posoudit člověkem

Tohle ti žádný nástroj nenahlásí a agent to má v `REVIEW.md` vypsat jako otevřené:

- **Pinout a footprint U1 proti datasheetu.** Netlist může sedět a footprint být přesto špatný.
- **Polarita LED.**
- **Plochy proudových smyček** a vzdálenost kondenzátorů od pouzdra. Vzdálenost nestačí změřit, musí se vyhodnotit.
- **Odvod tepla.** Pouzdro bez tepelného padu odvádí teplo nohama; pod zemním padem patří prokovy.
- **Úroveň filtrace** na vstupu i výstupu. Tohle zadání má na výstupu jen jeden kondenzátor, což je pro test dost a pro výrobek málo.

---

## Vyhodnocení testu

Prošlo, když platí všechno:

- ERC bez chyb, netlist odpovídá zadání
- DRC bez chyb
- Deska vypadá na obrázku i ve 3D rozumně

Tím je ověřeno, že toolchain funguje od zadání po hotový layout. **Deska z tohohle testu není připravená k výrobě** a není to její účel.

Chceš-li si zkusit něco dalšího, nabízí se stejný obvod pro **12 V → 5 V**. To už ale znamená jiný regulátor, jiné napěťové dimenzování kondenzátorů a vlastní rešerši — tedy přesně to, čemu se tenhle test vyhýbá. Diskutuj to s agentem jako nové zadání, ne jako variantu téhle specifikace.
