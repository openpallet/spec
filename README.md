# OpenPallet

OpenPallet is an open exchange format for palletizing data. One JSON file, the OpenPallet file, describes the
pallet, the product types, the slip sheets, the stack of layers and every robot move, all in one pallet frame
and with every orientation and direction in whole degrees counter-clockwise from the top of the drawing. The same values have a fixed
layout for PLCs and robots (PLC structures and CSV): 900 for centred, for straight down from above and for a
gripper without an open side, 999 where nothing is assigned.

**Status: OpenPallet 1.2 · text and cards in English, with their Polish translation.**

Website: <https://openpallet.org>

## Contents

| Path | What |
|---|---|
| [`specification/en/file-description.md`](specification/en/file-description.md) | the specification: the frame and the numbering, every field of the file, heights, version recognition and the checksum (with its code), the PLC members, the CSV columns, the orientation rules |
| [`cards/en/`](cards/en/index.html) | the concept cards, one A4 page per concept: HTML pages, SVG drawings (`rysunki/`), one PDF of all cards (`OpenPallet-cards-en.pdf`), an index (`karty.json`) |
| [`specification/pl/opis-pliku.md`](specification/pl/opis-pliku.md), [`cards/pl/`](cards/pl/index.html) | the same in Polish (`OpenPallet-karty-pl.pdf`) |
| [`CHANGELOG.md`](CHANGELOG.md) | the editions of the standard |

The cards:

1. The station and directions: every element in its own frame, the symbols, the compass (X down the drawing
   along `length`, Y right along `width`, Z up)
2. Orientation (where the front of a product or the gripper looks, 0 = the normal pose; the file gives
   orientations, the robot program decides how to turn)
3. The gripper (orientation 0, at the pick in the conveyor's frame, at the place in the pallet's frame)
4. Open side and alignment (900 = centred)
5. Placing direction (900 = straight down from above)
6. The stack and slip sheets (slip sheet types with a thickness, the pallet's height, layer heights)
7. The OpenPallet 1.2 file (its structure as a tree, the `x-palletdesigner` block)

The JSON Schema of the file and example files live in [openpallet/schema](https://github.com/openpallet/schema).

## Licence

Text and cards: [Creative Commons Attribution 4.0 International](LICENSE) (CC BY 4.0).
Copyright 2026 Marcin Daniun Automatyka (Mardan Automation).

Every copy, translation or quotation names the author with this line:

> OpenPallet, created and maintained by the makers of PalletDesigner
