---
title: "Lock, Stock and Two Tiled Fences: Haiku on a MacBook4,1"
date: 2026-10-15T22:00:00+02:00
description: "What sliced the intel_extreme display on a MacBook4,1 (GMA X3100), and the patches that fix it."
draft: false
tags: [haiku, intel_extreme, gpu, efi, macbook]
categories: Haiku
---

On a MacBook4,1, Haiku showed a sliced desktop: skewed horizontal bands across the whole screen from boot onwards. It also showed no boot splash. Here is what was wrong, what changed, and why.

## TL;DR

| | |
|---|---|
| **Hardware** | MacBook4,1, GMA X3100 (i965GM, `0x2a02`, Gen4), LVDS on pipe B, Apple EFI 1.1 (UGA only, no GOP) |
| **Symptom** | Sliced/skewed screen; no boot splash |
| **Root cause** | Apple EFI leaves two **X-tiled fences** over aperture 0–32 MB. Haiku cleared fences on Gen6+ only, so every linear CPU write to our frame buffer got tiled on the way in. |
| **Fix** | Clear the 16 fences on Gen4–5 before the first mode set |
| **Bonus fixes** | Plane/pipe hygiene on Gen ≤4, the GTT location and entry count in `intel_gart`, a missing `&`, and a boot splash on GOP-less Intel EFI |

Tested on one MacBook4,1 only. See [Testing wanted](#testing-wanted).

## Root cause: two fences

Haiku never uses tiling. Apple's firmware does, and it didn't tidy up before leaving. The accelerant log, with the new trace:

```
disable_fences: clearing fence 0: start 0x0 end 0xffffff pitch 8192 X-tiled
disable_fences: clearing fence 1: start 0x1000000 end 0x1ffffff pitch 8192 X-tiled
```

A fence makes the GMCH tile CPU writes that go through the aperture. Our frame buffer sits at aperture offset `0x10000`, right inside fence 0. app_server writes linear rows with a 5120-byte stride, the fence reshuffles them into 8192-pitch X tiles, and the plane scans that out linearly. The result is bands.

Linux has a matching quirk for these machines: `sysfb_efi` `M_MB`, stride 2048 * 4.

Two fences, one pitch, and nobody tidied up. Lesson learned: always check the fences.

## 1. Clear the fences on Gen4–5 (the actual fix)

`headers/private/graphics/intel_extreme/intel_extreme.h`

```diff
 #define INTEL_GRAPHICS_MEMORY_FENCE_GEN6(i) (0x100000 + (i * 8))
+// i965 to IronLake: 16 fences, each 64 bit (low: start, pitch, tiling, valid;
+// high: end)
+#define INTEL_GRAPHICS_MEMORY_FENCE_965(i)	(0x3000 + (i * 8))
+#define INTEL_GRAPHICS_MEMORY_FENCE_VALID	(1UL << 0)
```

`src/add-ons/accelerants/intel_extreme/accelerant.cpp`, `disable_fences()`. It runs at the end of `init_accelerant()`, before the first mode set:

```diff
 		for (int i = 0; i < numFences; i++)
 			write32(INTEL_GRAPHICS_MEMORY_FENCE_GEN6(i), 0);
+	} else if (gInfo->shared_info->device_type.Generation() >= 4) {
+		// The firmware may leave fences around its own tiled frame buffer
+		// (Apple EFI on i965GM does). CPU writes through the aperture to a
+		// fenced range get tiled, which garbles our linear frame buffer.
+		for (int i = 0; i < 16; i++) {
+			uint32 fence = INTEL_GRAPHICS_MEMORY_FENCE_965(i);
+			uint32 low = read32(fence);
+			if ((low & INTEL_GRAPHICS_MEMORY_FENCE_VALID) != 0) {
+				TRACE("%s: clearing fence %d: start 0x%" B_PRIx32 " end 0x%"
+					B_PRIx32 " pitch %" B_PRIu32 " %s-tiled\n", __func__, i,
+					low & ~(uint32)0xfff, read32(fence + 4) | 0xfff,
+					(((low >> 2) & 0x3ff) + 1) * 128,
+					(low & (1UL << 1)) != 0 ? "Y" : "X");
+			}
+			write32(fence, 0);
+			write32(fence + 4, 0);
+		}
+		read32(INTEL_GRAPHICS_MEMORY_FENCE_965(15));
 	}
```

The Gen6+ branch comes first, so this only covers Gen4 and Gen5 (Ironlake uses the same 0x3000 layout).

## 2. Plane/pipe hygiene on Gen ≤4

These changes came first, during the hunt. On their own they didn't fix the slicing, but each one closes a real hole left by firmware state, so they stay.

On Gen2–4 any primary plane can feed any pipe (DSPCNTR bit 24). Haiku assumed plane B feeds pipe B and never wrote the bit. A tiled plane also scans from `DSPxTILEOFF`, not `LINOFF`.

`headers/private/graphics/intel_extreme/intel_extreme.h`

```diff
 #define DISPLAY_CONTROL_TILE_MODE_MASK	(7UL << 10)
+// Gen2-4 only: primary plane to pipe assignment (overlaps Skylake color bits)
+#define DISPLAY_CONTROL_PIPE_SELECT_MASK	(3UL << 24)
+#define DISPLAY_CONTROL_PIPE_B			(1UL << 24)
+
+// i965 and G4x: used instead of INTEL_DISPLAY_A_BASE when the plane is tiled
+#define INTEL_DISPLAY_A_TILE_OFFSET		INTEL_DISPLAY_A_OFFSET_HAS
+#define INTEL_DISPLAY_B_TILE_OFFSET		INTEL_DISPLAY_B_OFFSET_HAS
```

`src/add-ons/accelerants/intel_extreme/Pipes.cpp`, `Pipe::Enable()`: tie our plane to our pipe and zero the tile offset.

```diff
-		write32(planeReg, (read32(planeReg) | DISPLAY_CONTROL_ENABLED)
-			& ~DISPLAY_CONTROL_TILE_MODE_MASK);
+		uint32 planeControl = (read32(planeReg) | DISPLAY_CONTROL_ENABLED)
+			& ~DISPLAY_CONTROL_TILE_MODE_MASK;
+		if (gInfo->shared_info->device_type.Generation() <= 4
+			&& fPipeIndex <= INTEL_PIPE_B) {
+			// Planes can feed either pipe here, and the firmware may have
+			// crossed them (MacBook EFI); tie our plane to our pipe.
+			planeControl &= ~DISPLAY_CONTROL_PIPE_SELECT_MASK;
+			if (fPipeIndex == INTEL_PIPE_B)
+				planeControl |= DISPLAY_CONTROL_PIPE_B;
+			if (gInfo->shared_info->device_type.Generation() == 4)
+				write32(INTEL_DISPLAY_A_TILE_OFFSET + fPlaneOffset, 0);
+		}
+		write32(planeReg, planeControl);
```

`src/add-ons/accelerants/intel_extreme/mode.cpp`, `intel_set_display_mode()`:
- Turn off every firmware plane before touching the ports.
- Program the stride before the planes come back on, so the first surface write latches the new stride instead of the firmware's.

Plane registers are double-buffered: a write to SURF latches them on i965, a write to BASE on older chips.

```diff
 	write32(INTEL_VGA_DISPLAY_CONTROL, VGA_DISPLAY_DISABLED);
 	read32(INTEL_VGA_DISPLAY_CONTROL);
 
+	if (sharedInfo.device_type.Generation() <= 4) {
+		// The firmware may have left a plane enabled on a pipe we are about
+		// to drive (possibly tiled, and with its own stride). Turn all
+		// primary planes off; Pipe::Enable() turns on the ones we use.
+		write32(INTEL_DISPLAY_A_CONTROL,
+			read32(INTEL_DISPLAY_A_CONTROL) & ~DISPLAY_CONTROL_ENABLED);
+		write32(INTEL_DISPLAY_B_CONTROL,
+			read32(INTEL_DISPLAY_B_CONTROL) & ~DISPLAY_CONTROL_ENABLED);
+		// plane updates are only latched by a write to the surface register
+		// (i965+), or to the base register on older chips
+		uint32 latchA = sharedInfo.device_type.Generation() == 4
+			? INTEL_DISPLAY_A_SURFACE : INTEL_DISPLAY_A_BASE;
+		uint32 latchB = sharedInfo.device_type.Generation() == 4
+			? INTEL_DISPLAY_B_SURFACE : INTEL_DISPLAY_B_BASE;
+		write32(latchA, read32(latchA));
+		write32(latchB, read32(latchB));
+		wait_for_vblank();
+	}
+
```

```diff
 	program_pipe_color_modes(colorMode);
 
+	if (sharedInfo.device_type.Generation() <= 4) {
+		// Program the stride before the planes come back on, so that the
+		// surface write in set_display_power_mode() already latches it,
+		// instead of scanning out with the old (firmware) stride.
+		write32(INTEL_DISPLAY_A_BYTES_PER_ROW, bytesPerRow);
+		write32(INTEL_DISPLAY_B_BYTES_PER_ROW, bytesPerRow);
+		sharedInfo.current_mode = target;
+		sharedInfo.bytes_per_row = bytesPerRow;
+		sharedInfo.bits_per_pixel = bitsPerPixel;
+	}
+
 	// TODO: This may not be neccesary (see DPMS OFF at top)
 	set_display_power_mode(sharedInfo.dpms_mode);
```

## 3. intel_gart: the GTT was in someone else's house

Two bugs, found while reading the boot logs.

**Wrong GTT location.** Everything except 8xx and 91x assumed MMIO + 2 MB. That's correct for G4x and later only:
- 945, G33 and Pineview have a separate GTT BAR (like 915).
- On i965 the GTT directly follows the 512 KB of registers in its 1 MB MMIO BAR.

On this MacBook, MMIO + 2 MB is `0xd0200000`: the **FireWire controller** behind the PCI bridge. So the GTT was never programmed, and all GTT writes went to the FireWire controller. It only appeared to work because the firmware's GTT already mapped stolen memory linearly.

**Wrong entry count.** `gttSize / 4096` treats each entry as a page. Before Gen8 an entry is 4 bytes, so a 512 KB GTT was reported as 128 entries instead of 131072, and the scratch-page loop skipped nearly the whole table. Gen8+ uses 64-bit entries, which `set_gtt_entry()` doesn't handle yet, so it keeps the old count on purpose.

`src/add-ons/kernel/busses/agp_gart/intel_gart.cpp`, `intel_map()`

```diff
-	} else if (info.type->InGroup(INTEL_GROUP_91x)) {
-		info.gtt_physical_base = get_pci_config(info.display, i915_GTT_BASE, 4);
+	} else if (info.type->Generation() == 3) {
+		// 915, 945, G33 and Pineview have a separate GTT BAR
+		info.gtt_physical_base = get_pci_config(info.display, i915_GTT_BASE, 4)
+			& ~(phys_addr_t)0xf;
+	} else if (info.type->Generation() == 4
+		&& !info.type->InGroup(INTEL_GROUP_G4x)) {
+		// i965: the GTT follows the 512 KB of registers in the MMIO BAR
+		info.gtt_physical_base = addr + (512UL << 10);
 	} else {
-		// 945+?
+		// G4x and later
 		info.gtt_physical_base = addr + (2UL << 20);
 	}
 
 	size_t gttSize = determine_gtt_size(info);
 	size_t stolenSize = determine_gtt_stolen(info);
 
-	info.gtt_entries = gttSize / 4096;
+	if (info.type->Generation() == 4 && !info.type->InGroup(INTEL_GROUP_G4x)
+		&& info.gtt_physical_base + gttSize > addr + barSize) {
+		// we would write the GTT entries into whatever device follows
+		ERROR("GTT (%" B_PRIuSIZE " bytes at 0x%" B_PRIxPHYSADDR ") is outside"
+			" of the MMIO BAR!\n", gttSize, info.gtt_physical_base);
+		return B_ERROR;
+	}
+
+	if (info.type->Generation() < 8) {
+		// one 32 bit entry per page
+		info.gtt_entries = gttSize / sizeof(uint32);
+	} else {
+		// TODO: Gen8+ uses 64 bit entries, which set_gtt_entry() does not
+		// handle yet; keep the old (too small) count so we don't scribble
+		// over live entries with the scratch page loop.
+		info.gtt_entries = gttSize / 4096;
+	}
```

Log before and after:

```
before: intel_gart: GTT base = 0xd0200000   (FireWire, bus 3)
after:  intel_gart: GTT base = 0xd0080000, entries 131072
```

## 4. Status page: one missing `&`

`src/add-ons/kernel/drivers/graphics/intel_extreme/intel_extreme.cpp`, `intel_extreme_init()`. The old code passed the pointer's value instead of its address, so the allocation failed with `B_BAD_VALUE`. Nothing uses the page yet, but now it actually exists.

```diff
 	if (intel_allocate_memory(info, B_PAGE_SIZE, 0, B_APERTURE_NEED_PHYSICAL,
-			(addr_t*)info.shared_info->status_page,
+			(addr_t*)&info.shared_info->status_page,
 			&info.shared_info->physical_status_page) == B_OK) {
```

## 5. EFI loader: a boot splash without GOP

Apple EFI 1.1 offers UGA only, so the loader found no GOP, set no frame buffer, and showed no splash. The display is running, though, so the loader now reads what the firmware left in the plane registers.

Two traps along the way:
- **The firmware frame buffer is tiled.** It's at aperture `0x1000000` with stride 8192 and covered by fence 1. Linear writes are only correct if a matching X fence covers the surface, so the loader checks for one.
- **The firmware frame buffer isn't in stolen memory.** The GTT maps it to *boot services* RAM, which the kernel reuses after `ExitBootServices()`. Every splash write would scribble on kernel memory. Like Brick Top's pigs, the page allocator eats whatever it's given. The loader checks the GTT entries. If the buffer isn't 1:1 stolen, it moves the plane to aperture offset 0 (stolen, fenced identically) when the splash appears, or else gives up.

`src/system/boot/platform/efi/video.cpp`: about 250 new lines, restricted to i965/G4x device IDs. The parts that matter:

```cpp
/*!	Returns whether the GTT maps the aperture range 1:1 to stolen memory.
	Only that is safe to draw to: the firmware may have put its frame buffer
	into boot services memory, which the kernel reuses.
*/
static bool
intel_maps_stolen_memory(efi_pci_root_bridge_io_protocol* pci, uint64 gtt,
	uint64 stolenBase, uint32 offset, uint32 size)
{
	for (uint32 page = offset; page < offset + size; page += B_PAGE_SIZE) {
		uint32 entry = intel_read32(pci, gtt + page / B_PAGE_SIZE * 4);
		uint64 address = (entry & ~0xfffULL) | ((uint64)(entry & 0xf0) << 28);
		if ((entry & GTT_ENTRY_VALID) == 0 || address != stolenBase + page)
			return false;
	}
	return true;
}
```

In `find_intel_firmware_frame_buffer()`, after validating plane, pipe source size, stride and fence:

```cpp
		bool relocate = !intel_maps_stolen_memory(pci, gtt, stolenBase,
			surface, size);
		if (relocate) {
			// Apple EFI scans out of boot services memory; use the start of
			// the aperture instead
			surface = 0;
			if (!intel_maps_stolen_memory(pci, gtt, stolenBase, surface, size)
				|| (tiled && !intel_is_fenced(pci, mmio, surface, size,
					stride))) {
				dprintf("Intel firmware frame buffer is not in stolen memory,"
					" ignoring it\n");
				continue;
			}
		}
		...
		// only enabled in platform_switch_to_logo()
		gKernelArgs.frame_buffer.enabled = false;
```

The relocation is deferred to `platform_switch_to_logo()`, so the firmware text console, and with it the boot menu, keeps working until then:

```cpp
static void
switch_to_intel_frame_buffer()
{
	if (sIntelFrameBuffer.relocate) {
		...
		// the offsets are latched by the surface write
		intel_write32(pci, mmio + INTEL_PLANE_LINEAR_OFFSET(plane), 0);
		intel_write32(pci, mmio + INTEL_PLANE_TILE_OFFSET(plane), 0);
		intel_write32(pci, mmio + INTEL_PLANE_SURFACE(plane),
			sIntelFrameBuffer.surface);
		sIntelFrameBuffer.relocate = false;
	}

	gKernelArgs.frame_buffer.enabled = true;
}
```

```diff
 		dprintf("GOP protocol not found\n");
 		gKernelArgs.frame_buffer.enabled = false;
 		sGraphicsOutput = NULL;
-		return B_ERROR;
+		sFirmwareFrameBuffer = find_intel_firmware_frame_buffer();
+		return sFirmwareFrameBuffer ? B_OK : B_ERROR;
```

```diff
 platform_switch_to_logo(void)
 {
+	if (sFirmwareFrameBuffer) {
+		// no mode setting possible, keep what the firmware set up
+		switch_to_intel_frame_buffer();
+		video_display_splash(gKernelArgs.frame_buffer.physical_buffer.start);
+		return;
+	}
```

Result:

```
using Intel firmware frame buffer: 1280x800, stride 8192, at 0xc0000000 (fenced) (moved to stolen memory)
```

`src/system/boot/platform/generic/video_splash.cpp`: the splash clears `bytes_per_row * height` instead of `width * height * 4`. The old size cleared only 62% of an 8192-stride screen.

```diff
-	addr_t size = min_c(gKernelArgs.frame_buffer.width
-			* gKernelArgs.frame_buffer.height * 4u,
+	addr_t size = min_c((addr_t)gKernelArgs.frame_buffer.bytes_per_row
+			* gKernelArgs.frame_buffer.height,
 		gKernelArgs.frame_buffer.physical_buffer.size);
```

Side note: if your splash shows icons but a nearly black logo, that isn't a bug. Default builds use `images-sans-tm.h`. Build with `--distro-compatibility official` to get the real logo.

## Who is affected

| Change | Reaches | Risk |
|---|---|---|
| Fence clearing | Gen4–5 | Low: Haiku doesn't use tiling |
| Plane/pipe select, tile offset, plane off, early stride | Gen ≤4 (tile offset: Gen4) | Low: makes Haiku's existing plane B → pipe B assumption explicit |
| GTT location | Gen3 (945, G33, Pineview), i965 | **Medium:** changes where the GTT is written; matches Linux `intel-gtt.c` |
| GTT entry count | Gen < 8 | Low: unused entries now point at the scratch page, as on Linux |
| Status page `&` | all | Very low |
| EFI frame buffer | No GOP + Intel i965/G4x at 0:2.0 only | Very low: every other machine takes the old path |
| Splash clear size | all | Very low: exact instead of estimated |

## Testing wanted

Only one MacBook4,1 has run this. Boot tests are especially welcome on:
- **945 / G33 / Pineview** (lots of netbooks): the GTT now comes from BAR 0x1c. Check that `intel_gart: GTT base` matches that BAR.
- **Ironlake**: fence clearing.
- **G4x** and any other GOP-less Intel EFI machine.

If it breaks, please send the syslog; it's worth more than a thousand words. *"It's been emotional."*

## Sources of Documentation

1. Haiku boot logs. Everything specific to Macbook4,1 came from them: the fence addresses, stride 8192, stolen memory at 0xbf000000, the FireWire controller at 0xd0200000, and the boot services memory in the EFI map.
2. Haiku's source code.
3. Intel's official manuals (PRMs), published for open-source driver developers: Index: https://www.intel.com/content/www/us/en/docs/graphics-for-linux/developer-reference/1-0/overview.html
  - The volume for my chip is "Intel 965 Express Chipset Family and Intel G35 Express Chipset Graphics Controller PRM" (2008):
    - Vol. 1 covers the GTT, fences and memory interface.
    - Vol. 3 covers the display registers: plane control, stride, surface, tile offset.
  - "Mobile Intel 965 Express Chipset Family Datasheet" covers the PCI configuration registers, such as stolen memory size (GGC, 0x52) and stolen
    memory base (BSM, 0x5C).
4. Linux kernel source.
