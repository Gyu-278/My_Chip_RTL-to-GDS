# Attribution

This repository contains my **personal practice work** while following the
open-source RTL-to-GDS course *"My Chip on My Desk"* (ETRI MyChip MPW service,
instructor: 국일호 / GoodKook, ETRI).

## Original source

The example code here is **copied and modified** from the official design kit:

- **Project:** ETRI 0.5µm CMOS MPW Standard-Cell Design Kit
- **Author:** GoodKook (goodkook@gmail.com)
- **Repository:** https://github.com/GoodKook/ETRI-0.5um-CMOS-MPW-Std-Cell-DK
- **License:** CC BY-NC (as stated in the original source file headers)

Original license/authorship headers inside the source files are kept unchanged.

## My modifications

- `*/pong_SbS/pong_SbS.v` (01_Table, 03_TableDraw): fixed mixed blocking/
  non-blocking assignments (`x_pos += 1;` → `x_pos <= x_pos + 1;`) for
  Verilator 5.x compatibility, and moved the wrap condition to `x_pos == 127`.
- Builds were done in a personal conda environment instead of the kit's
  assumed `/opt/systemc` (invoked with `make ... SYSTEMC=$CONDA_PREFIX`).

## Third-party material

Datasheets under `*/_Docs_/` (e.g. KS0108B, JHD12864E) belong to their
respective manufacturers and are included here only as reference, as they were
in the original kit.

## Note

This repo is for **non-commercial, educational** use only, consistent with the
original CC BY-NC license.
