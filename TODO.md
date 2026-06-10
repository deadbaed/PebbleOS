# Fix #898: Action layer drawn on wrong side in Left-Handed mode

## Issue

The UI toolkit draws the action bar on the **right** side of the screen regardless of display
orientation. In Left-Handed mode the physical buttons are on the **left**, so action icons don't
align with their buttons.

- **GitHub**: https://github.com/coredevices/PebbleOS/issues/898
- **Status**: Closed as "wontfix" — maintainer open to a PR
- **Boards affected**: All with `CONFIG_ORIENTATION_MANAGER=y` (Asterix, Obelix, Getafix, …)

---

## How Left-Handed Mode Works

When `display_orientation_is_left()` returns `true` (set via Settings → Display → Orientation):

1. `display_set_rotated(true)` — flips the display 180°
2. `button_set_rotated(true)` — remaps button IDs (UP ↔ DOWN)
3. Physical buttons are now on the **left** side of the watch
4. **Bug**: action bar still renders on the **right** side → misaligned with actual buttons

Key API:
```c
#ifdef CONFIG_ORIENTATION_MANAGER
bool display_orientation_is_left(void);   // src/fw/shell/prefs.h
#endif
```

Reference usage in `src/fw/shell/normal/watchface.c` shows the pattern of checking this flag
and swapping button combos / layouts accordingly.

---

## Files to Modify

| File | What to change |
|------|---------------|
| `src/fw/applib/ui/action_bar_layer.h` | Add `action_bar_is_on_right()` helper |
| `src/fw/applib/ui/action_bar_layer.c` | Fix `add_to_window()` position + round background alignment |
| `src/fw/applib/ui/dialogs/expandable_dialog.c` | Flip content x-offset when action bar is on the left |
| *(others?)* | Search for `ACTION_BAR_WIDTH` used in content positioning |

---

## TODOs

### 1. Add orientation-aware helper in `action_bar_layer.h`

```c
#ifdef CONFIG_ORIENTATION_MANAGER
#include "shell/prefs.h"
#endif

static inline bool action_bar_is_on_right(void) {
#ifdef CONFIG_ORIENTATION_MANAGER
  return !display_orientation_is_left();
#else
  return true;
#endif
}
```

### 2. Fix `action_bar_layer_add_to_window()` in `action_bar_layer.c`

Current code hardcodes right-side placement:

```c
// Before (always right):
rect.origin.x = window_bounds->size.w - width;
```

Change to:

```c
// After (orientation-aware):
rect.origin.x = action_bar_is_on_right()
    ? window_bounds->size.w - width
    : 0;
```

### 3. Fix `prv_draw_background_round()` in `action_bar_layer.c`

The round display background oval uses `GAlignLeft` unconditionally:

```c
grect_align(&action_bar_circle_frame, &action_bar->layer.bounds, GAlignLeft, false);
```

Should align to `GAlignRight` when `action_bar_is_on_right()` is false (left-handed mode).

### 4. Fix `prv_expandable_dialog_load()` in `expandable_dialog.c`

All content positioning (header x, text x, icon x, widths) assumes action bar is on the right.
When the action bar moves to the left:
- Content should start at `action_bar_offset` instead of `0` on the x-axis
- Width calculations stay the same but origin shifts

Key variables to adjust:
- `x` for header layer, text layer, icon layer
- Width reductions using `action_bar_offset`

### 5. Audit other dialogs / UI components

Search the codebase for other places that use `ACTION_BAR_WIDTH` or `action_bar_offset` to
position content. Candidates:
```bash
grep -rn "action_bar_offset\|ACTION_BAR_WIDTH" src/fw/applib/ui --include="*.c"
```

Apply the same left-handed flip logic where needed.

---

## Testing

Build for a board with `CONFIG_ORIENTATION_MANAGER` (e.g. Asterix):

```bash
./pbl configure --board asterix
./pbl build
./pbl test
```

Verify in QEMU:
1. Default orientation → action bar on **right**, content fills remaining space
2. Left-Handed orientation → action bar on **left**, content fills remaining space
3. Round display (Chalk/Gabbro) → background oval renders on correct side
4. Expandable dialogs → text/icons don't overlap the action bar in either orientation
