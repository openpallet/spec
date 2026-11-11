# The OpenPallet 1.2 file: a description for the integrator

One JSON file (UTF-8) per pallet: the layers, the robot moves, the product and slip sheet types, the gripper
settings. The fields of the standard plus one block `x-palletdesigner` with the data of the PalletDesigner program.
Examples: `examples/*.json` of the Core package, the same in the openpallet/schema repository (`examples/1.2/`).
The values the controller and the robot get from them: `first-moves.md` of the Core package. The OpenPallet packages (Core,
controllers, robots, panel) can be downloaded free of charge at openpallet.org.

## 1. The pallet frame and the numbering

```
  O-------------------------------------> +Y   right on the drawing, along the pallet's `width`
  |
  |                                              0  up (-X)
  |      pallet as on the drawing                ^
  |      (seen from above)         90 left <-----+-----> 270 right
  |                                              v
  |                                             180 down (+X)
  v
  +X   down the drawing, along the pallet's `length`
```

- **O** = the top left corner of the pallet on the drawing, the point (0, 0) for every pallet size.
- **X** down the drawing along `pallet.length`, **Y** to the right along `pallet.width`, **Z** up
  (a right-handed frame). X runs along `length` also when `length` is shorter than `width`.
- **Z = 0 on top of the pallet deck**: layer 1 on a bare pallet lies at Z = 0. The robot adds the height of the
  pallet (`pallet.height`).
- **Angle** = a whole number of degrees to the left (counter-clockwise, seen from above), 0 = the top of the
  drawing. +90 turns +X (down) onto +Y (right). Direction `d` as a vector: `(-cos d, -sin d)`.

| Degrees | On the drawing | Vector (X, Y) |
|---|---|---|
| 0 | up | (-1, 0) |
| 45 | up-left | (-1, -1) |
| 90 | left | (0, -1) |
| 135 | down-left | (+1, -1) |
| 180 | down | (+1, 0) |
| 225 | down-right | (+1, +1) |
| 270 | right | (0, +1) |
| 315 | up-right | (-1, +1) |
| 900 | straight down from above (no approach from the side) | - |

| A number on | means | example |
|---|---|---|
| a face of a product, the gripper, the infeed | the direction the face looks in the element's normal pose 0 | label on face 270, infeed stop at 180 |
| an orientation (product, gripper at the pick and at the place) | the direction face 0 of the element looks | orientation 270 (right) |
| the gripper's open side | the direction of that side in the gripper's frame | 90, 270, 900 (no open side) |
| the faces with a label | the list of the faces with a label | `[0, 180]` |
| the approach | the direction the group travels in its last approach, pallet frame | 270 (right) |
| the alignment | where the group lies along the gripper's length | 0 at end 0, 180 at end 180, 900 (centre) |

**900 and 999.** 900 = a neutral value set on purpose: the centre in the alignment, straight down from
above in the approach, no open side for the gripper's open side (e.g. a gripper from above).
999 = **not assigned**, no data: the start value of every angle member in the PLC and the value of an
empty slot (PLC member, CSV column). PalletDesigner never writes 999 for a move it computed, and a robot
program stops on it. In JSON a move carries every angle, never `null`. Both numbers lie outside 0..359
on purpose. 0 is an ordinary value (up, orientation 0).

**Normal poses (orientation 0):**

- **Product:** `length` along X, `width` along Y. Face 0 looks up the drawing.
- **Gripper:** its length along X, **end 0** at -X, **end 180** at +X. Its width along Y,
  **side 90** at -Y, **side 270** at +Y. A direction `g` of the gripper looks in the pallet direction
  `(g + gripper_place_orientation) mod 360`.
- **Infeed:** the stop at its end 180 (`source.stop`). The orientations of the product and of the gripper at the
  pick count in the infeed's frame.

## 2. The file

The top-level keys, in this order: `format`, `header`, `pallet`, `product`, `summary`,
`product_types`, `slip_sheet_types`, `stack`, `layouts`, `gripper`, `source`, `limits`,
`x-palletdesigner`. Every angle and alignment is a JSON integer (the number of a product and of a type: a text of
digits). Lists of directions in ascending order.

### `format`, `header`

| Field | Type | Value |
|---|---|---|
| `format.name` | text | `"OpenPallet"` |
| `format.version` | text | `"1.2"`. A later 1.x version adds only optional keys (section 4) |
| `header.name` | text | the recipe name with `.json`, the name the controller compares (`sgName`) |
| `header.created` | text | ISO 8601, local time without a zone |
| `header.producer` | text | `"PalletDesigner <program version>"` |
| `header.checksum` | text | 8 hexadecimal digits, CRC-32 (section 4) |

### `pallet`, `product`, `summary`

| Field | Type | Value |
|---|---|---|
| `pallet.name` | text | name of the pallet |
| `pallet.type` | text | name of the pallet template or `Custom` |
| `pallet.length` | int, mm | size along X |
| `pallet.width` | int, mm | size along Y |
| `pallet.height` | int, mm | height of the empty pallet |
| `product.name` | text | the product of the pallet (in a mix: the product whose variants the types are) |
| `product.code` | text of digits | product number, 0..2147483647 |
| `summary.mix` | bool | layers of several product types |
| `summary.layers` | int | number of layers |
| `summary.products` | int | number of products |
| `summary.moves` | int | number of robot moves on the whole pallet |
| `summary.slip_sheets` | int | number of slip sheets |
| `summary.height` | number, mm, one decimal place | height of the loaded pallet from the floor, with the pallet (a size, not Z) |
| `summary.mass` | int, g | mass of all products |

### `product_types[]`

| Field | Type | Value |
|---|---|---|
| `id` | int | 1..`limits.types`, the `type` of a product refers to it |
| `name`, `code` | text | name and number of the type (digits) |
| `color` | text | `#RRGGBB`, only in a mix |
| `length`, `width`, `height` | int, mm | the product at orientation 0: `length` along X, `width` along Y |
| `mass` | int, g | one product |
| `label_sides` | list of int | ascending subset of `[0, 90, 180, 270]`: the faces with a label at orientation 0 |
| `count` | int | products of this type on the pallet |

A pallet without a mix: exactly one type, `id` 1, with the name and number of the product. Its products have no
key `type`. The size of the product is one per pallet: type 1 carries the sizes, the mass and the labels for the
controller.

### `slip_sheet_types[]`, `stack[]`

A slip sheet is a sheet on the pallet deck, between layers or on top, not a replacement for the
pallet.

| Field | Type | Value |
|---|---|---|
| `slip_sheet_types[].id` | int | 1..10, no repeats, at most 10 types |
| `slip_sheet_types[].thickness` | number, mm | 0..1000, one decimal place |
| `stack[].position` | int | 0 = the pallet alone, 1..n = layer n from the bottom, one entry per position |
| `stack[].layout` | int | `id` of the layer layout from `layouts`, absent at position 0 |
| `stack[].slip_sheet` | int | `id` of the type of the slip sheet lying on this position (at position 0: on the pallet deck), absent = no slip sheet |

A slip sheet type carries only the thickness. The size and the material of a slip sheet belong to the robot
program.

### `layouts[]` and the moves

`layouts[]`: `{id, moves}`, in `id` order. `id` = the number of the layer layout by first use
from the bottom. `moves` = the robot moves in placing order, keys in this order:

| Key | Type | Values | Meaning |
|---|---|---|---|
| `placing_order` | int | 1.. | number of the move in the layer |
| `products` | list | table below | the products of the move in placing order |
| `x_min`, `y_min`, `x_max`, `y_max` | int | mm | outline of the group |
| `x_middle`, `y_middle` | number | always .0 or .5 | centre of the group (for people, the controller gets the centre cut to whole mm) |
| `size_x`, `size_y` | int | mm | size of the group along X, along Y |
| `gripper_pick_orientation` | int | 0, 180 | orientation of the gripper at the pick (infeed frame) |
| `gripper_place_orientation` | int | 0, 90, 180, 270 | orientation of the gripper at the place |
| `placing_direction` | int | 0, 45 .. 315, 900 = straight down from above | direction the group travels in its last approach (pallet frame, does not turn with the gripper) |
| `gripper_align` | int | 0, 180, 900 | the group along the gripper's length: at end 0, at end 180, 900 (centre) |
| `gripper_open_side` | int | 90, 270, 900 = no open side (e.g. a gripper from above) | the side of the gripper whose jaw opens (gripper frame). The group touches the jaw on the opposite side |
| `products_at_pick` | list `{product_id, orientation}` | `orientation` 0/90/180/270 | the products in the order they reach the infeed stop, each with its orientation at the pick (section 7) |

A product of a move (`products[]`), keys in this order:

| Key | Type | Meaning |
|---|---|---|
| `id` | int | number of the product in the layer |
| `orientation` | int | orientation of the product on the pallet: 0/90/180/270 |
| `x`, `y` | int, mm | the corner of the product with the smaller X and Y |
| `size_x`, `size_y` | int, mm | size of the product along X, along Y |
| `type` | int | `id` of the product type, only in a mix |

### `gripper`, `source`, `limits`

Copies of the station settings the file was made with. All three absent when the project was made without
station settings.

| Field | Value |
|---|---|
| `gripper.length`, `gripper.width` | outline of the open gripper, mm: end 0 → end 180, side 90 → side 270 |
| `gripper.outline_relevant` | `false`: the outline is not checked (gripper from above) |
| `gripper.group_length_min`, `_max`, `group_width_min`, `_max` | size of the group along and across the gripper, mm |
| `gripper.approach_clearance` | free space beside the group for the approach, mm |
| `gripper.allowed_pick_orientation`, `allowed_place_orientation` | allowed orientations of the gripper at the pick and at the place |
| `gripper.allowed_open_side` | subset of `[90, 270]` |
| `gripper.allowed_align` | one of the lists `[0, 180, 900]`, `[180]`, `[0]`, `[0, 180]`, `[900]` |
| `source.stop` | 180: direction of the infeed stop in the infeed's frame |
| `source.allowed_product_orientation` | product orientations allowed on the infeed |
| `limits.types`, `layers`, `layouts`, `moves`, `products` | sizes of the recipe structure in the controller: types, layers, layouts, moves in a layout, products in a move |
| `limits.name_length`, `recipe_name_length` | length of the texts in the controller: names of the product, the type and the pallet, and the recipe name |

### `x-palletdesigner`

A block of data only the PalletDesigner program needs: its station settings, the file names of the layouts,
helper fields of the moves (parallel to `layouts[].moves[]`), and in the copy of a pallet from an order
`x-palletdesigner.order` (order name, frame type, locations). A reader outside the program skips it.
The checksum covers it, so a change in it also needs a new checksum. A file with `x-palletdesigner.order`
loads only as the pallet of an order.

## 3. Heights

The bottom of layer K, from the top of the pallet deck:

`Z(K) = (K - 1) · h + sum of the slip sheet thicknesses at positions 0..K-1`

- `h` = `height` of product type 1 (one product size per pallet).
- The thickness of the slip sheet at position p = `thickness` of type `stack[p].slip_sheet`. A position without
  `slip_sheet` = 0.

Example: a 2.7 mm slip sheet on the pallet deck: layer 1 lies at Z = 2.7 mm (`LayerBottomZ_x10`
in the CSV: `27`). `summary.height` = `pallet.height` + `summary.layers` · h + the thicknesses of all slip
sheets (positions 0..`summary.layers`).

**Slip sheet thickness:** how much the slip sheet raises the next layer. Rims and edges no product stands on do
not count.

## 4. Version recognition and the checksum

| A pallet project file that | is | is read |
|---|---|---|
| has `format.name` = `"OpenPallet"` and `format.version` `"1.2"` or a later 1.x | an OpenPallet file | directly, keys the reader does not know skipped |
| anything else | - | refused |

The version in `format.version` is the version of the OpenPallet standard. A later 1.x version adds only optional
keys, another major version is refused. A panel of a PalletDesigner PLC template 175 or older refuses an
OpenPallet file, because the file has no `metadata` section.

**The checksum `header.checksum`:**

1. A copy of the whole document without `header.checksum` (with `x-palletdesigner`).
2. Whole numbers written with `.0` as integers: `165.0` → `165`. Other numbers as Python's `repr()`:
   the shortest form, below 0.0001 with an exponent (`1e-05`).
3. The canonical text: JSON without spaces and new lines, separators `,` and `:`, the keys of every object
   sorted as text (`"10"` before `"2"`), lists in the order of the file, characters outside ASCII as
   `\uXXXX` (lower-case hex letters, above 0xFFFF a surrogate pair).
4. CRC-32 (as `zlib.crc32` and ZIP) of the UTF-8 bytes of this text, which after step 3 is pure ASCII.
5. The result: 8 hexadecimal digits in lower case, with leading zeros.

The canonical text is made from the document as read, not from the bytes of the file (the file is indented).

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

The same function in JavaScript: `GUIDE.md` of the `wincc-unified` package.

## 5. The PLC members `tPlacingData` and `tRecipeData` 2.0

The controller's types (TIA Portal sources: package `siemens`). Every angle member is of type Int with
the start value 999 (not assigned).

**Move `tPlacingData`:** `asLayerData[n].asPlacing[m]` ← move `moves[m-1]` of the layout with `id` n.

| Member | Type | Values | From the move |
|---|---|---|---|
| `ai16BoxPickRotation[1..14]` | Array of Int, start 999 | 0/90/180/270, 999 = no product | `products_at_pick[k-1].orientation`, then 999 |
| `ai8Btype[1..14]` | Array of SInt | 1..types, 0 = none | `type` of the products in pick order (the product by `product_id`), without a mix 0 |
| `u16Length` | UInt | mm along the gripper's length | `size_x` at `gripper_place_orientation` 0 or 180, else `size_y` |
| `u16Width` | UInt | mm across the gripper | the other size |
| `i16MiddleY` | Int | mm | `(y_min + y_max) / 2` cut towards zero |
| `i16MiddleX` | Int | mm | `(x_min + x_max) / 2` cut towards zero |
| `i16MidYCorrect`, `i16MidXCorrect` | Int | mm | 0 in files, a correction from the panel |
| `i16GripperPickRotation` | Int, start 999 | 0, 180 | `gripper_pick_orientation` |
| `i16GripperPlaceRotation` | Int, start 999 | 0, 90, 180, 270 | `gripper_place_orientation` |
| `i16GripperOpenSide` | Int, start 999 | 90, 270, 900 = no open side | `gripper_open_side` |
| `i16GripperAlign` | Int, start 999 | 0, 180, 900 | `gripper_align` |
| `i16PlacingDirection` | Int, start 999 | 0, 45 .. 315, 900 = straight down from above | `placing_direction` |

The names of the PLC members are those of the controller template: `i16GripperPlaceRotation` carries
`gripper_place_orientation`, `ai16BoxPickRotation` carries the orientations from `products_at_pick`.

**End of a layer:** a move slot is empty when `ai16BoxPickRotation[1]` = 999. The first empty slot
ends the layer, and an empty recipe has `asLayerData[1].asPlacing[1].ai16BoxPickRotation[1]` = 999. After the
`m` moves of layout n comes the write `asLayerData[n].asPlacing[m+1].ai16BoxPickRotation[1] := 999`, and the
unused product slots of a written move = 999. `asPlacing` has `limits.moves + 1` slots, so a full layer
also has its empty slot. The number of moves is in no member.

**Recipe `tRecipeData`:**

| Member | Type | From the file |
|---|---|---|
| `sgName` | String[90] | `header.name` (texts for the controller without Polish diacritics) |
| `sProductData.i32Id`, `sgName` | DInt, String[30] | `product.code`, `product.name` |
| `sProductData.u16Length`, `u16Width`, `u16Height`, `u32Weight` | UInt, UDInt | product type 1: `length`, `width`, `height` (mm), `mass` (g) |
| `sProductData.b8LabelSides` | Byte | the label byte of type 1: **bit n = face 90·n** (bit0 = 0, bit1 = 90, bit2 = 180, bit3 = 270), the sum of `1 << (d / 90)` over `label_sides`, e.g. `[270]` → `16#08` |
| `sProductData.asTypeData[n]` | Struct | the type with `id` n, only when `summary.mix`: `boIsUsed` TRUE, `sgName` = `name`, `i32Id` = `code`, `w32Color` = `16#FFRRGGBB` from `color`, `i8Location` 0 (order: `x-palletdesigner.order.input_location`). Unused: FALSE, '', 0, 0, 0 |
| `sPalletData.sgName`, `u16Length`, `u16Width`, `u16Height` | String[30], UInt | `pallet.name`, `length`, `width`, `height` |
| `sSpacerData.au16Thickness[1..10]` | Array of UInt, 0.1 mm | index = `id` from `slip_sheet_types`: `thickness` · 10 rounded, an `id` without a type: 0 |
| `sSpacerData.ai8SpacerTypePositions[0..64]` | Array of SInt | `stack[p].slip_sheet`, absent → 0 |
| `sSummary.boIsMix`, `u16LayerCount`, `u16ProductCount`, `u16PlacingCount` | Bool, UInt | `summary.mix`, `layers`, `products`, `moves` |
| `sSummary.u16Height` | UInt, 0.1 mm | `summary.height` · 10 rounded |
| `sSummary.u32Weight` | UDInt, g | `summary.mass` |
| `sSummary.i8Location` | SInt | 0 (order: `x-palletdesigner.order.location`) |
| `au8LayerPositions[1..64]` | Array of USInt | `stack[p].layout` |
| `asLayerData[1..5]` | Struct | the layout with `id` n: the corrections `i16WholeLayerYCorrect`, `i16WholeLayerXCorrect` (0 in files) and `asPlacing[1..65]` |

The thickness of the slip sheet at position p in the controller: `au16Thickness[ai8SpacerTypePositions[p]]` (type 0 =
no slip sheet). The bounds of the arrays = the `limits` of the file. The numbers in the tables are the default
values. Rounding of tenths: half up (the panel's `Math.round`).

## 6. The CSV columns 2.0

One CSV file per pallet, in the pallet frame. The legend lines start with `#`:

- the first two lines: `# PalletDesigner - Pallet Placings CSV Export` and `# Format: 2.0`,
- the legend: the frame of section 1, angles in degrees counter-clockwise seen from above, 999 = not assigned,
- `# Slip sheet types:` with a line per slip sheet type, e.g. `#   1: Thickness_x10 27`, then
  `# Slip sheets` with `PalletHasSlipSheet` and the slip sheet type at every position,
- `LayerBottomZ_x10`: the bottom of every layer from the top of the pallet deck, in 0.1 mm (the pallet height
  separately, `PalletHeight_x10`).

Then the row of column headers and one row per move of a layout (the layouts in `id` order):

| Column | Name | Unit, values |
|---|---|---|
| 1 | `layer` | `id` of the layout |
| 2 | `used_in_layers` | the stack positions with this layout, separated by a space (`1 3 5`) |
| 3 | `placing_order` | number of the move in the layer |
| 4 | `PacketLength_x10` | 0.1 mm along the gripper's length |
| 5 | `PacketWidth_x10` | 0.1 mm across the gripper |
| 6 | `MiddleX_x10` | 0.1 mm, X of the centre of the group (exact, not cut: 250.5 mm = `2505`) |
| 7 | `MiddleY_x10` | 0.1 mm, Y of the centre of the group |
| 8 | `GripperPickOrientation` | 0, 180 |
| 9 | `GripperPlaceOrientation` | 0, 90, 180, 270 |
| 10 | `GripperOpenSide` | 90, 270, 900 = no open side |
| 11 | `GripperAlign` | 0, 180, 900 |
| 12 | `PlacingDirection` | 0, 45 .. 315, 900 = straight down from above |
| 13 | `PacketWeight_x10` | 0.1 kg |
| 14 | `NumOfProd` | number of products in the move |
| 15-28 | `ProductPickOrientation1..14` | 0/90/180/270, **999 = empty** |
| 29-42 | `ProductType1..14` | product type, 0 = none |

After the move rows: a section per layout (a `#` line with its number), one row per product,
six columns: `X_x10`, `Y_x10` (the centre of the product in the pallet frame, 0.1 mm), `Z_x10` = 0,
`Rx_deg` = 0, `Ry_deg` = 0, `Rz_deg` = the product's `orientation` in degrees counter-clockwise.

## 7. The orientation rules

**Orientation of a product on the pallet:**

`orientation = orientation at the pick + gripper_place_orientation + gripper_pick_orientation  (mod 360)`

where the orientation at the pick = the `orientation` of this product (`product_id` = `id`) in
`products_at_pick`. Example: orientation at the pick 270, place 90, pick 0:
270 + 90 + 0 = 360 → 0. A second one: 90 + 90 + 0 = 180.

**The first product.** `products_at_pick[0]` is the product at the infeed stop. The list runs from the
stop backwards along the gripper's length. The infeed stop is at its end 180, so:

- at the pick the stop comes to the gripper's end `(180 - gripper_pick_orientation) mod 360`
  (orientation 0: end 180, orientation 180: end 0),
- at the place this end looks in the pallet direction
  `(180 - gripper_pick_orientation + gripper_place_orientation) mod 360`. **The first product is the
  product of the group lying farthest in this direction**.

| `gripper_pick_orientation` | place 0 | 90 | 180 | 270 |
|---|---|---|---|---|
| 0 | down | right | up | left |
| 180 | up | left | down | right |

(the first product = the farthest in the given direction of the drawing)

**Outline and labels of a product.** Orientation 0 or 180: the size along X = the type's `length`, along Y =
`width`. Orientation 90 or 270: the other way round. Face `f` of the product looks on the pallet in the direction
`(f + orientation) mod 360`.
