# Terminal Compatibility

slk works in any modern terminal, but the experience scales with what your
terminal supports. This table summarizes the important capabilities; pick
something from the top of the list for the richest experience.

| Terminal              | Inline images        | True-pixel avatars | OSC 8 links | OSC 52 clipboard | Notes                                                       |
|-----------------------|----------------------|--------------------|-------------|------------------|-------------------------------------------------------------|
| **kitty**             | kitty graphics       | yes                | yes         | yes              | Best overall experience. Older versions may need `clipboard_control write-clipboard`. |
| **ghostty**           | kitty graphics       | yes                | yes         | yes              | Recommended.                                                |
| **WezTerm** (recent)  | kitty graphics       | yes                | yes         | yes              |                                                             |
| **foot** (Wayland)    | sixel                | half-block         | yes         | yes              | Best Wayland-native option.                                 |
| **iTerm2 ≥ 3.5**      | half-block           | half-block         | yes         | yes              | Implements kitty graphics but not unicode placeholders, so slk falls back to half-block. |
| **Alacritty**         | half-block           | half-block         | yes (≥0.11) | yes              | Fast and reliable, but no inline images.                    |
| **gnome-terminal** (recent) | half-block     | half-block         | yes         | yes              |                                                             |
| **mlterm**            | sixel                | half-block         | partial     | partial          |                                                             |
| **screen**            | half-block           | half-block         | no          | no               | No working OSC 52 path; consider switching to tmux.         |

## Inside tmux

By default slk forces half-block for inline images when it detects tmux.

To get sharp kitty graphics inside tmux (tmux 3.4+ recommended):

1. Set slk's image protocol to kitty explicitly in `~/.config/slk/config.toml`:
   ```toml
   [appearance]
   image_protocol = "kitty"
   ```
2. slk will automatically run `tmux set -p allow-passthrough on` at startup
   for the current pane. You can also add it globally in `~/.tmux.conf`:
   ```bash
   set -g allow-passthrough on
   ```

When running inside tmux with `image_protocol = "kitty"`, slk wraps kitty
APC sequences in tmux DCS passthrough so they reach the outer terminal.

OSC 52 clipboard (copy/paste) still requires `set -g set-clipboard on` in your
tmux config — that setting is unrelated to image rendering.

## Overriding the image protocol

You can override slk's image-protocol pick via the `[appearance] image_protocol`
config key (`auto` / `kitty` / `sixel` / `halfblock` / `off`). See
[[Configuration]] for details.

## Related

- [[Clipboard and OSC 52|Clipboard-and-OSC-52]] — getting copy/paste to land
- [[Tradeoffs and Non-Goals|Tradeoffs-and-Non-Goals]] — image rendering caveats (animated GIFs, unfurls, threads pane sixel)
