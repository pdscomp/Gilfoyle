# Parity reviews when the target merges multiple upstream drivers

Some U-Boot / bootloader ports fold what Linux keeps as two separate drivers into one file. The classic case is a MIPI-DSI host whose D-PHY is a separate `phy-` driver in Linux but is inlined in the U-Boot driver.

## What to do

1. Identify every upstream driver that contributed logic to the target.
   - Read the target's header comment; it usually names the upstream files.
   - Search the target for function names that match upstream (`*_dphy_*`, `*_dsi_*`, etc.).

2. Open each upstream file in parallel and compare the relevant sections:
   - Register definitions and bitfield masks.
   - Power-on / init sequences (watch ordering and register defaults).
   - Algorithmic functions: timing calculations, packet builders, delay formulas.
   - Error handling and state cleanup.

3. For each finding, quote `file:line` from **both** the target and the upstream reference(s). A divergence without an upstream line number is just an opinion.

4. Distinguish:
   - **Intentional SoC-specific change:** document it in the finding (e.g., different TCON divider, different reset line).
   - **Silent divergence:** flag it as a bug, because the port claims to match the upstream hardware behavior.

## Common traps

- The target may skip a register write because the upstream driver relies on a reset default. Flag it as a divergence even if it likely works.
- Timing calculations are easy to miscopy. Verify each term against the upstream formula algebraically; do not trust that "it looks the same."
- If the target uses the same register names but different reset/assert sequencing, that is a behavioral divergence, not a cosmetic difference.

## Example: U-Boot DSI + D-PHY vs Linux split drivers

In this session, the target was `drivers/video/sunxi/sunxi_mipi_dsi.c` in U-Boot, which merged logic from:
- `drivers/gpu/drm/sun4i/sun6i_mipi_dsi.c` (DSI host)
- `drivers/phy/allwinner/phy-sun6i-mipi-dphy.c` (D-PHY)

Both files were read and compared section-by-section. The D-PHY PLL and register sequences matched; the video timing formulas diverged.
