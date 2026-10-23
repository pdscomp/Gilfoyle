# Embedded DSI panel bring-up review notes

Concrete checklist distilled from a U-Boot MIPI-DSI host + panel review (`cc2-dsi` branch, Allwinner T113 / ST77922).

## Scope traps

- Feature branch names (`cc2-dsi`) are not evidence of what is in the diff.
- Always diff against the merge base: `git diff $(git merge-base HEAD <base>)..HEAD --stat`.
- If the last commit reverted defconfig/DTS to generate a clean patch, note that the board-specific files are not in the review and build cannot be verified independently.

## No test suite is normal

- U-Boot BSP bring-ups rarely have `cargo test` / `pytest` / `npm test`.
- State explicitly that no test suite exists.
- Rely on: diff review, upstream driver comparison, DT bindings, and any build smoke tests you can run.

## Panel-side correctness checks

1. **Attach the DSI device before use.** Every working panel driver calls `mipi_dsi_attach(dsi)` before sending DCS commands. The host driver's `attach` callback sets `host->device` and validates wiring.
2. **Reset GPIO polarity.** Use `GPIOD_ACTIVE_LOW` in `gpio_request_by_name()` so the framework translates logical assert/deassert values. Do not hard-code active-low in the driver.
3. **Enable sequencing.** The DSI host must be configured (clocks out, reset released, D-PHY up) before the panel sends DCS initialization. If the host has a two-phase enable (command mode, then HS video), confirm the first phase runs before DCS and the second after.
4. **Backlight handling.** `backlight_enable()` may return `-ENOSYS` or `-ENOENT` when no backlight is configured; treat those as success, not failure.

## Host-side correctness checks

1. **Clock-before-reset ordering.** Enable all clocks before releasing reset lines. Releasing reset into a powered-down clock domain can cause the block to latch garbage.
- D-PHY variant selection. If the driver uses a compatible -> variant table, ensure the board's exact compatible is listed. A fallback to a different variant is a silent failure on future DTS updates.
3. **Two-phase enable state machine.** If `enable()` does command setup on first call and HS video start on second call, document the expected call sequence and verify there is a caller for the first phase. Do not assume the panel driver's `enable_backlight()` will call the host `enable()` for you — trace it.
4. **Register reset symmetry.** If `video_setup()` asserts/deasserts the host reset, it should also handle the D-PHY reset, or assert it is already handled and document why.
5. **Blind delays vs polls.** `udelay()` between D-PHY PLL enable and HS start is acceptable for boot, but polling the PLL lock bit is more robust.

## TCON-LCD DSI trigger-mode checklist

When a U-Boot TCON-LCD driver is ported to drive a MIPI-DSI panel through the CPU/trigger interface, compare the register sequence against Linux `sun4i_tcon0_mode_set_cpu()` in `drivers/gpu/drm/sun4i/sun4i_tcon.c`. U-Boot ports commonly skip prerequisite writes:

- `SUN4I_TCON0_CTL_REG` interface field must be set to `SUN4I_TCON0_CTL_IF_8080` before the CPU-IF register is configured.
- `SUN4I_TCON_ECC_FIFO_REG` must have `SUN4I_TCON_ECC_FIFO_EN` enabled.
- `SUN4I_TCON0_CPU_IF_REG` must include:
  - `SUN4I_TCON0_CPU_IF_MODE_DSI`
  - `SUN4I_TCON0_CPU_IF_TRI_FIFO_FLUSH`
  - `SUN4I_TCON0_CPU_IF_TRI_FIFO_EN`
  - `SUN4I_TCON0_CPU_IF_TRI_EN`
- `SUN4I_TCON0_CPU_TRI0/1/2_REG` block-space, block-number, and start-delay values must be computed from the mode timing.
- `SUN4I_TCON_SAFE_PERIOD_REG` must be configured.
- When TCON-TOP is present, open the DSI gate in `TCON_TOP_GATE_SRC_REG` (`TCON_TOP_TCON_DSI_GATE`) before expecting the DSI path to move pixels.
- Enable the DSI host in command mode before the panel sends DCS initialization; only enable the panel backlight after the host is ready.

## U-Boot TCON-LCD DSI parity pitfalls (vs Linux)

- **Ordering of panel init vs. TCON setup.** The panel's DCS init sequence must run *after* the DSI host is configured and the TCON-LCD0 CPU-IF path is programmed. Do not call `panel_enable_backlight()` (which often triggers DCS init + HS video start) before `lcdc_tcon0_mode_set()` and the CPU-IF/trigger register writes. Linux calls `sun4i_tcon0_mode_set_cpu()` before the DSI encoder is enabled, and the encoder enable is what lets DCS traffic flow.
- **The CPU-IF register alone is not enough.** A working DSI trigger interface also requires `SUN4I_TCON0_CPU_TRI0/1/2_REG` and `SUN4I_TCON_SAFE_PERIOD_REG`. The Linux reference writes them with mode-derived block-space, block-number, start-delay, and a safe-period of mode 3 / num 3000, then clears the output tristate. Skipping these registers produces a DSI path that accepts commands but never generates valid HS video frames.
- **TCON-TOP DSI gate is a bit shift, not a mask.** `TCON_TOP_TCON_DSI_GATE` is defined as `16` (the bit position). The register write must use `BIT(16)` or `(1U << 16)`, not the raw shift value.
- **TCON-TOP reset should not be optional.** Unlike some generic bus drivers where reset is optional, the TCON-TOP block on D1/R528 has a bus reset. Returning success when `reset_get_by_name()` returns `-ENOENT` or `-ENOSYS` will boot on some boards and fail on others; fail hard if the reset is missing.
- **TCON-TOP init ordering vs. Linux.** Linux `sun8i_tcon_top.c` deasserts the bus reset (`reset_control_deassert`) before enabling the bus clock (`clk_prepare_enable`). If parity is the goal, match that order. Also clear `TCON_TOP_PORT_SEL_REG` and `TCON_TOP_GATE_SRC_REG` before writing the new route/gate, as Linux does, so stale firmware values do not leak into other TCON/HDMI paths.
- **Avoid `uclass_get_device_by_driver()` to find the DSI host or TCON-TOP.** Tying the TCON driver to a specific driver implementation by name is brittle. Use the normal U-Boot DM parent/child or phandle relationships so the correct host and top block are discovered from the device tree.
- **Do not misstate the Linux DSI TCON divider.** Linux defines `SUN6I_DSI_TCON_DIV` as `4` in `drivers/gpu/drm/sun4i/sun6i_mipi_dsi.h`, and `sun4i_tcon0_mode_set_cpu()` divides the byte clock by that value. A U-Boot port that hardcodes `8` (or `pixelclock * bpp / 8 / lanes`) must explicitly document that as an intentional SoC-specific override, not as "matches Linux." The same actual TCON divider must be used in `block_space` and `start_delay` calculations, not a blindly hardcoded `/8`.
- **`start_delay` must divide by the real TCON divider, not a magic 8.** Linux computes `block_space = mode->htotal * bpp / (tcon_div * lanes)` and reads `tcon_div` back from `SUN4I_TCON0_DCLK_REG`. If the U-Boot PLL code chooses a divider other than 1, a hardcoded `/8` in `start_delay` (or the implicit divider in `block_space`) silently diverges from upstream.

## Reference comparison

- For merged port drivers (e.g., one U-Boot driver combining a Linux DSI host + D-PHY driver), pull both Linux source files side-by-side.
- Quote `file:line` from the target and the reference for each divergence.
- Distinguish intentional improvement from silent divergence.
- **Linux source fetch fallback.** If `web_extract` from Elixir Bootlin fails (common for source pages), use a sparse Linux git clone instead:

  ```bash
  git clone --depth 1 --filter=blob:none --sparse \
    https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git /tmp/linux-ref
  cd /tmp/linux-ref
  git sparse-checkout init --cone
  git sparse-checkout set drivers/gpu/drm/sun4i
  ```

  Then `read_file` or `search_files` against `/tmp/linux-ref` for authoritative `file:line` citations.

## Output discipline

- One-line scope statement first.
- Severity-ranked findings: 🔴 Critical → 🟠 Major → 🟡 Minor.
- Each finding: `file:line`, problem, impact, concrete fix.
- Verdict: ship / fix-then-ship / back-to-drawing-board.

## Worked example: U-Boot T113 DSI TCON-LCD0 parity review

Real review: u-boot-cc2 commit `ad7f2511f62`, branch `cc2-dsi-patches`, vs Linux `sun4i_tcon0_mode_set_cpu()` and `sun8i_tcon_top.c`.

Divergences flagged:

1. **Panel/TCON enable ordering** — `panel_enable_backlight()` ran before `lcdc_enable()`, while the commit message claimed the opposite. The DSI host must be in command mode for DCS init, then HS video mode, and the TCON must be enabled only after HS video is active. If the panel driver starts HS video immediately, a disabled TCON misses the first frame.
2. **CPU-TRI block_space omitted hsync_len** — Linux uses `htotal` (hdisplay + hfront + hsync + hback). The U-Boot code used only `hactive + hback + hfront`, underestimating the line and producing a wrong block_space.
3. **start_delay formula** — Linux uses `mode->crtc_htotal` directly; U-Boot expanded it manually. Verify the exact `-1` semantics of the `BLOCK_SPACE`/`BLOCK_SIZE` macros when porting.
4. **TCON byte clock missing DSI divider** — Linux divides `tcon->dclk` by `SUN6I_DSI_TCON_DIV`. U-Boot used `pixelclock * bpp / 8 / lanes` without the divider. On T113 the host side may compensate, but this is a silent parity divergence.
5. **TCON-TOP reset treated as optional** — Linux fails hard if the bus reset is missing. U-Boot ignored `-ENOENT`/`-ENOSYS`.
6. **TCON-TOP clock/reset ordering** — Linux deasserts reset before enabling the clock; U-Boot enabled the clock first.
7. **TCON-TOP stale bits** — Linux clears `PORT_SEL` and `GATE_SRC` before writing; U-Boot only ORed in the DSI gate.
