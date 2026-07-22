# WebCad – projekt-tudásbázis (CLAUDE.md)

> Ez a fájl a WebCad fejlesztésének teljes átadási dokumentuma a Claude Code-ra
> való átálláshoz. Tedd a projekt gyökerébe `CLAUDE.md` néven – a Claude Code
> automatikusan beolvassa minden munkamenet elején. Minden itt leírt szabály és
> konvenció a korábbi (claude.ai-os) fejlesztési beszélgetésekben alakult ki:
> „Webcad" → „WebCAD 2" → „WebCAD 3" → aktuális szál, valamint a
> „WeCAD PDF export" és a „GUI fejlesztés IBN-DXF konverzióhoz" mellékszálak.

---

## 1. Mi a WebCad?

**WebCad** = „Földmérő CAD ingyen, a böngésződben". Egyetlen, önálló HTML-fájl
(~4,5 MB, ~14 800 sor), külső függőség és szerver nélkül fut bármely modern
böngészőben, asztali gépen és mobilon (érintőképernyőn is). Magyar nyelvű UI,
magyar földmérő munkafolyamatokra: EOV-koordináták, DAT/ITR/FreeTR állományok,
COGO pontok, területszámítás, sraffozás, vektoros PDF-nyomtatás.

- Eredeti név: **GeoCAD** (v5.x) → átnevezve **WebCad**-re, verziószámozás
  v1.0-tól újraindítva (a „Webcad" beszélgetésben).
- Szerző/tulajdonos: © 2026 WebCad · Csóri Miklós.
- Aktuális állapot: **v1.95** a munkafájl (`webcad.html`).
  A látható verziócímke (`#wcVer`, `#verTag`) v1.94-re frissítve.

### 1.1 Melléktermék: WebCad Sraff Lite

Külön fájl (`webcad_sraff_lite_v1_1.html`): a teljes WebCad + a fájl végére
fűzött **„LITE MÓD" blokk** (CSS+JS), ami mindent elrejt, kivéve:
Kezdőlap→RAJZOLÁS panel (csak Sraff + áthelyezett FreeTR import/export gomb),
Tulajdonságok + Réteg-tulajdonságkezelő paletta, Undo/Redo.
Nincs: Mentés/Megnyitás/PDF (gombok + Ctrl+S/P/O lefogva), COGO, Beállítások,
kávé-felugró; drag&drop csak `.ftr`-t fogad. A FreeTR ikonok `stroke:none`-t
kapnak (az örökölt `stroke:currentColor` fehér keretet rajzolt).
**A Lite mindig a fő fájlból regenerálható** az appendix-blokk ráfűzésével –
soha ne ágazzon el a motor.

---

## 2. Fájl- és kódszerkezet

Egyetlen HTML, benne **3 `<script>` blokk**:

| # | Kezdősor (kb.) | Tartalom |
|---|---|---|
| 1 | ~2020 | **proj4js 2.11.0** (MIT, minified) + `EPSG:23700` (EOV) definíció |
| 2 | ~2027 | **LibreDWG-web 0.7.7** (GPL-3.0) beágyazott WASM futtatókörnyezet, gzip+base64 |
| 3 | ~2035 | **A teljes alkalmazás** (~12 000 sor kézzel írt JS) |

A 3. blokkon belül a modulokat nagy banner-kommentek határolják
(`/* ====== MODULNÉV ====== */`). Fontosabbak sorrendben (hozzávetőleges sor):

```
2038   Alap: doc modell, undo, view, W2S/S2W
2424   RAJZOLÁS (render pipeline, batching, entBBox)
3097   Kurzor-szemantika (cursorKind)
3180   TÁRGYRASZTER (SNAP)
3338   TALÁLATVIZSGÁLAT ÉS KIJELÖLÉS
3462   EGÉR ÉS ESZKÖZÖK (+ érintés: take-off, hosszú nyomás)
4529   RÉTEGPANEL
4692   TULAJDONSÁGOK PANEL
5302   ESZKÖZTÁR + COGO-BEÁLLÍTÁSOK bekötés
5465   DXF BEOLVASÁS (ASCII R12–2018)
5715   DWG BEOLVASÁS (LibreDWG WASM)
5976   MENTÉS – WCD + DXF (R12 ASCII, CP1250)
6210   DWG-ÍRÓ (acad-ts, MIT; gzip+base64, lustán töltve; AC1018)
6444   PDF EXPORT (vektoros nyomtatás)
7553   HOSSZ-FELIRAT
7804   RAJZPECSÉT SZERKESZTŐ (izolált vászon)
8177   COGO PONTLISTA ÉS SZŰRÉS
8334   DIGICART ITR (.ibn) import  [FEATURES.ibnImport]
8912   RAJZFÜLEK (drawings[], curDwg)
9379   TRANSZFORMÁCIÓ – kép/PDF georeferálás (pdf.js op-lista vektorizálás)
9654   DAT (DAT-M1/DATR) import
10337  FreeTR (.ftr) IMPORT
10433  FreeTR (.ftr) EXPORT
10588  MÉRÉS (távolság, terület) + zöld pipa
10911  SRAFF (sraffozás)
11189  TERÜLETKIMUTATÁS (areaRep)
11280  Planáris gráf lapkeresés (half-edge)
12288  Méretarány+eltolás/forgatás georeferálás (Helmert-szerű)
12682  RASZTER XREF-ek (rasterStore)
12748  KOORDINÁTA-FELIRAT „zászló" (type:"zaszlo")
12900  PAL128 – 136 színű paletta, uc128Wrap
13336  MAPS RÉTEG (Google csempék a rajz alatt) + Maps Link (pin)
13748  MÓDOSÍTÓ MŰVELETEK (MOVE/COPY/ROTATE/MIRROR/OFFSET/MATCHPROP/LAYOFF/SELSIM)
14592  KOORDINÁTAJEGYZÉK import (TXT/CSV → COGO)
14776  INDÍTÁS (uc128Wrap(document), splash, coffee)
```

---

## 3. Adatmodell

### 3.1 A rajz (`doc`)

```js
let doc = {
  layers:  [ {name:"0", color:"#d7dde5", ltype:"cont", visible:true, lw?} ],
  entities:[ {id, type, layer, color?, ltype?, lw?, ...geometria} ],
  xrefs:   [ {id, name, fade, mono, monoColor, visible, ents:[...]}, // + raster xref
             {kind:"raster", rasterId, ra, rb, rtx, rty, iw, ih, fade, invert} ],
  blocks:  { név: {base:[x,y], ents:[...]} },
  current: "0",           // aktuális réteg neve
  cogoFilter: {...},      // COGO szűrő (rajzhoz kötve)
};
```

Több rajzfül: `drawings[] = {name, doc, view, sel, undo, redo}`, `curDwg` index.
Fülváltáskor teljes állapot-csere (undo-stackekkel együtt).

### 3.2 Entitástípusok és mezőik

| type | Geometria-mezők | Megjegyzés |
|---|---|---|
| `line` | `x1,y1,x2,y2` (+`z1,z2`) | |
| `polyline` | `pts:[[x,y],…]`, `closed` | 3D-nél a pontok `[x,y,z]` |
| `circle` | `cx,cy,r` | |
| `arc` | `cx,cy,r,a1,sweep` | radián; képernyőn Y-tükrözve rajzolva |
| `point` | `x,y(,z)` | kereszt-marker |
| `text` | `x,y,text,h,rot` | rot radián |
| `cogo` | `x,y,z,num,code` | COGO pont; megjelenés a `cogoCfg` szerint |
| `insert` | `name,x,y,rot,scale` | blokk-beillesztés (`flattenInsert`) |
| `zaszlo` | `x,y,z,ex,ey,rot,useY/X/Z,…` | koordináta-felirat mutatóval; `flattenZaszlo` |
| `sraff` | `pts,nums,zs,closed,color,scale,angle` | sraffozás; hatch-vonalak cache-elve |

**Cache-konvenció:** minden `_`-sal kezdődő mező (pl. `_bb`, `_hatch`, `_hk`,
`_pts`) futásidejű gyorsítótár – a `snapshot()` és a WCD-mentés
`(k,v)=>k.startsWith("_")?undefined:v` replacerrel **kihagyja**. Geometria-
módosítás után a cache-t nullázni kell (`e._bb=null` stb., ill. `entCacheClear`,
amely v1.95-től a sraff `_hatch`/`_hk` hatch-cache-t is üríti).

### 3.3 Nézet és transzformáció

```js
const view = { s, tx, ty };                    // s = px/méter
W2S = (x,y)=>[x*view.s+view.tx, -y*view.s+view.ty];   // világ→képernyő (Y fordul!)
S2W = (sx,sy)=>[(sx-view.tx)/view.s, (view.ty-sy)/view.s];
```

`cw, ch` = vászon CSS-mérete, `dpr` = devicePixelRatio (minden canvas
`setTransform(dpr,0,0,dpr,0,0)` alapon rajzol).

### 3.4 Undo

`snapshot()` **minden mutáció ELŐTT** (JSON-mélymásolat, max 60 lépés,
`_`-mezők nélkül). `undo()/redo()` teljes doc-visszaállítás + panel-refresh.

---

## 4. Koordináta-konvenciók (KRITIKUS)

- Világegység = **méter**, a rajz jellemzően **EOV**-ban (EPSG:23700).
- **EOV Y (keleti) → CAD világ X**, **EOV X (északi) → CAD világ Y**.
  Ugyanez a csere a DAT-, FTR-, IBN-importnál is.
- Érvényes EOV-tartomány: `eovInRange(Y,X)` → `Y∈[400000,960000], X∈[30000,380000]`.
- proj4 def: `+proj=somerc +lat_0=47.14439372… +lon_0=19.04857177… +k_0=0.99993
  +x_0=650000 +y_0=200000 +ellps=GRS67 +towgs84=52.17,-71.82,-14.9,0,0,0,0 +units=m`.
- Átváltás: `proj4("EPSG:23700","WGS84",[eovY,eovX]) → [lon,lat]` és vissza.
- Képernyőn az Y-tengely lefelé nő → a W2S-ben negálva; ívek szöge ezért
  rajzoláskor `-a1` irányban megy.

---

## 5. Render pipeline

```
render() → renderScene() → composite() → updateCogoUi()
```

- `renderScene()`: offscreen `sceneCv`-re rajzol (háttér → **Maps réteg** →
  rács/tengelyek → raszterek (`drawRasters`) → xref-ek → `drawEntsFast(doc.entities)`).
  A rajzi elemek MINDIG minden alátét fölött vannak.
- `composite()`: a scene-t a fő vászonra másolja az aktuális view-val skálázva
  (pan/zoom közben olcsó), majd overlay-ek: előnézetek, mérés/sraff overlay,
  szálkereszt, UCS-ikon, fogók, badge-ek.
- Pan/zoom végén `scheduleSharp()` (120 ms) éles újrarajzolást ütemez.
- **Batching:** azonos szín+vastagság+szaggatás vonalas elemek egy `Path2D`-be
  (`batchKey/addToBatch/strokeBatches`); képernyőn <0,7 px-es polyline-lépések
  kihagyva. Láthatósági vágás `entBBox` + `viewRectW` alapján.
- Vonalvastagság-megjelenítés kapcsolható (`lwDisplay`, alsó LWT gomb);
  lekerekített cap/join a csatlakozásokhoz.

---

## 6. UI-struktúra

- **QAT** (gyorselérés): Megnyitás, Mentés, PDF, Undo, Redo (`btnOpenQat`,
  `btnSaveQat`, `btnPdfQat`, `btnUndo`, `btnRedo`).
- **Ribbon fülek** (`.ribTab[data-tab]` ↔ `.ribPage[data-page]`):
  `home` (Kezdőlap), `insert` (Import), `export`, `cogo`, + `#tabSettings`.
  - Kezdőlap panelek: RAJZOLÁS (line, pline, poly3d, circle, arc(+2P-R, ívelt
    vonallánc), COGO-pont, point, **Sraff**), MÓDOSÍTÁS, FELIRAT (zászló,
    hossz-felirat, szabad szöveg), MÉRÉS (táv, terület, területkimutatás),
    BLOKK, RÉTEGEK, GOOGLE MAPS (**Maps Link** pin + **Maps réteg** toggle +
    Műhold/Utca váltó, amely csak aktív rétegnél látszik).
  - Import: DXF/DWG megnyitás; IMPORT panel: DAT, FreeTR, Koordinátajegyzék,
    ITR(IBN), Xref csatol; GEOREFERÁLÁS: PDF, Raszterkép; ELHELYEZÉS panel.
  - Export: FreeTR.
- **Oldalsó paletta** `#side`: `#secProps` (Tulajdonságok + Gyorskijelölés),
  `#secLayers` (Réteg-tulajdonságkezelő), `#secXref` (rejtett, ha nincs xref).
  Húzható grip, `body.noside` kapcsoló.
- **Alsó sáv**: snapBar (3 állású SNAP: on/safe/off + módok: end/mid/node/
  center/int/aint/near/**ext**/**perp**/**par**), rács/tengely checkbox, LWT,
    koordináta-kijelző, hint. Az utóbbi három AutoCAD-stílusú kiegészítő snap:
    Kihosszabbítás (`ext`), Merőleges (`perp`), Párhuzamos (`par`); a perp/par
    csak rajzolás közben, a `drawPts` utolsó pontjából (horgony) számol.
- **Dialógusok**: natív `<dialog>`; üzenet/kérdés a `uiAlert(msg,title)`
  msg-modallal. `optModalOpen/Close` a lebegő opciópanelekhez (zászló, sraff).
- **Színválasztó:** MINDEN `input[type=color]`-t az `uc128Wrap()` cserél
  136 színű palettára (`PAL128`, 17×8). A label-kattintás is a palettát nyitja
  (preventDefault a natív picker ellen – ez volt a „dupla paletta" bug).
- **Kurzor-szemantika (kőbe vésett szabály):** PONT megadásakor CSAK
  szálkereszt; OBJEKTUM-kiválasztáskor CSAK pick-box; semlegesben mindkettő.
  Központi hely: `cursorKind()` – ÚJ parancsnál ide kell felvenni az állapotot
  (measState/sraffState → "point", areaRep pick → "object" stb.).
- **Érintés:** „take-off" ujjkurzor (a cél az ujj FÖLÖTT), két ujj = pan+pinch,
  hosszú nyomás = jobb klikk, lebegő ✓/✕ sáv a módosító műveleteknél.
- **Zöld pipa minta:** poligonzárás a mérésnél/sraffnál az utolsó pont melletti
  zöld ✓ ikonnal (r=12, **hoverkor 2×**, hit-teszt követi); Enter/jobb klikk is
  zár. Elfogadás után az alakzat LEZÁRT: több pont nem vehető fel, gumiszalag
  és pipa eltűnik (meas: `st.done`; sraff edit: `st.edit` – nem bővíthető).

---

## 7. Funkciómodulok (mit hol keress)

- **SNAP** (`computeSnap`, `snapMode`, `safeSnap`): „safe" módban elem csak
  raszterpontra tehető. Több parancs időlegesen felfüggeszti
  (minta: `snapWas` elmentése → `snapMode="off"` → végén visszaállítás;
  lásd `gmapsPinStart`, zászló-szerkesztés `clSnapSuspend/Restore`).
- **Módosítók** (`modOp`): AutoCAD-minta, ige-főnév ÉS főnév-ige; szakaszok:
  sel → base/target/ang/…; élő előnézet; MATCHPROP, LAYOFF (ablakos jelöléssel),
  SELECTSIMILAR.
- **Mérés** (`measState`): táv (2 pont, popover Y/X/Z-vel, másolás gombok),
  terület (pipa → popover: terület/kerület/pontszám, **Mentés TXT-be**
  [pontszám|Y|X|oldalhossz oszlopok + összesítés], **Új poligon** a
  TERULETSZAMITAS rétegre). COGO-ra pattanáskor átveszi a pontszámot és Z-t.
- **Sraff** (`sraffState`, `sraffCfg{color,scale,angle}`): pontonkénti zárt
  poligon; lebegő panel élő 180×180 minta-előnézettel VALÓS léptékben
  (zoomfüggő), élő területtel; „Területszám. TXT"; a kész elem a **Sraffozas**
  rétegre kerül; meglévő sraffra kattintva a panel szerkesztő módban nyílik
  (poligon fix, csak paraméterek). Hatch: `sraffHatchLines` (forgatott
  scanline), cache `sraffCachedHatch`.
- **Területkimutatás** (`areaRep`, `AR`): rétegek vonalaiból planáris gráf +
  half-edge lapkeresés, azonosító feliratok réteg szerint, eredménylista,
  szűrt megjelenítés (`areaRepFilter`).
- **Zászló** (`type:"zaszlo"`, `CL`, `clOpen`): horgony + könyök + szár,
  Y/X/Z pipálható, 4 irány-ikon, helyi fogók (mozgatás töréspontnál, forgatás
  a száron), „Alaphelyzet" gomb csak ha volt mozgatás/forgatás, szerkesztés
  alatt SNAP off. Beállítás mentve: `localStorage["webcad.coordlab"]`.
- **PDF export** (`pdfPrint`): A4–A0, álló/fekvő, méretarány 1:50…1:10000 +
  egyedi + illesztett, színes/FF, húzható zöld keret, teljes képernyős
  előnézet, VEKTOROS PDF-írás kézzel (nem raszter!). Papír-nézetben
  (`pdfPrint.paperMode`) fehér háttér + fekete szálkereszt.
- **Rajzpecsét**: külön izolált vásznú szerkesztő, DWG-mentéssel.
- **COGO**: `cogoCfg` (marker, méret, szín, num/Z/kód feliratok színnel,
  zoomkövető világméret); pontlista-dialógus szűréssel (rajzhoz kötött
  `doc.cogoFilter`); koordinátajegyzék-import TXT/CSV-ből.
- **Georeferálás**: kép/PDF → raszter-xref; pdf.js op-listából VEKTOR-kinyerés
  is; kézi elhelyezés vagy pontpáros (Helmert-jellegű: 
  `X = a·u + b·v + tx ; Y = b·u − a·v + ty`).
- **Maps Link** (`gmapsPin`): pont kijelölése (SNAP off), EOV→WGS84, link
  modalban Google Maps-re.
- **Maps réteg** (`mapsLayerOn`, `mapsLayerType:"s"|"m"`): Google csempék
  (`https://mt{0-3}.google.com/vt/lyrs={s|m}&hl=hu&x&y&z`) a rajz ALÁ;
  zoomszint ≈ 1 csempepx/képernyőpx (HiDPI +1); csempénként WGS84→EOV affin
  illesztés (NW/NE/SW sarkok), max 80 csempe/nézet, 400-as LRU cache,
  crossOrigin="anonymous" + fallback; csak EOV-tartományban tölt.
  Váltógombok (Műhold default / Utca) csak bekapcsolt rétegnél látszanak.
- **Grip-szerkesztés** (v1.95, `vGrip`/`gripHit`/`gripsFor`/`applyGrip`/`drawGrips`):
  kijelölt elem csúcsain kék négyzetek, megfogva húzhatók. Típusok: vonal (2 vég),
  vonallánc / 3D vonallánc és **sraff** (minden `pts[i]`, max 400 csúcs/elem),
  kör (közép = mozgatás + 4 kvadráns = sugár), ív (2 vég + közép → `arc3p`
  újraszámítás; a fix pontok `vGrip.arcPts`-ban rögzülnek a megfogáskor). A húzás
  a tárgyraszterre pattan (`snapExclude` = az elem kimarad a `computeSnap`-ből,
  hogy önmagára ne ragadjon). `snapshot()` a megfogáskor (undo). Minta: a
  zászló/szöveg fogó-rendszere (down/move/up a pointer-router tetején). Csak
  `select`/`none` eszköznél aktív (`gripsBlocked`).

---

## 8. Fájlformátumok

### 8.1 WCD (natív projekt)

JSON: `{app:"WebCad", format:"wcd", version:1, savedAt, name, doc, rasters,
view, cogoCfg}`. A `doc` a `_`-mezők nélkül; raszterek dataURL-ként külön.
Betöltés: mindig ÚJ rajzfülre; a régi `.wcd.json` nevű fájlokat is elfogadja
(tartalom-ellenőrzéssel). **Minden letöltés `application/octet-stream`
MIME-mal megy** – különben egyes böngészők `.json`-t fűznek a névhez (ez volt
egy régi bug).

### 8.2 DXF

- Olvasás: saját ASCII parser, R12–2018 alapelemek.
- Írás: R12 ASCII, **CP1250** kódolás; opciók: COGO pontszám/Z/kód TEXT-ként,
  feliratmagasság. Sraff és zászló nem-natív formátumba egyszerű VONALAKKÉNT
  megy (`sraffToLines`, `flattenZaszlo`).

### 8.3 DWG

- Olvasás: LibreDWG-web WASM (beágyazva, `(0,eval)(umd)` betöltés).
- Írás: **acad-ts** motor (MIT), gzip+base64 beágyazva, LUSTÁN töltve első
  DWG-mentéskor; AC1018 (AutoCAD 2004) bináris.

### 8.4 DAT (magyar DAT-M1 / DATR)

A `DAT_BEOLVAS.lsp` logikája JS-ben. Rétegnév: `DAT_<objid>_<leírás>`;
koordináta-csere: CAD X = pont_y, CAD Y = pont_x. T_PONT/T_HATARVONAL/
T_SZIMBOLUM/T_OBJ_* táblák.

### 8.5 FreeTR (.ftr) – teljes visszafejtett spec

ITR-szerű magyar térképszerkesztő natív formátuma. **ISO-8859-2 + CRLF,
TAB-tagolt**, első mező = rekordtípus. Szekciófejléc: `1 \t név \t darab`.
Szekciósorrend: paraméterek (1+101–200) → Rétegcsoport → **Réteg (3)** →
Felirattípus (4) → Vonaltípus (5) → Jelkulcs csoport → Jelkulcsdef (7/71/72/73)
→ **Pontok (10)** → **Vonalak (20)** → Ívek → Jelkulcsok (30) → **Feliratok (40)**.

- Type 3 réteg: `3|id|csoport|vonaltípus|0|R|G|B|1|1|1|1|NÉV`
- Type 10 pont: `10|pontszám|Y|X|Z|pontkód|0|0|0|pontszám` (EOV!)
- Type 20 vonal: `20|ptIdxA|ptIdxB|réteg_id|2|vonaltípus|0` – a pontindex
  **1-alapú sorszám a pontlistában** (nem pontszám!)
- Type 40 felirat: `40|Y|X|szög|…stílus…|SZÖVEG` (16 mezős; réteg az f[4]-ben)
- Jelkulcs-definíciók szögei mikroradiánban.

Import: `importFtrFile` (Type 3/10/20/40 + 30). Export: `exportFtrFile` –
a valós fájlból kimentett paraméterblokk-sablonnal, ívek/körök szakaszolva,
vonalvégpontok koordináta szerint deduplikálva COGO-pontokkal közösítve,
színek `aciToHex`/`hexToAci` úton (a szín-roundtrip bug már javítva).
Jelkulcs-szekciók exportnál üresek (ismert TODO). Round-trip tesztelve.

### 8.6 ITR (.ibn) – DIGICART

Bináris; a parser az ibn2dxf v5 magja. Tollszám = ACI KÖZVETLENÜL (a hivatalos
konverter DLL-elemzésével bizonyítva); PenStyle = vonaltípus (név szerint);
„zászló" az ITR-ben pontszám-megírás (ZASZLO réteg). Kapcsolható:
`FEATURES.ibnImport` (localStorage `geocad.features`; UI: `data-feature` attr,
`WebCad.setFeature("ibnImport", false)` a konzolból).

### 8.7 Koordinátajegyzék (TXT/CSV)

Elválasztó-autodetekt, fejléc-felismerés; sorrend: `psz, EOV Y, EOV X, Z`
vagy `EOV Y, EOV X, Z` → COGO pontok.

---

## 9. Kőbe vésett konvenciók (NE szegd meg)

1. **Egy fájl.** Minden beágyazva (lib-ek gzip+base64), offline működik.
2. **Magyar UI**, magyar kommentek. Hint a `setHint()`-tel, hiba `uiAlert`-tel.
   Felugró „toast" nincs.
3. `snapshot()` minden doc-mutáció előtt; `_`-mezők = cache (undo/mentés kihagyja).
4. Réteg-létrehozás CSAK `ensureLayer(név, szín, ltype)`-pal. Kötött rétegnevek:
   `Sraffozas`, `TERULETSZAMITAS`, DAT_/ZASZLO minták.
5. Kurzor-szabály (6. pont) minden új parancsra; SNAP-felfüggesztés
   mentsd-vissza mintával.
6. Zöld pipa viselkedés (6. pont): hover 2×, elfogadás után lezárt alakzat.
7. Minden színinput a 136-os palettát használja (uc128Wrap automatikus, de
   dinamikus DOM-nál hívd meg: `uc128Wrap(gyökér)`); label ne nyisson natív pickert.
8. Letöltés `downloadBlob(név, blob)` + octet-stream; fájlnév-suffixok
   levágása: `/\.(wcd|dxf|dwg|json)$/i`.
9. EOV mező-sorrend fájlokban: **Y, X, Z**; CAD-be: x=Y, y=X.
10. FTR: ISO-8859-2 + CRLF, saját kódolóval (TextEncoder nem tud latin2-t!).
11. Sraff/zászló nem-natív exportba vonalakra bontva.
12. Mobil/érintés minden új funkciónál: pointer events, take-off, hosszú nyomás.
13. Teljesítmény: új entitás-típusnál `entBBox` + batching/samplePts támogatás,
    cache-invalidálás módosításkor.
14. A „hívj meg egy kávéra" felugró (localStorage: `webcad.coffeeSeen`,
    `webcad.coffeeOptOut`) a fő verzióban marad, a Lite-ban tiltva.

---

## 10. Fejlesztési munkafolyamat (eddig, és Claude Code alatt)

### 10.1 Eddigi módszer (claude.ai)

- A user feltölti az aktuális `webcad_v1_XX.html`-t → munkamásolat
  `/home/claude/wc/webcad_work.html` → módosítás → export
  `webcad_v1_{XX+1}.html`.
- Kis módosítás: `str_replace` egyedi horgonyszöveggel. Nagy/blokkos:
  Python heredoc (`python3 - << 'PY' … src.replace(old,new); assert old in src`).
- **Kötelező szintaxis-ellenőrzés minden mentés előtt** (a fájl túl nagy a
  `node --check`-hez egyben, script-blokkonként):

```bash
node -e "
const fs=require('fs');
const html=fs.readFileSync('webcad_work.html','utf8');
const re=/<script(?:[^>]*)>([\s\S]*?)<\/script>/gi;
let m,i=0,fail=0;
while((m=re.exec(html))){ i++; const s=m[1]; if(!s.trim()) continue;
  try{ new Function(s); }catch(e){ fail=1; console.log('Script #'+i+': '+e.message); } }
console.log(fail?'FAILED':'OK');"
```

- Funkcionális ellenőrzés jellemzően kód-szintű (pl. FTR round-trip teszt
  Pythonból), mert headless böngésző nem mindig elérhető.

### 10.2 Javasolt Claude Code repo-elrendezés

```
webcad/
  CLAUDE.md              ← ez a fájl
  webcad.html            ← az élő fő fájl (v1.93 tartalma)
  lite/webcad_sraff_lite.html
  docs/FTR_formatum_visszafejtes.md   (ha megvan a régi spec-fájl)
  tools/syntax-check.mjs ← a fenti node-ellenőrző scriptként
  tools/build-lite.mjs   ← LITE MÓD blokk ráfűzése a fő fájlra (regenerálás)
  samples/               ← ftr.ftr, DAT, IBN, DXF mintafájlok (ha vannak)
```

Git: minden release külön commit `v1.XX` taggel. A HTML-ben a `#wcVer` és
`#verTag` szinkronban bumpolandó.

### 10.3 Hasznos keresési horgonyok a kódban

`grep -n "====" webcad.html` → modul-bannerek; entitás-létrehozás:
`type:"..."`; ribbon: `data-page=`, `ribCap`; egérkezelés: pointerdown-router
a ~3670-es sortól (pipa-hit, sraffClick, measClick, modPointClick sorrend);
`updateMouse` (hover-állapotok); `cursorKind`; `renderScene`.

### 10.4 Nyitott TODO-k / ismert ügyek

- [x] UI verziócímke v1.94-re frissítve (release-kor továbbra is bumpolandó).
- [ ] Kihosszabbítás (`ext`) csak a kurzorhoz közeli elem (`nearEnts`)
      meghosszabbítására aktív; ívekre még nincs. Merőleges/Párhuzamos csak
      vonalas elemre + körre/ívre (perp); spline/ellipszis nincs.
- [ ] FTR export: jelkulcs (Type 30 forrás + 7x katalógus) üres; felirat
      méret/szín egyszerűsített.
- [ ] Sraff méretarány-csúszka felső határa 20 m – nagy területeknél emelhető.
- [ ] Maps réteg: Google csempe-endpoint nem hivatalos API (ToS-kockázat);
      alternatíva OSM. Esetleg átlátszóság-csúszka, hibrid (lyrs=y) visszahozható.
- [ ] Dist-mérésnél 2. pont után a további kattintás fura hintet ad (ismert apróság).
- [ ] IBN: vonal menti jelkulcsozás (mintafájl kell hozzá); egyéni DXF-rétegnév-
      megfeleltetés; pontszám ZASZLO rétegre opció (a hivatalos konverterből
      visszafejtett lehetőségek).
- [ ] Lite: külön verziószámozás (jelenleg v1.1), build-scriptesítés.

### 10.5 Legutóbbi változások (ez a szál)

- **v1.91**: zöld pipa hoverkor 2× (mérés+sraff, hit-teszt is); sraff szín
  dupla-paletta fix (uc128 label/preventDefault); mérés közben csak
  szálkereszt; **Maps réteg** toggle + csempe-alátét.
- **v1.92**: Maps réteg típusváltó (akkor még Térkép/Műhold/Hibrid).
- **v1.93**: Műhold default, Hibrid törölve, „Utca" név, váltók csak aktív
  rétegnél; „Google maps" → **„Maps Link"**; sraff-szerkesztésnél nincs
  gumiszalag; **Terület: pipa után lezárt alakzat** (st.done).
- **v1.94**: SNAP-ikonok AutoCAD-stílusra (snapBar SVG-k + `drawSnapMarker`
  glyphek); **3 új tárgyraszter**: Kihosszabbítás (`ext`), Merőleges (`perp`),
  Párhuzamos (`par`). A perp/par a `drawPts` utolsó pontjából (horgony) számol,
  a `par` a legutóbb megcélzott vonal irányát jegyzi meg (`parRef` globális,
  rajzoláson kívül nullázódik). Szaggatott segédvonal a `curSnap.guide`
  mezőből. Új mezők a `snapModes`-ban: `ext/perp/par` (alapból ki).
- **v1.95**: „Poligon"/„3D poligon" eszköz átnevezve **„Vonallánc"/„3D vonallánc"**-ra
  (gombok, hintek). **Szétvetés** (`btnExplode`) most vonalláncot is szétbont
  külön vonalakra (per-csúcs Z megtartva). **Grip-szerkesztés** bevezetve
  (lásd 6. fejezet): kék fogó-négyzetek vonal/vonallánc/3D-vonallánc/sraff/kör/ív
  kijelölésekor, húzással mozgathatók, raszterre pattannak, undo-zhatók.
  `entCacheClear` most a sraff hatch-cache-t is üríti. UI verziócímke v1.95.
- **Lite v1.0 → v1.1**: melléktermék létrehozva; FreeTR import/export a
  RAJZOLÁS panelre, Import/Export fülek törölve, FreeTR ikon keret nélkül.

---

## 11. Hogyan dolgozz ezen a kódon (Claude-nak szóló utasítások)

- A fájl NAGY: mindig `grep`-pel/`sed`-del célzottan olvasd, ne egészben.
- Módosítás előtt olvasd el az érintett modult ÉS a hívási helyeket
  (pointer-router, cursorKind, render-overlay lánc).
- Minden edit után futtasd a szintaxis-ellenőrzőt; UI-változásnál ellenőrizd
  az id-k egyediségét (`grep -c 'id="..."'`).
- Új funkciónál: magyar felirat/hint/tooltip, uc128-kompatibilis színinput,
  érintés-támogatás, undo-snapshot, cache-szabály, kurzor-szemantika.
- A Lite-ot SOHA ne szerkeszd kézzel a motorban – a fő fájl + LITE blokk
  újragenerálásával frissítsd.
- Kérdéses formátum-részletnél (FTR/IBN/DAT) a 8. fejezet a hiteles forrás;
  ha mintafájl áll rendelkezésre, mindig round-trip teszttel igazolj.
- A user (Miklós) magyarul kommunikál, gyakorlott földmérő: a válaszokban a
  szakmai pontosság (EOV, pontszám, réteg) elsődleges; release-enként pontos,
  tömör changelogot vár.
