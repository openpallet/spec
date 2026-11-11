# Plik OpenPallet 1.2: opis dla integratora

Jeden plik JSON (UTF-8) na paletę: warstwy, ruchy robota, typy produktu i przekładek, ustawienia
chwytaka. Pola standardu plus jeden blok `x-palletdesigner` z danymi programu PalletDesigner.
Przykłady: `examples/*.json` pakietu Podstawy, te same w repozytorium openpallet/schema (`examples/1.2/`).
Wartości, które z nich dostają sterownik i robot: `pl/pierwsze-ruchy.md` pakietu Podstawy. Pakiety OpenPallet (Podstawy, sterowniki,
roboty, panel) są do pobrania bezpłatnie na openpallet.org.

## 1. Układ palety i numeracja

```
  O-------------------------------------> +Y   w prawo na rysunku, wzdłuż `width` palety
  |
  |                                              0  góra (-X)
  |      paleta jak na rysunku                   ^
  |      (widok z góry)            90 lewo <-----+-----> 270 prawo
  |                                              v
  |                                             180 dół (+X)
  v
  +X   w dół rysunku, wzdłuż `length` palety
```

- **O** = narożnik palety lewy górny na rysunku, punkt (0, 0) przy każdym rozmiarze palety.
- **X** w dół rysunku wzdłuż `pallet.length`, **Y** w prawo wzdłuż `pallet.width`, **Z** w górę
  (układ prawoskrętny). X biegnie wzdłuż `length` także wtedy, gdy `length` jest krótszy od `width`.
- **Z = 0 na górze pokładu palety**: warstwa 1 na gołej palecie leży na Z = 0. Wysokość palety
  (`pallet.height`) dodaje robot.
- **Kąt** = liczba całkowita stopni w lewo (przeciwnie do ruchu wskazówek zegara, patrząc z góry),
  0 = góra rysunku. +90 obraca +X (dół) na +Y (prawo). Kierunek `d` jako wektor: `(-cos d, -sin d)`.

| Stopnie | Na rysunku | Wektor (X, Y) |
|---|---|---|
| 0 | góra | (-1, 0) |
| 45 | góra-lewo | (-1, -1) |
| 90 | lewo | (0, -1) |
| 135 | dół-lewo | (+1, -1) |
| 180 | dół | (+1, 0) |
| 225 | dół-prawo | (+1, +1) |
| 270 | prawo | (0, +1) |
| 315 | góra-prawo | (-1, +1) |
| 900 | prosto z góry (bez najazdu z boku) | - |

| Liczba przy | znaczy | przykład |
|---|---|---|
| ścianie produktu, chwytaka, podajnika | kierunek, w który ściana patrzy w pozycji normalnej 0 elementu | etykieta na ścianie 270, zderzak podajnika na 180 |
| orientacji (produkt, chwytak przy pobraniu i przy odłożeniu) | kierunek, w który patrzy ściana 0 elementu | orientacja 270 (prawo) |
| stronie otwarcia chwytaka | kierunek tej strony w układzie chwytaka | 90, 270, 900 (bez strony otwarcia) |
| ścianach z etykietą | lista ścian z etykietą | `[0, 180]` |
| najeździe | kierunek ruchu grupy przy ostatnim najeździe, układ palety | 270 (prawo) |
| wyrównaniu | miejsce grupy wzdłuż długości chwytaka | 0 przy końcu 0, 180 przy końcu 180, 900 (środek) |

**900 i 999.** 900 = wartość neutralna ustawiona celowo: środek (wyrównanie), prosto z góry (najazd),
bez strony otwarcia (strona otwarcia chwytaka, np. chwytak od góry). 999 = **nieprzypisane**, brak
danych: wartość startowa każdej składowej kątowej w PLC i wartość pustego slotu (składowa PLC, kolumna
CSV). PalletDesigner nigdy nie zapisuje 999 dla ruchu, który policzył, a program robota na tej wartości
staje. W JSON ruch ma każdy kąt, nigdy `null`. Obie liczby leżą celowo poza 0..359. 0 jest zwykłą
wartością (góra, orientacja 0).

**Pozycje normalne (orientacja 0):**

- **Produkt:** `length` wzdłuż X, `width` wzdłuż Y. Ściana 0 patrzy w górę rysunku.
- **Chwytak:** długość wzdłuż X, **koniec 0** przy -X, **koniec 180** przy +X. Szerokość wzdłuż Y,
  **strona 90** przy -Y, **strona 270** przy +Y. Kierunek `g` chwytaka patrzy w kierunek palety
  `(g + gripper_place_orientation) mod 360`.
- **Podajnik:** zderzak na jego końcu 180 (`source.stop`). Orientacje produktu i chwytaka przy
  pobraniu liczy się w układzie podajnika.

## 2. Plik

Klucze najwyższego poziomu, w tej kolejności: `format`, `header`, `pallet`, `product`, `summary`,
`product_types`, `slip_sheet_types`, `stack`, `layouts`, `gripper`, `source`, `limits`,
`x-palletdesigner`. Każdy kąt i wyrównanie to liczba całkowita JSON (numer produktu i typu: tekst z
cyfr). Listy kierunków rosnąco.

### `format`, `header`

| Pole | Typ | Wartość |
|---|---|---|
| `format.name` | tekst | `"OpenPallet"` |
| `format.version` | tekst | `"1.2"`. Późniejsza wersja 1.x dodaje tylko klucze opcjonalne (rozdział 4) |
| `header.name` | tekst | nazwa receptury z `.json`, nazwa, którą porównuje sterownik (`sgName`) |
| `header.created` | tekst | ISO 8601, czas lokalny bez strefy |
| `header.producer` | tekst | `"PalletDesigner <wersja programu>"` |
| `header.checksum` | tekst | 8 cyfr szesnastkowych, CRC-32 (rozdział 4) |

### `pallet`, `product`, `summary`

| Pole | Typ | Wartość |
|---|---|---|
| `pallet.name` | tekst | nazwa palety |
| `pallet.type` | tekst | nazwa szablonu palety albo `Custom` |
| `pallet.length` | int, mm | wymiar wzdłuż X |
| `pallet.width` | int, mm | wymiar wzdłuż Y |
| `pallet.height` | int, mm | wysokość pustej palety |
| `product.name` | tekst | produkt palety (przy miksie: produkt, którego wariantami są typy) |
| `product.code` | tekst z cyfr | numer produktu, 0..2147483647 |
| `summary.mix` | bool | warstwy z kilku typów produktu |
| `summary.layers` | int | liczba warstw |
| `summary.products` | int | liczba produktów |
| `summary.moves` | int | liczba ruchów robota na całej palecie |
| `summary.slip_sheets` | int | liczba przekładek |
| `summary.height` | liczba, mm, jedno miejsce po przecinku | wysokość załadowanej palety od podłogi, z paletą (wymiar, nie Z) |
| `summary.mass` | int, g | masa wszystkich produktów |

### `product_types[]`

| Pole | Typ | Wartość |
|---|---|---|
| `id` | int | 1..`limits.types`, do niego odnosi się `type` produktu |
| `name`, `code` | tekst | nazwa i numer typu (cyfry) |
| `color` | tekst | `#RRGGBB`, tylko przy miksie |
| `length`, `width`, `height` | int, mm | produkt w orientacji 0: `length` wzdłuż X, `width` wzdłuż Y |
| `mass` | int, g | jeden produkt |
| `label_sides` | lista int | rosnący podzbiór `[0, 90, 180, 270]`: ściany z etykietą w orientacji 0 |
| `count` | int | produkty tego typu na palecie |

Paleta bez miksu: dokładnie jeden typ, `id` 1, z nazwą i numerem produktu. Jej produkty nie mają
klucza `type`. Rozmiar produktu jest jeden na paletę: typ 1 niesie wymiary, masę i etykiety dla
sterownika.

### `slip_sheet_types[]`, `stack[]`

Przekładka (ang. slip sheet) to arkusz na pokładzie palety, między warstwami albo na wierzchu, nie
zamiennik palety.

| Pole | Typ | Wartość |
|---|---|---|
| `slip_sheet_types[].id` | int | 1..10, bez powtórzeń, najwyżej 10 typów |
| `slip_sheet_types[].thickness` | liczba, mm | 0..1000, jedno miejsce po przecinku |
| `stack[].position` | int | 0 = sama paleta, 1..n = warstwa n od dołu, jeden wpis na pozycję |
| `stack[].layout` | int | `id` układu warstwy z `layouts`, brak na pozycji 0 |
| `stack[].slip_sheet` | int | `id` typu przekładki leżącej na tej pozycji (na pozycji 0: na pokładzie palety), brak = bez przekładki |

Typ przekładki niesie tylko grubość. Rozmiar i materiał przekładki należą do programu robota.

### `layouts[]` i ruchy

`layouts[]`: `{id, moves}`, w kolejności `id`. `id` = numer układu warstwy według pierwszego użycia
od dołu. `moves` = ruchy robota w kolejności odkładania, klucze w tej kolejności:

| Klucz | Typ | Wartości | Znaczenie |
|---|---|---|---|
| `placing_order` | int | 1.. | numer ruchu w warstwie |
| `products` | lista | tabela niżej | produkty ruchu w kolejności odkładania |
| `x_min`, `y_min`, `x_max`, `y_max` | int | mm | obrys grupy |
| `x_middle`, `y_middle` | liczba | zawsze .0 albo .5 | środek grupy (dla ludzi, sterownik dostaje środek obcięty do pełnych mm) |
| `size_x`, `size_y` | int | mm | wymiar grupy wzdłuż X, wzdłuż Y |
| `gripper_pick_orientation` | int | 0, 180 | orientacja chwytaka przy pobraniu (układ podajnika) |
| `gripper_place_orientation` | int | 0, 90, 180, 270 | orientacja chwytaka przy odłożeniu |
| `placing_direction` | int | 0, 45 .. 315, 900 = prosto z góry | kierunek ruchu grupy przy ostatnim najeździe (układ palety, nie obraca się z chwytakiem) |
| `gripper_align` | int | 0, 180, 900 | grupa wzdłuż długości chwytaka: przy końcu 0, przy końcu 180, 900 (środek) |
| `gripper_open_side` | int | 90, 270, 900 = bez strony otwarcia (np. chwytak od góry) | strona chwytaka, której szczęka się otwiera (układ chwytaka). Grupa dotyka szczęki po przeciwnej stronie |
| `products_at_pick` | lista `{product_id, orientation}` | `orientation` 0/90/180/270 | produkty w kolejności dojazdu do zderzaka podajnika, każdy z orientacją przy pobraniu (rozdział 7) |

Produkt ruchu (`products[]`), klucze w tej kolejności:

| Klucz | Typ | Znaczenie |
|---|---|---|
| `id` | int | numer produktu w warstwie |
| `orientation` | int | orientacja produktu na palecie: 0/90/180/270 |
| `x`, `y` | int, mm | narożnik produktu z mniejszym X i Y |
| `size_x`, `size_y` | int, mm | wymiar produktu wzdłuż X, wzdłuż Y |
| `type` | int | `id` typu produktu, tylko przy miksie |

### `gripper`, `source`, `limits`

Kopie ustawień stacji, z którymi powstał plik. Wszystkie trzy nieobecne, gdy projekt powstał bez
ustawień stacji.

| Pole | Wartość |
|---|---|
| `gripper.length`, `gripper.width` | obrys otwartego chwytaka, mm: koniec 0 → koniec 180, strona 90 → strona 270 |
| `gripper.outline_relevant` | `false`: obrys nie jest sprawdzany (chwytak od góry) |
| `gripper.group_length_min`, `_max`, `group_width_min`, `_max` | rozmiar grupy wzdłuż i w poprzek chwytaka, mm |
| `gripper.approach_clearance` | wolne miejsce obok grupy na najazd, mm |
| `gripper.allowed_pick_orientation`, `allowed_place_orientation` | dozwolone orientacje chwytaka przy pobraniu i przy odłożeniu |
| `gripper.allowed_open_side` | podzbiór `[90, 270]` |
| `gripper.allowed_align` | jedna z list `[0, 180, 900]`, `[180]`, `[0]`, `[0, 180]`, `[900]` |
| `source.stop` | 180: kierunek zderzaka podajnika w układzie podajnika |
| `source.allowed_product_orientation` | dozwolone orientacje produktu na podajniku |
| `limits.types`, `layers`, `layouts`, `moves`, `products` | rozmiary struktury receptury w sterowniku: typy, warstwy, układy, ruchy w układzie, produkty w ruchu |
| `limits.name_length`, `recipe_name_length` | długość tekstów w sterowniku: nazwy produktu, typu i palety oraz nazwa receptury |

### `x-palletdesigner`

Blok danych, których potrzebuje tylko program PalletDesigner: jego ustawienia stacji, nazwy plików
układów, pola pomocnicze ruchów (równoległe do `layouts[].moves[]`), a w kopii palety z zamówienia
`x-palletdesigner.order` (nazwa zamówienia, ramka, lokalizacje). Czytnik spoza programu go pomija.
Suma kontrolna go obejmuje, więc zmiana w nim też wymaga nowej sumy. Plik z `x-palletdesigner.order`
wczytuje się tylko jako paleta zamówienia.

## 3. Wysokości

Dół warstwy K, od góry pokładu palety:

`Z(K) = (K - 1) · h + suma grubości przekładek na pozycjach 0..K-1`

- `h` = `height` typu produktu 1 (jeden rozmiar produktu na paletę).
- Grubość przekładki na pozycji p = `thickness` typu `stack[p].slip_sheet`. Pozycja bez
  `slip_sheet` = 0.

Przykład: przekładka 2,7 mm na pokładzie palety: warstwa 1 leży na Z = 2,7 mm (`LayerBottomZ_x10`
w CSV: `27`). `summary.height` = `pallet.height` + `summary.layers` · h + grubości wszystkich
przekładek (pozycje 0..`summary.layers`).

**Grubość przekładki:** o ile przekładka podnosi następną warstwę. Ranty i krawędzie, na których
nie stoi produkt, się nie liczą.

## 4. Rozpoznanie wersji i suma kontrolna

| Plik projektu palety, który | to | czyta się |
|---|---|---|
| ma `format.name` = `"OpenPallet"` i `format.version` `"1.2"` albo późniejsze 1.x | plik OpenPallet | wprost, klucze nieznane czytelnikowi pomija się |
| cokolwiek innego | - | odmowa |

Wersja w `format.version` to wersja standardu OpenPallet. Późniejsza wersja 1.x dodaje tylko klucze opcjonalne.
Inną wersję główną się odrzuca. Panel szablonu PLC PalletDesigner 175 lub starszego odmawia pliku OpenPallet, bo
plik nie ma sekcji `metadata`.

**Suma kontrolna `header.checksum`:**

1. Kopia całego dokumentu bez `header.checksum` (z `x-palletdesigner`).
2. Liczby całkowite zapisane z `.0` jako całkowite: `165.0` → `165`. Pozostałe liczby jak `repr()`
   w Pythonie: najkrótszy zapis, poniżej 0,0001 z wykładnikiem (`1e-05`).
3. Tekst kanoniczny: JSON bez spacji i nowych linii, separatory `,` i `:`, klucze każdego obiektu
   posortowane jak tekst (`"10"` przed `"2"`), listy w kolejności z pliku, znaki spoza ASCII jako
   `\uXXXX` (małe litery hex, powyżej 0xFFFF para surogatów).
4. CRC-32 (jak `zlib.crc32` i ZIP) z bajtów UTF-8 tego tekstu, który po kroku 3 jest czystym ASCII.
5. Wynik: 8 cyfr szesnastkowych małymi literami, z zerami na początku.

Tekst kanoniczny powstaje z przeczytanego dokumentu, nie z bajtów pliku (plik ma wcięcia).

```python
import json
import zlib

def _whole_floats_as_ints(value):
    if isinstance(value, float) and value.is_integer():
        return int(value)
    if isinstance(value, dict):
        return {key: _whole_floats_as_ints(item) for key, item in value.items()}
    if isinstance(value, list):
        return [_whole_floats_as_ints(item) for item in value]
    return value

def openpallet_checksum(document):
    copied = json.loads(json.dumps(document))
    copied.get("header", {}).pop("checksum", None)
    canonical = json.dumps(_whole_floats_as_ints(copied), sort_keys=True, separators=(",", ":"))
    return format(zlib.crc32(canonical.encode("utf-8")) & 0xFFFFFFFF, "08x")
```

Ta sama funkcja w JavaScript: `pl/PRZEWODNIK.md` pakietu `wincc-unified`.

## 5. Składowe PLC `tPlacingData` i `tRecipeData` 2.0

Typy sterownika (źródła TIA Portal: pakiet `siemens`). Każda składowa kątowa jest typu Int z
wartością startową 999 (nieprzypisane).

**Ruch `tPlacingData`:** `asLayerData[n].asPlacing[m]` ← ruch `moves[m-1]` układu o `id` n.

| Składowa | Typ | Wartości | Z ruchu |
|---|---|---|---|
| `ai16BoxPickRotation[1..14]` | Array of Int, start 999 | 0/90/180/270, 999 = brak produktu | `products_at_pick[k-1].orientation`, dalej 999 |
| `ai8Btype[1..14]` | Array of SInt | 1..typy, 0 = brak | `type` produktów w kolejności pobrania (produkt wg `product_id`), bez miksu 0 |
| `u16Length` | UInt | mm wzdłuż długości chwytaka | `size_x` przy `gripper_place_orientation` 0 albo 180, inaczej `size_y` |
| `u16Width` | UInt | mm w poprzek chwytaka | drugi wymiar |
| `i16MiddleY` | Int | mm | `(y_min + y_max) / 2` obcięte w stronę zera |
| `i16MiddleX` | Int | mm | `(x_min + x_max) / 2` obcięte w stronę zera |
| `i16MidYCorrect`, `i16MidXCorrect` | Int | mm | 0 w plikach, korekta z panelu |
| `i16GripperPickRotation` | Int, start 999 | 0, 180 | `gripper_pick_orientation` |
| `i16GripperPlaceRotation` | Int, start 999 | 0, 90, 180, 270 | `gripper_place_orientation` |
| `i16GripperOpenSide` | Int, start 999 | 90, 270, 900 = bez strony otwarcia | `gripper_open_side` |
| `i16GripperAlign` | Int, start 999 | 0, 180, 900 | `gripper_align` |
| `i16PlacingDirection` | Int, start 999 | 0, 45 .. 315, 900 = prosto z góry | `placing_direction` |

Nazwy składowych PLC to nazwy z szablonu sterownika: `i16GripperPlaceRotation` niesie
`gripper_place_orientation`, `ai16BoxPickRotation` niesie orientacje z `products_at_pick`.

**Koniec warstwy:** slot ruchu jest pusty, gdy `ai16BoxPickRotation[1]` = 999. Pierwszy pusty slot
kończy warstwę, a pusta receptura ma `asLayerData[1].asPlacing[1].ai16BoxPickRotation[1]` = 999. Po
`m` ruchach układu n zapis `asLayerData[n].asPlacing[m+1].ai16BoxPickRotation[1] := 999`, a nieużyte
sloty produktów zapisanego ruchu = 999. `asPlacing` ma `limits.moves + 1` slotów, więc pełna warstwa
też ma swój pusty slot. Liczby ruchów nie ma w żadnej składowej.

**Receptura `tRecipeData`:**

| Składowa | Typ | Z pliku |
|---|---|---|
| `sgName` | String[90] | `header.name` (teksty do sterownika bez polskich znaków diakrytycznych) |
| `sProductData.i32Id`, `sgName` | DInt, String[30] | `product.code`, `product.name` |
| `sProductData.u16Length`, `u16Width`, `u16Height`, `u32Weight` | UInt, UDInt | typ produktu 1: `length`, `width`, `height` (mm), `mass` (g) |
| `sProductData.b8LabelSides` | Byte | bajt etykiet typu 1: **bit n = ściana 90·n** (bit0 = 0, bit1 = 90, bit2 = 180, bit3 = 270), suma `1 << (d / 90)` po `label_sides`, np. `[270]` → `16#08` |
| `sProductData.asTypeData[n]` | Struct | typ o `id` n, tylko przy `summary.mix`: `boIsUsed` TRUE, `sgName` = `name`, `i32Id` = `code`, `w32Color` = `16#FFRRGGBB` z `color`, `i8Location` 0 (zamówienie: `x-palletdesigner.order.input_location`). Nieużyte: FALSE, '', 0, 0, 0 |
| `sPalletData.sgName`, `u16Length`, `u16Width`, `u16Height` | String[30], UInt | `pallet.name`, `length`, `width`, `height` |
| `sSpacerData.au16Thickness[1..10]` | Array of UInt, 0,1 mm | indeks = `id` z `slip_sheet_types`: `thickness` · 10 zaokrąglone, `id` bez typu: 0 |
| `sSpacerData.ai8SpacerTypePositions[0..64]` | Array of SInt | `stack[p].slip_sheet`, brak → 0 |
| `sSummary.boIsMix`, `u16LayerCount`, `u16ProductCount`, `u16PlacingCount` | Bool, UInt | `summary.mix`, `layers`, `products`, `moves` |
| `sSummary.u16Height` | UInt, 0,1 mm | `summary.height` · 10 zaokrąglone |
| `sSummary.u32Weight` | UDInt, g | `summary.mass` |
| `sSummary.i8Location` | SInt | 0 (zamówienie: `x-palletdesigner.order.location`) |
| `au8LayerPositions[1..64]` | Array of USInt | `stack[p].layout` |
| `asLayerData[1..5]` | Struct | układ o `id` n: korekty `i16WholeLayerYCorrect`, `i16WholeLayerXCorrect` (0 w plikach) i `asPlacing[1..65]` |

Grubość przekładki na pozycji p w sterowniku: `au16Thickness[ai8SpacerTypePositions[p]]` (typ 0 =
brak przekładki). Granice tablic = `limits` pliku. Liczby w tabelach to wartości domyślne.
Zaokrąglenie dziesiątych: połówka w górę (`Math.round` panelu).

## 6. Kolumny CSV 2.0

Jeden plik CSV na paletę, w układzie palety. Linie legendy zaczynają się od `#`:

- dwie pierwsze linie: `# PalletDesigner - Pallet Placings CSV Export` i `# Format: 2.0`,
- legenda: układ z rozdziału 1, kąty w stopniach w lewo patrząc z góry, 999 = nieprzypisane,
- `# Slip sheet types:` z linią na typ przekładki, np. `#   1: Thickness_x10 27`, potem
  `# Slip sheets` z `PalletHasSlipSheet` i typem przekładki na każdej pozycji,
- `LayerBottomZ_x10`: dół każdej warstwy od góry pokładu palety, w 0,1 mm (wysokość palety osobno,
  `PalletHeight_x10`).

Potem wiersz nagłówka kolumn i jeden wiersz na ruch układu (układy w kolejności `id`):

| Kolumna | Nazwa | Jednostka, wartości |
|---|---|---|
| 1 | `layer` | `id` układu |
| 2 | `used_in_layers` | pozycje stosu z tym układem, oddzielone spacją (`1 3 5`) |
| 3 | `placing_order` | numer ruchu w warstwie |
| 4 | `PacketLength_x10` | 0,1 mm wzdłuż długości chwytaka |
| 5 | `PacketWidth_x10` | 0,1 mm w poprzek chwytaka |
| 6 | `MiddleX_x10` | 0,1 mm, X środka grupy (dokładnie, bez obcinania: 250,5 mm = `2505`) |
| 7 | `MiddleY_x10` | 0,1 mm, Y środka grupy |
| 8 | `GripperPickOrientation` | 0, 180 |
| 9 | `GripperPlaceOrientation` | 0, 90, 180, 270 |
| 10 | `GripperOpenSide` | 90, 270, 900 = bez strony otwarcia |
| 11 | `GripperAlign` | 0, 180, 900 |
| 12 | `PlacingDirection` | 0, 45 .. 315, 900 = prosto z góry |
| 13 | `PacketWeight_x10` | 0,1 kg |
| 14 | `NumOfProd` | liczba produktów w ruchu |
| 15-28 | `ProductPickOrientation1..14` | 0/90/180/270, **999 = pusty** |
| 29-42 | `ProductType1..14` | typ produktu, 0 = brak |

Za wierszami ruchów: sekcja na każdy układ (linia `#` z jego numerem), jeden wiersz na produkt,
sześć kolumn: `X_x10`, `Y_x10` (środek produktu w układzie palety, 0,1 mm), `Z_x10` = 0,
`Rx_deg` = 0, `Ry_deg` = 0, `Rz_deg` = `orientation` produktu w stopniach w lewo.

## 7. Zasady orientacji

**Orientacja produktu na palecie:**

`orientation = orientacja przy pobraniu + gripper_place_orientation + gripper_pick_orientation  (mod 360)`

gdzie orientacja przy pobraniu = `orientation` tego produktu (`product_id` = `id`) w
`products_at_pick`. Przykład: orientacja przy pobraniu 270, odłożenie 90, pobranie 0:
270 + 90 + 0 = 360 → 0. Drugi: 90 + 90 + 0 = 180.

**Pierwszy produkt.** `products_at_pick[0]` to produkt przy zderzaku podajnika. Lista biegnie od
zderzaka wstecz wzdłuż długości chwytaka. Zderzak podajnika jest na jego końcu 180, więc:

- przy pobraniu zderzak trafia na koniec chwytaka `(180 - gripper_pick_orientation) mod 360`
  (orientacja 0: koniec 180, orientacja 180: koniec 0),
- przy odłożeniu ten koniec patrzy w kierunek palety
  `(180 - gripper_pick_orientation + gripper_place_orientation) mod 360`. **Pierwszy produkt to
  produkt grupy leżący najdalej w tym kierunku**.

| `gripper_pick_orientation` | odłożenie 0 | 90 | 180 | 270 |
|---|---|---|---|---|
| 0 | dół | prawo | góra | lewo |
| 180 | góra | lewo | dół | prawo |

(pierwszy produkt = najdalej w podanym kierunku rysunku)

**Obrys i etykiety produktu.** Orientacja 0 albo 180: wymiar wzdłuż X = `length` typu, wzdłuż Y =
`width`. Orientacja 90 albo 270: odwrotnie. Ściana `f` produktu patrzy na palecie w kierunek
`(f + orientation) mod 360`.
