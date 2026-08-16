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
- Aktuális állapot: **v4.3** a munkafájl (`webcad.html`).
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
| `text` | `x,y,text,h,rot` | rot radián; opc. `italic`,`underline`,`grp`,`gseg` (Auto táv) |
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
  Y/X/Z pipálható, 4 irány-ikon, helyi fogók (mozgatás töréspontnál `move`,
  forgatás a száron `rot`, és v1.98-tól a **kezdőpont/horgony** `anchor` – kék
  négyzet a `(x,y)`-on: húzva a horgony mozog, a könyök (E) a helyén marad
  (`zGrip.E0`), raszterre pattan (`snapExclude`), a koordináták frissülnek),
  „Alaphelyzet" gomb csak ha volt mozgatás/forgatás, szerkesztés alatt SNAP off.
  Beállítás mentve: `localStorage["webcad.coordlab"]`.
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
- **Sraff SZERKESZTŐ mód csúcs-fogói** (v1.96, `sGrip`/`sraffGripHit`): amikor egy
  sraffot a **Sraff** gombbal szerkesztésre nyitsz (`sraffState.edit`), a
  csúcsokon **2× méretű** kék fogók jelennek meg (`drawSraffOverlay`), húzással
  mozgathatók. A húzás a `sraffState.pts` working-copyt ÉS az élő entitást is
  frissíti (`_hk=null`, hatch újraszámol), raszterre pattan (`snapExclude`),
  `snapshot()` a megfogáskor. A generikus fogók (`gripsBlocked`) sraff-szerkesztés
  alatt ki vannak kapcsolva, hogy ne duplázódjanak.
- **Felirat helyben szerkesztése** (v1.99, `txtEdit`/`txtEditOpen`/`txtEditReposition`/
  `txtEditClose`): egy `text` elemre **duplán kattintva** a valós helyén és méretében
  átírható (fixen pozicionált `<input>` overlay, `font-size = h·view.s`, `-rot`
  forgatással). Induláskor minden kijelölve. Fölötte kis eszköztár: **méret-léptető
  ▲▼ (0.1) + számmező**, ami élőben állítja `e.h`-t. Enter: kész, Esc: `undo()`
  (a nyitáskori `snapshot()`). Üresre törölve az elem törlődik. Szerkesztés alatt
  az eredeti canvas-szöveg rejtve (`e._editing` → a `text` rajz-ág kihagyja).
- **Felirat méret-HUD + forgatás-fogó** (v2.0): a méret-panel már NEM a
  szerkesztőben van, hanem **állandó HUD**ként a kijelölt `text` elemtől BALRA
  (`textHud`/`ensureTextHud`/`updateTextHud`, a `composite()` hívja): „Méret"
  számmező + nagy ▲▼ léptető (0.1), élőben állítja `e.h`-t. A felirat-fogók
  (`textGripCenters`/`textGripHit`/`drawTextGrips`) mostantól: `move` (fölül),
  `flag` (zászlózás), és **`rot`** – forgatás-fogó a **bal alsó sarokban**
  (v2.3: a felirat **alján-középen**, `loc(b.wpx/2, 9)`). Forgáspont a bal alsó sarok
  (beszúrási pont), az egér az **alján-középet** húzza: `e.rot = atan2(9,wpx/2) −
  atan2(sy−ay, sx−ax)` – a fogón megfogva nincs ugrás. A forgató-ikon glyphje a közös
  `drawRotArrow(cx,cy,r)` (körív + ÉRINTŐ-irányú nyílhegy); a zászló forgató-fogója is ezt használja.

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
- [x] Kihosszabbítás (`ext`) v1.97-től végpont-megjegyzéssel (acquire) működik:
      a végpont fölé víve megjegyzi a kifelé irányt (`extAcq`, max 4 FIFO), majd a
      folytatás mentén bármeddig pattan. Ívekre még nincs. Merőleges/Párhuzamos
      csak vonalas elemre + körre/ívre (perp); spline/ellipszis nincs.
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
- **v1.96**: **Sraff szerkesztő mód csúcs-szerkesztés** – a Sraff gombbal
  szerkesztésre nyitott sraff csúcsain **2× méretű** kék fogók, húzással
  mozgathatók (`sGrip`/`sraffGripHit`; a working-copy + az élő entitás is
  frissül, raszterre pattan, undo-zható). A generikus fogók sraff-szerkesztés
  alatt tiltva (`gripsBlocked` most `sraffState`-re is figyel). UI címke v1.96.
- **v1.97**: **Kihosszabbítás (`ext`) újraírva** AutoCAD-módra – végpont-megjegyzés
  (`extAcq` [{ox,oy,dx,dy,z}], `extAcqAdd`): a vonal/vonallánc végpontja fölé víve
  megjegyzi a kifelé mutató irányt, majd a képzeletbeli folytatás mentén BÁRMILYEN
  távolságra rápattan (szaggatott segédvonallal). A korábbi verzió csak a kurzorhoz
  közeli elemre (`nearEnts`) működött. UI címke v1.97.
- **v1.98**: **Zászló kezdőpont-fogó** (`anchor`) – a horgony `(x,y)`-án kék
  négyzet, húzva mozgatható; a könyök (E) a helyén marad (`zGrip.E0`), a horgony
  raszterre pattan (`snapExclude`), a felirat koordinátái frissülnek. Új zászlónál
  (textToFlag) és meglévő kijelölésekor is (`selectedZaszlo`). UI címke v1.98.
- **v1.99**: **Felirat helyben szerkesztése** – `text` elemre duplán kattintva a
  valós helyén és méretében átírható (`txtEdit` overlay-input), induláskor minden
  kijelölve; fölötte méret-léptető (▲▼, 0.1 lépés) + számmező, ami élőben állítja a
  betűméretet. Enter: kész, Esc: mégse (undo). Üres szöveg → az elem törlődik.
  Szerkesztés alatt az eredeti szöveg rejtve (`_editing`). UI címke v1.99.
- **v2.0**: **Felirat méret-HUD** – a méret-léptető (▲▼ 0.1) + számmező már a
  `text` **kijelölésekor** látszik, a felirattól BALRA (nagyobb nyilakkal),
  nem csak dupla kattintáskor (`textHud`, a `composite()` frissíti). Új
  **forgatás-fogó** a felirat **bal alsó sarkában** (`textGrip` `rot` kind), a
  beszúrási pont körül forgat – a zászló forgató-fogójának mintájára. UI címke v2.0.
- **v2.1**: a felirat **forgatás-fogója** finomítva – a forgáspont a **bal alsó
  sarok** (beszúrási pont), az egér a **jobb alsó sarkot** képviseli (abszolút
  leképezés `e.rot = −atan2(kurzor−pivot)`), így az alapvonal a kurzor felé néz.
  UI címke v2.1.
- **v2.2**: a felirat **forgatás-ikonja a szöveg közepére** került, és az egér a
  **középpontot** húzza (forgáspont továbbra is a bal alsó/beszúrási pont):
  `e.rot = atan2(-hpx/2,wpx/2) − atan2(kurzor−pivot)`. Középen megfogva nincs
  kezdeti ugrás. UI címke v2.2.
- **v2.3**: a felirat **forgatás-ikonja az aljára-középre** került (`loc(wpx/2,9)`),
  az egér az alján-közepet húzza. A forgató-nyíl iránya javítva: közös
  `drawRotArrow()` a körív **érintője** szerinti nyílheggyel – a **zászló** forgató-
  fogója is ezt kapta (ugyanaz a hibás glyph volt). A méret-HUD ▲▼ gombjai már
  CSS-háromszögek (nem szöveg-karakterek) + `user-select:none`, így nem jelölődnek
  ki szövegként. UI címke v2.3.
- **v2.4**: **Érintés-audit 1. kör – kritikus javítások.** (a) `selCtx()` most hamis
  `measState`/`sraffState` alatt → a lassabb (>350 ms) koppintás már nem nyelődik el
  SHIFT-horgonyként, a mérés/sraff pont lekerül. (b) A hosszú-nyomás (0,6 s jobb klikk)
  nem indul el pont-célzás (`aimTools()`) / mérés / sraff közben → nem szakítja meg a
  módosítót / rajzolást a nyugodt célzás. (c) **Dupla-koppintás** felismerés
  (`lastTap`) a koppintás-ágon → érintésen is megnyílik a felirat helyben-szerkesztője.
  (d) A zöld pipa (terület/sraff zárás) érintésen mindig 2×-es méretben rajzolódik és
  akkora a találata (`isTouchMode()`). (e) Új **befejező pipa** vonal/vonallánchoz
  érintés-módban (`finishBtnRect`, az utolsó csúcs mellett). (f) Vonallánc zárása az
  első pontra érintésen 18 px (volt 10). (g) Módosító objektum-választás érintésen 18 px
  (volt 8). Segéd: `isTouchMode()` (forceTouch || `pointer:coarse`). UI címke v2.4.
  MÉG HÁTRA (audit): grip-húzás nem követi a take-off szálkeresztet (M2/M3), érintéses
  ablakos kijelölés, rétegátnevezés érintésen, HUD képernyő-szélre csúszása/`touch-action`,
  apró (44px alatti) koppintás-célok, snap-tűrés érintésre, 2. ujj grip-húzás közben.
- **v2.5**: **Érintés-audit 2. kör.** (a) **Grip-húzás take-off szálkereszttel**: a
  `pointermove` tetején, ha `vGrip||sGrip||zGrip||textGrip` aktív és érintés, a hatásos
  pont `aimTarget(sx,sy)` (ujj fölött-balra 52 px) → a fogott csúcs/handle láthatóvá
  válik, nem takarja az ujj. (b) **2. ujj kizárása**: `pointerdown` tetején, ha grip-húzás
  aktív és érintés → `preventDefault(); return` (nem pan/zoom, nem kap új gripet). (c)
  **snap-tűrés** érintésen 1,7× (`SNAP_APERTURE*(isTouchMode()?1.7:1)`). (d) A méret-HUD
  és a helyben-szerkesztő **viewportba szorítva** (nem lóg ki bal szélen; balról jobbra
  fordul) + `touch-action:manipulation` (nincs dupla-koppintás-zoom). (e) Nagyobb
  koppintás-célok érintés-eszközön (`@media (pointer:coarse)`: `.clSw`, `.measPop .cpy`,
  `.mbBtn`). UI címke v2.5. MÉG HÁTRA: érintéses ablakos kijelölés, rétegátnevezés
  érintésen, `title` tooltipek, 136-színű paletta cellaméret.
- **v2.6**: **TERVPECSÉT TERVEZŐ** – új ribbon-fül „Tervpecsét" (`data-tab="stamp"`,
  panel „TERVEZÉS", `btnStampDesign`) → 80%-os modal (`#dlgStamp`). Táblázatos
  tervpecsét-szerkesztő (önálló IIFE-modul a script végén, `window.TervPecset.open()`).
  Modell: `stamp={name, rows:[{h:1..4, cells:[{w, items:[{label,value}|null …h]}]}]}`,
  `localStorage["webcad.tervpecsetek"]` (Mentés/Betöltés/Törlés/Új/Név). Sor: föl-le
  nyíl a magassághoz (1–4×, `setRowH`), sor-törlés; cellák egyenlő szélességgel
  (`equalize`), közöttük húzható elválasztó (`wireDivider`, pointer-events). Mező:
  „címke" (bal-fent, kicsi, contentEditable) + „mezőérték" („Minta szöveg", középen,
  nagyobb, contentEditable); piros X = érték törlése; üres alsor = zöld +; több
  magasságú sorban az érték ▲▼-vel mozgatható alsorok között, és több alsorba is
  tehető érték. `+ Új sor`, sor-végi `+` = új mező. MÉG: a tervpecsét rárakása a
  rajzra/PDF-re (a meglévő Rajzpecsét-motorral integrálva) – következő lépés. UI címke v2.6.
- **v2.7**: **Tervpecsét a nyomtatási képen.** A tervező eszköztárában a „Nyomtatásra"
  gomb a modellt a `tervpecsetForPrint` globálisba teszi és megnyitja a PDF-módot. A
  `paintPrintScene(P,r)` sétálóba egy hívás került (`paintTervpecset(P,r)`), így a
  tervpecsét a papír **jobb-alsó sarkába** rajzolódik – UGYANAZ a kód fut a teljes
  képernyős előnézeten ÉS a vektoros PDF-ben (a `P` festő absztrakción át:
  `setStroke/begin/move/line/close/stroke/text/fillRectW`, világkoordináták,
  `pdfMM2W`/`pdfRectWorld`/`printColor`). Méret: `TP_WIDTH_MM=185`, `TP_ROW_MM=8`;
  a szöveg balra igazított/alfabetikus alapvonal → a mezőérték középre igazítása
  szélesség-becsléssel (`0.5·h·hossz`). Az élő rajzvászon-előnézethez egy külön
  `screenPrintPainter()` + `drawPrintTervpecsetOnScreen()` (a `drawPdfPrintOverlay`-ből,
  `drawPrintStampOnScreen` után). `window.TervPecset.setForPrint(model)`. MÉG FINOMÍTHATÓ:
  méret/pozíció fogóval, mezőnkénti betűméret/igazítás, ki/be kapcsoló a PDF-sávban. UI címke v2.7.
- **v2.8**: **Címkoordináta** – új gomb a RAJZOLÁS panelen (`btnCimCoord`, a Sraff
  után). Speciálisan elhelyezett COGO pont, amit **csak egész méteres rácspontra**
  lehet letenni (méter élességgel, kerek értékre). A `cogoPlace` mechanizmusra épül
  egy `cim:true` zászlóval, így minden meglévő útvonal (kurzor, érintéses aim,
  `primaryAction`→`placeCogoAt`) újrahasznosul. `dlgCim`: kezdő pontszám + pontkód
  (alapérték **5412**, z=0, növekmény fix **+1**). `startCimCoord()` állítja be a
  módot. `placeCogoAt` elején `if(cogoPlace.cim){ x=Math.round(x); y=Math.round(y);
  z=0; }` → egész rácspont; a letett pontok a `cimPts[]`-be gyűlnek
  (`{num,kelet,eszak,code}`, kelet=CAD x=EOV Y, észak=CAD y=EOV X). A **snap
  ideiglenesen KI** (`computeSnap` elején `cim` ág visszatér üres `snapInfo`-val).
  Vizuál: `drawCimGridDots()` a `renderScene`-ben (rajz ALATT) halvány szürke pontok
  a rácspontokon (léptékfüggő, `view.s<6` alatt nem rajzol, >40000 pontnál kihagy);
  `drawCimNearest()` a `composite`-ban (rajz FELETT) a kurzorhoz legközelebbi
  rácspontot **piros, ~2× akkora** ponttal emeli ki. Befejezés jobb klikk
  (`secondaryAction`) vagy Esc (`cancelTool`) → ha van letett pont, `openCimSave()`.
  `dlgCimSave`: pontkód-oszlop opcionális (`cimWantCode`), elválasztó választható
  (`cimSep`, alap **TAB**; `;` `,` szóköz). `cimDoSave()` → `koordinatajegyzek.txt`
  (`downloadBlob`), oszlopsorrend: **pontszám · kelet(Y) · észak(X) · [pontkód]**,
  3 tizedes, CRLF sorvég. UI címke v2.8.
- **v2.9**: **Címkoordináta – teli kör jelkulcs + felirat, blokként exportálva.** A
  letett pont ezentúl NEM COGO pont, hanem a **`Cimkoordinata`** rétegre (amber
  `#ffd75e`, `ensureLayer`) kerülő **blokk-beillesztés + szöveg**. A jelkulcs egy
  közös blokk-definíció, **`CIMKOORD_PONTJEL`** (`doc.blocks`), **egység-sugarú kör +
  ~22 vízszintes kitöltővonal** = „teli kör" SŰRŰ vonalakból (nem solid fill, hanem
  vektor-vonalak). A blokk egységben (r=1) van, a beillesztés `sx=sy=jel sugár`-ral
  skáláz → a méret dialógusból állítható a blokk újradefiniálása nélkül. A `dlgCim`
  két új mezőt kapott: **Jel sugár (m)** (`cimSymR`, alap 0.5) és **Felirat magasság
  (m)** (`cimTxtH`, alap 1.0). `createCimMarker(x,y,num,code)`: `snapshot()` →
  `ensureLayer` → `ensureCimBlock()` → `insert` {name:CIMKOORD_PONTJEL, sx/sy:R} +
  `text` {a pontszám, a jeltől jobbra-fel, `x+R*1.4, y+R*0.3`, magasság H}. A
  `placeCogoAt` cim-ága `createCogoPoint` helyett `createCimMarker`-t hív; a
  koordinátajegyzék továbbra is a `cimPts[]`-ből épül. **DXF/DWG export: automatikus**
  – a meglévő blokk/INSERT/TEXT gépezet írja ki (a `buildDxfText` BLOCKS-szekciója és
  a `buildDwgBytes` `BlockRecord`+`Insert`-je); a blokk ents `layer:"0"` → beillesztés
  rétegét öröklik (`flattenInsertWith`). Tesztelt: 2 insert + 2 text a helyes rétegen,
  egész rácspontokon, DXF-ben blokk-def + 2 INSERT + 2 TEXT, DWG 27 entitás hibátlanul.
  UI címke v2.9.
- **v3.0**: **EOV-tudatos tengelynevek és koordináta-kiírás.** Új `computeEovCtx()` +
  gyorsítótárazott `_eovCtx` (a `renderScene` frissíti, NEM képkockánként): a rajz
  tartalmi közepe (üres rajznál a képernyő közepe) EOV-tartományban van-e
  (`eovInRange(világX, világY)`). `eovContext()` olvasó, `eovAxis("X"/"Y")` a CAD-tengely
  EOV-nevét adja (EOV-ban **világ-X = „Y" = kelet**, **világ-Y = „X" = észak**). Hatások
  EOV-tartományban: (a) a **bal-alsó origó-ikonon** a vízszintes (kelet) tengely neve
  **Y**, a függőleges (észak) **X**, és kis kék **„eov"** felirat (drawUcsIcon). (b) A
  **kurzor-koordináta kiírás** (`#coords`) EOV-ban `Y <kelet>  X <észak>  Z` alakú
  (nem-EOV: a régi `x, y, z`). (c) A **tulajdonságpanel** tengelycímkéi EOV-ban
  átnevezve (X↔Y): „X"/„Y", „Közép X/Y", „Kezdő/Vég X/Y", „Pozíció X/Y", „Beill. X/Y",
  valamint a „Mind (X,Y,Z)” → „Mind (Y,X,Z)” másoló gomb. **A koordináták SORRENDJE
  mindig kelet, észak (világ X, majd Y) marad** – csak a MEGNEVEZÉS változik; a
  COGO-export és a koordinátajegyzék eleve kelet(Y)/észak(X) sorrendű. UI címke v3.0.
- **v3.1**: **Auto táv – automatikus szakasz-hossz feliratozás feliratcsoporttal.**
  Új „Auto táv” gomb a FELIRAT panelen (`btnAutoLen`). Nem-modális, jobb-felső sarokba
  dokkolt beállító ablak (`dlgAutoLen`, `.show()`, átlátszó `::backdrop`, `transform:none`
  a globális `dialog[open]` középre-húzás felülírására) → **élő előnézet a vásznon**
  (`drawAutoLenPreview` a composite-ban). Lépések: (1) **forrás réteg** listából vagy
  **elemre mutatással** (`alPickBtn` → `pickMode`, a `primaryAction` legelső ága kezeli,
  `lenHitSegment`); (2) **cél réteg** listából vagy új név, alapból **`hossz-<forrás>`**
  (előtag!); (3) betűméret (±0.1 léptető gombok), tizedes, **vonal fölött/alatt/on**,
  eltolás, **álló/dőlt**, **aláhúzott**. A forrás réteg összes `line`/`polyline`
  (2D és 3D) szakaszának hosszát feliratozza. **A feliratok MINDIG talpon** (sosem
  fejjel lefelé): `lenUprightAngle`, az „olvasó felfelé” irány `(-sin rot, cos rot)`
  adja a fölött/alatt eltolást. „Megírás” csak akkor aktív, ha van forrás- és cél
  réteg és **van méretezhető vonal** (különben piros üzenet). Kulcs-fv-ek:
  `autoLenCollect`, `autoLenPlace` (szakasz→felirat), `autoLenWrite`, `autoLenEditGroup`.
  **Feliratcsoport:** minden megírás egy `grp` id-t kap; minden felirat-entitáson
  `grp` + `gseg:{a,b,is3d}` (a szakasz geometriája az újraszámításhoz), a csoport
  beállításai `doc.labelGroups[grp]`-ban. A tulajdonságpanelen egy csoportbeli felirat
  kijelölésekor „Csoport szerkesztése…” gomb (`btnGrpEdit`→`autoLenEditGroup`): ugyanaz
  az ablak „Alkalmaz” gombbal, ami a csoport MINDEN feliratát egyszerre újraszámolja
  (méret/tizedes/oldal/eltolás/stílus/réteg). **Perzisztencia:** `grp`/`gseg`/`labelGroups`
  nem `_`-mezők → a **WCD megőrzi** őket; DXF/DWG-be csak a geometria megy → **sima
  önálló feliratok** (a dőlt = 51-es oblique 15°, az aláhúzás `%%u` előtag; a `text`
  render `case` kezeli az `italic`/`underline`/`oblique` megjelenítést). UI címke v3.1.
- **v3.2**: **Ortogonális – derékszögű vonallánc mérési vonal + abszcissza/ordináta
  módszerrel.** Új „Ortogon." gomb a RAJZOLÁS panelen (`btnOrtho`). Állapotgép (`ortho`
  globális): `phase` = `"s"`(kezdőpont)→`"e"`(végpont)→`"num"`(számbevitel)→`"choose"`
  (nyilak/pipa/X). A mérési vonal **piros szaggatott** (abszcissza előtt VASTAG, utána
  vékony); a kész szakaszok **vastag zöldek** (`drawOrthoOverlay` a composite-ban).
  Számbevitel: **`#orthoNum` mini-modal** a ponthoz pozicionálva – először **Abszcissza**
  (a vonal mentén, lehet negatív), utána **Ordináta** (merőlegesen). Élő zöld előnézet
  gépelés közben (`_preview`). Az utolsó pontban 3 kattintható képernyő-ikon
  (`orthoIconPositions`, hit-teszt a `primaryAction` ortho-ágában): **jobb/bal nyíl**
  (a szakasz jobb/bal oldalán, merőleges irány `orthoPerp`), **zöld pipa** (befejezés),
  és az utolsó szakasz közepén **piros X** (utolsó szakasz visszavonása → előző pont).
  Nyílra kattintva `pendingDir` = a merőleges irány, majd újra `#orthoNum`. A `heading`
  mindig az utolsó szakasz iránya → derékszögű, **folyamatos** vonallánc. Befejezéskor
  (`orthoFinish`) egy `polyline` entitás jön létre a `doc.current` rétegen.
  **Beállítópanel (`#orthoBar`) végig látszik**, opciók pipával nyílnak: (1) **COGO pont
  minden töréspontban** – kezdő pontszám / növekmény / kód (Z nélkül, ahogy kérted);
  (2) **Távolságok megírása** – betűméret/tizedes/eltolás, KÖZÉPRE a szakasz fölé
  (`autoLenPlace`, feliratcsoportba `grp`-vel). `cancelTool`/`setTool`/`secondaryAction`
  integrálva (jobb klikk: choose→befejezés, num→mégse). UI címke v3.2.
- **v3.3**: **Ortogonális érintő-támogatás + finomítás.** (a) A pont-célzás (`s`/`e`
  fázis) érintésen a take-off célkereszttel megy: `aimTools()` igazat ad ortho pick
  fázisban, és a pick fázis kizárva az egyujjas panból (`touchPan` feltétel) → az
  ujj-húzás célkeresztet ad, felengedésre a kereszt helyére kerül a pont. A choose
  fázisban a nyilak/pipa/X **közvetlen koppintással** kezelhetők (touchPan-tap →
  `orthoClick` ikon hit-teszt, `isTouchMode()`-nál 26 px tűrés). (b) **A nyilak
  közelebb a vonalhoz**: `Rr` 42→26 (érintésen 30), a pipa 30/34. Érintésen az ikonok
  1.28×-osra nagyítva (`orthoTouchK`). (c) Új **gumivonal** a végpont-célzáshoz
  (`e` fázis: kezdőponttól a kurzorig piros szaggatott). UI címke v3.3.
- **v3.4**: **Tollszínek – rétegenkénti nyomtatási szín (színes nyomtatás).** Új
  „Tollszínek” gomb a réteg-tulajdonságkezelő eszköztárában (`btnPenColors`), modal
  `dlgPenColors`. Két mód: **eredeti színnel** (`doc.penUseOriginal=true`, alapértelmezett)
  vagy **egyedi tollszínek** (`doc.penUseOriginal=false`, `doc.penColors={rétegnév:hex}`).
  A modal **csak azokat a rétegeket listázza, amelyeken van rajzi elem** (`penUsedLayers()`
  = a `doc.entities` rétegei). Rétegenként `<input type=color>`, és **tömeges** kitöltés
  (`penBulkApply` → minden rétegre ugyanaz). Nyomtatáskor `penColorFor(e)` adja a forrás-
  színt: egyedi módban a réteg tollszíne (ha nincs, az eredeti), különben az eredeti.
  Beépítve a nyomtatási útvonalakba: `entDrawColor` (papír-előnézet) és `printEntStyle`
  (vektoros PDF) – `printColor(penColorFor(e))`. A **fekete-fehér** (`pdfPrint.mono`)
  nyomtatás a `printColor` első sorában MINDIG felülírja feketére (a tollszínt is).
  Csak a nyomtatásra hat; a képernyőn és DXF/DWG-ben az eredeti színek maradnak; a
  `penColors`/`penUseOriginal` a `doc`-on van → **WCD-be mentődik**. Undo: `penMarkDirty`
  (egy snapshot editálási munkamenetenként). UI címke v3.4.
- **v3.5**: **Pont-kereszt elrejtése nyomtatáskor.** A `point` (csomópont) entitás
  keresztjelölője nyomtatáskor nem jelenik meg: a papír-előnézetben a `drawEntity`
  `case "point"` elején `if(pdfPaperView) break;`, a vektoros PDF-ben a
  `paintPrintScene` `case "point"` üres (`break`). A szerkesztő nézetben továbbra is
  látszik. A COGO-pontok jelkulcsa/felirata változatlan. UI címke v3.5.
- **v3.6**: **PONTFELHŐ – LAS/E57/PLY beolvasás + négynézetes WebGL nézegető, ÚJ
  RAJZFÜLBEN.** Új ribbon-fül „Pontfelhő" (`data-tab="pcloud"`): Betöltés (rejtett
  `#pcFile` input), Nézegető, INFÓ panel. **Parserek:** `pcParseLAS` (1.2–1.4, PDRF
  0–10, LAZ-t elutasítja; skála+offset, intenzitás, osztály, RGB 16→8 bit
  autodetekt), `pcParsePLY` (ascii + binary_little_endian + BE; vertex x/y/z +
  red/green/blue/intensity, típus-normalizálás), `pcParseE57` (ASTM E57: 1024 bájtos
  CRC-lapok logikai olvasása `e57ReadLogical`, XML DOMParser, CompressedVector
  szakasz-fejléc + adat-paketek, **bitpack kodek**: Float double/single bájtos,
  Scaled/Integer LSB-first bitmezők; póz (kvaternió+eltolás), spherical→cartesian,
  több szken összefűzve). Mindegyik **ritkít** `PC_MAX_POINTS`=5M fölött. **EOV-nagy
  koordináták offsetelve** (Float32 pontosság): |közép|>5000 → kerekített offset,
  a kijelzés visszaadja. **Fül-integráció:** a felhő új `drawings[]` elemként jön
  létre `pc` mezővel (`pcState`: cloud, views, GL-bufferek); `activateDwg` végén
  `pcTabSync()` mutatja/rejti a `#pcPanel`-t (a `#canvasWrap`-ben, a vászon fölött),
  `closeDwg` `pcFreeTab`-bal szabadít. NEM mentődik a projektbe. **Nézegető:** egy
  WebGL-vászon 4 scissor-viewporttal – bal-fent TOP (X/Y), jobb-fent RIGHT (Y/Z),
  bal-lent FRONT (X/Z), jobb-lent IZOMETRIKUS; **csak az ISO forgatható** (húzás:
  yaw/pitch), a többi kötött; pan (húzás), zoom-a-kurzorra (görgő), pinch (2 ujj).
  **Perspektíva** checkbox csak az ISO-ra, alapból KI (ortho). Színezés: RGB /
  magasság-gradiens / intenzitás / osztályozás (ASPRS paletta) / egyszínű – a nem
  elérhető opciók tiltva. Pontméret-csúszka, Illesztés (fit), státuszsor EOV-
  koordinátákkal (TOP: Y X, FRONT: Y Z, RIGHT: X Z). `window.PC` teszt-API
  (load/active/render/fit). UI címke v3.6.
- **v3.7**: **Pontfelhő-teljesítmény: adaptív LOD + progresszív renderelés.** A v3.6
  minden pointermove-ra szinkron újrarajzolta mind a 4 nézetet a teljes pontszámmal →
  használhatatlanul lassú. Megoldás: (1) betöltéskor **Fisher–Yates keverés** – így a
  tömb ELEJE egyenletes véletlen minta, `drawArrays(0,lodN)` = uniform LOD; (2)
  interakció közben rAF-re kötve **csak az érintett negyed** rajzolódik `lodN`
  ponttal (`pcRequestRender(key,true)` → `pcDrawInteractive`); a `lodN` a MÉRT
  rajzidőből adaptív (>36 ms → csökken, <12 ms → nő), a szinkronpont **readPixels**
  (a `gl.finish` SwiftShaderen nem blokkol!); (3) nyugalomban (220 ms, csak ha nincs
  lenyomott gomb/ujj) **progresszív teljes rajz**: `pcRender`→`pcProgStep` képkockán-
  ként `lodN`-nyi pontot AD HOZZÁ minden negyedhez (POINTS + depth test miatt a
  darabolás korrekt), interakcióra azonnal megszakad (`pcCancelProg`). Kontextus:
  `powerPreference:"high-performance"`; a `preserveDrawingBuffer` kell a
  negyed-frissítéshez. Mérés (SwiftShader, 3M pont): húzás képkocka 4700 ms → ~30 ms.
  UI címke v3.7.
- **v3.8**: **Pontfelhő-vágás (crop) + nézet-beforgatás.** A ribbonból a „Nézegető"
  gomb és a tooltipek törölve; helyettük: **Tégl. vágás**, **Szabad vágás** (elforgatott
  négyzet ikon), **Vágás reset** és **Forgatás reset** (utóbbi kettő csak élő
  vágás/forgatás esetén látszik, `pcSyncCropUi`). **Megvalósítás:** a vágás a VERTEX
  SHADERBEN történik (uCropOn/uCropA/uCropB/uCropLim uniformok; a kieső pont
  `gl_Position=vec4(2,2,2,1)` → klippelve) – nincs CPU-szűrés, a reset azonnali.
  `st.crop={A,B,lim}`: tartás, ha `aMin≤dot(p,A)≤aMax && bMin≤dot(p,B)≤bMax` (a
  mélységi tengely mentén korlátlan sáv). **Téglalap vágás:** a 3 ortho nézet
  egyikében (IZOMETRIKUSBAN TILTOTT) húzással/két kattintással; A/B = a nézet
  eff. R/U tengelyei. **Szabad vágás:** (1) alapvonal gumivonallal, (2) harmadik
  ponttal merőleges szélesség (gumitéglalap) → elforgatott téglalap-vágás, ÉS (3) a
  nézetek beforgatása: `st.frame` 3×3 forgatómátrix (`m3rot` az adott nézet
  mélységtengelye körül az alapvonal szögével, `m3mul`-lal komponálva) – az
  `pcEffBasis(st,key)` minden nézet-bázist (MVP, pan, zoom, státusz) ezen át ad
  vissza, így az alapvonal lesz az új vízszintes tengely mindenhol. Gumivonalak:
  `#pcOverlay` 2D vászon a WebGL fölött (`pcOverlayDraw`). Eszköz-állapotgép:
  `pcTool` (stage 0/1/2, húzás ÉS klikk-klikk mód), Esc/jobb klikk: mégse; a
  pointerdown/move/up kezelők elején fut. UI címke v3.8.
- **v3.9**: **Pontfelhő-mérés gumivonallal.** Új „Mérés" gomb a Pontfelhő ribbonon
  (`btnPcMeasure`), a `pcTool` állapotgép `kind:"meas"` ágaként. A 3 ortho nézetben
  (ISO tiltva, mint a vágásnál) kattintás/húzás az első ponttól: sárga gumivonal, kék
  végpont-négyzetek, a vonal közepén ÉLŐ távolság-címke (`pcOverlayDraw` meas-ága,
  `pcToolScrV` nézet-független vetítővel), a hint-sávban élő `Táv / ΔY / ΔX(/ΔZ)`
  komponensek (`pcMeasFmt`, EOV-nevekkel nézetenként: TOP ΔY/ΔX, FRONT ΔY/ΔZ,
  RIGHT ΔX/ΔZ). Második pont → az eredmény a hintben marad + a vonal a rajzon
  (`pcTool.last`), és azonnal új mérés kezdhető; Esc / jobb klikk: befejezés.
  A távolság a nézetsík 2D távolsága (a,b skalárkoordináták különbsége – az
  EOV-offset a különbségben kiesik). Teszt: pixelből függetlenül számolt várt érték
  = mért érték (31.182 m). UI címke v3.9.
- **v4.0**: **Pontfelhő-digitalizálás: Pont / Vonal / Vonallánc CAD-entitás két
  nézetből.** Új gombok a Pontfelhő ribbonon (`btnPcVPt/VLine/VPline`), a `pcTool`
  állapotgép `vpt/vline/vpline` fajtái (`PC_VKINDS`). **Mechanika:** minden csúcshoz
  3 koordináta KÉT nézetből: az első kattintás egy ortho nézetben 2 munkakoordinátát
  ad (`PC_VAX`: TOP→x,y; FRONT→x,z; RIGHT→y,z), a hiányzó harmadikat egy MÁSIK
  nézetben kell megadni. Az első kattintás után: (a) a másik két nézet **hozzáigazodik**
  az ismert koordinátákhoz (`pcVexAlign` – a nézetközéppont az ismert tengelyek mentén
  eltolva), (b) bennük **szaggatott segédvonal** jelzi az ismert koordináta helyét, a
  kurzort zöld kör-jelölt követi A SEGÉDVONALON (a kattintás rá van pattintva: az
  ismert tengely értéke az első nézetből jön, csak a hiányzó jön a kattintásból).
  Bármelyik nézetben kezdhető (TOP/FRONT/RIGHT szimmetrikus); ISO-ban tiltott.
  Munkakoordináták a `frame`-forgatással kompatibilisek (`pcWorkOf`=Mᵀ·p /
  `pcWorldOfWork`=M·w). **Kimenet:** a kész elem VALÓDI CAD-entitásként (EOV
  koordináták = lokális + offset) az első nem-pontfelhő rajzba kerül (`pcVexTargetDoc`,
  ha nincs, létrejön), a **PF_vektor** rétegre: point / line (z1,z2) / 3D polyline.
  A nézegetőben az elhelyezett vektorok zölden mindig látszanak (`st.vec`, overlay,
  negyedenkénti clip-peléssel), pan/zoom közben is frissülnek (overlay hívás a
  `pcDrawInteractive`/`pcProgStep` végén). Vonal: 2 csúcs után kész; vonallánc:
  jobb klikk zárja (≥2 csúcs); Esc: megkezdett elem elvetése, második Esc: kilépés.
  UI címke v4.0.
- **v4.1**: **ISO-nézet javítás + Georeferálás + Küldés CAD-be.** (a) A pontfelhőben
  megrajzolt Pont/Vonal/Vonallánc mostantól **az izometrikus nézetben is látszik**:
  új `pcProjScreen(st,key,p)` a valódi `pcMVP`-mátrixon át vetít (RAW, frame-el nem
  előre forgatott pont, ugyanúgy ahogy a GPU vertex shader kapja), ortho ÉS
  perspektíva iso esetén is helyesen – a kamera mögötti pontoknál `null`-t ad, azok
  a szegmensek kimaradnak. (b) **Elhalasztott küldés:** a Pont/Vonal/Vonallánc többé
  NEM kerül azonnal CAD-entitásként a rajzba – csak a pontfelhő-nézegetőben tárolódik
  (`pcActive.vec`, zölden mindig látszik). Új **„Küldés CAD-be”** gomb (`pcSendToCad`)
  viszi át ténylegesen, VALÓDI 3D entitásként (point/line/3D-polyline, `PF_vektor`
  réteg) „a CAD nézetben aktív rajzba” – ehhez új `lastCadDwg` globális (az utoljára
  aktivált NEM pontfelhő fül indexe, `activateDwg`-ben frissül); `pcVexTargetDoc()`
  ezt preferálja. (c) **Georeferálás** – új „Georef” / „Georef reset” / „Küldés CAD-be”
  gombcsoport. Session-állapot `pcActive.geo={pairs,scaleMode}`, **túléli** a
  vágás/szabad vágás/vágás-reset eszközváltásokat (külön tárolva a `pcTool`-tól).
  Munkafolyamat: „Új pontpár” → a pontfelhő-oldali pont a megismert 2-nézetes
  mechanikával (`geopt` fajta) → ELFOGADÁS UTÁN **modal ablak nyílik 50vw×50vh
  méretben**, benne a CAD nézetben aktív rajz TARTALMA (a `#cv` canvas, a snap-sáv és
  a réteg-tulajdonságkezelő ideiglenes DOM-átköltöztetéssel – `insertBefore`
  visszaállítással –, ribbon/menü NÉLKÜL), snap-pel kattintható; „Elfogad” rögzíti a
  pontpárt. A `#cv` átméretezését modal alatt `pcGeoResizeCv()` végzi (a globális
  `resize()` a `geoModalOpen` flag alatt kilép); első megnyitáskor `zoomExtents()`,
  utána a nézet fülönként megőrződik. A tényleges kattintást `geoPickCapture` fogja el
  a `primaryAction()` tetején (nem hoz létre entitást, csak `pickPointZ()`-t olvas ki).
  **Minimum 2, javasolt 3+ pontpár** (a gomb 2 alatt tiltott). **Transzformáció:**
  N≥3-nál Horn (1987) kvaterniós legkisebb négyzetek illesztés (`pcHornFit`, saját
  4×4 szimmetrikus Jacobi-sajátérték-megoldó, `pcJacobiEigSym`) – forgatás + eltolás +
  opcionális méretarány (Umeyama-formula); N=2-nél egzakt 2D vízszintes Helmert +
  Z-affin (`pcTwoPointFit`, mert 2 ponttal a 3D forgatás nem egyértelmű). **Opció:**
  méretarány megtartása (1:1) vagy számítása a pontokból. Elfogadás után a
  megbízhatóság (RMS tengelyenként + összesen, legnagyobb eltérés) megjelenik.
  **Alkalmazás** (`pcGeoApply`): a TELJES pontfelhő pozícióit (abszolút = pos+offset)
  transzformálja, új Float32-barát offsetet számol, újraépíti a GPU puffereket, a
  `st.vec` elemeket is konzisztensen áthelyezi, törli a `crop`/`frame`-et (más
  koordinátarendszer). Első alkalmazáskor `st._geoPristine` pillanatkép (pos+offset+vec)
  → **„Georef reset”** ebből állít vissza. UI címke v4.1.
- **v4.2**: **Javítás: a georeferálás CAD-választó modal indításkor is látszott.** A
  `#dlgGeoPick{...display:flex...}` szabály ID-szelektora (specificitás 100) felülírta
  a böngésző natív `dialog:not([open]){display:none}` szabályát (specificitás ~11),
  így a dialog `open` attribútum nélkül is látszott. Javítás: a `display:flex` átkerült
  egy `#dlgGeoPick[open]{display:flex}` szabályba, a bázis `#dlgGeoPick{...}` szabály
  már nem határoz meg `display`-t → zárt állapotban ismét a natív `display:none`
  érvényesül. UI címke v4.2.
- **v4.3**: **Georeferálás – rajz-választás, 70%-os modal, húzható panel.** (a)
  **Javítás: a rajz nem jelent meg a CAD-választó modalban** – a `pcGeoResizeCv()`/
  `zoomExtents()` a `showModal()` ELŐTT futott, amikor a dialog még `display:none`
  volt (0×0 méretű), ezért a `view.s` érvénytelen (negatív/végtelen) skálát kapott.
  Sorrend cserélve: előbb `$("dlgGeoPick").showModal()`, csak utána a vászon
  méretezése és illesztése. (b) **Több nyitott CAD-rajz esetén választható a cél**:
  `pcCadDwgList()`/`pcGeoResolveTargetIdx()` – ha csak egy nem-pontfelhő fül van
  nyitva, automatikus (a választó sor rejtve); ha több, megjelenik a „Cél CAD-rajz”
  legördülő a georef-panelen (`pcGeoSyncTargetUi`), alapból a legutóbb aktivált CAD
  fülre (`lastCadDwg`) állva. A választás `pcActive.geoTargetIdx`-ben tárolódik,
  ugyanezt használja a „Küldés CAD-be” (`pcVexTargetDoc`) is – konzisztens célrajz.
  (c) **A modal 70vw×70vh** (volt 50×50). (d) **A georef-panel (`#geoBar`) középen
  felül** (`top:14px; left:50%; transform:translateX(-50%)`, nem jobb szélen), és
  **a fejlécénél fogva húzható** (pointerdown/move/up a `.gbTitle`-ön, első húzáskor
  a `transform` törlődik és abszolút `left/top` px-re vált; a ✕ gomb kattintása nem
  indít húzást). UI címke v4.3.
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
