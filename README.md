# termcopy
Try hard to copy to the clipboard from a terminal.

[osc52.sh](https://chromium.googlesource.com/apps/libapps/+/refs/tags/hterm-1.86/hterm/etc/osc52.sh)
from hterm can set the clipboard in terminals that support OSC 52.  It has
some workarounds for screen and tmux, but it is inconsistent with newline
handling and doesn't work everywhere.

termcopy is a fork of osc52.sh with additional workarounds that I've collected
and the ability to try other ways to set the clipboard.  termcopy first looks
for local programs that can reliably connect to the display server and set the
clipboard (xclip/xsel for X11, wl-copy for Wayland, pbcopy on macOS).  If all
of those fail, it emits OSC 52 sequences.

termcopy tries to preserve your exact input.  If the input includes a final
newline, so will the copied text, and vice versa.  This means that
`echo abc | termcopy` and `printf abc | termcopy` will produce different
results (just like they do when they print to the terminal).

## tmux

tmux creates difficulties because it hides your true terminal and also can't
be detected from a remote session without manual setup.  If termcopy detects
that it is running in tmux, it will emit a private OSC sequence that makes
tmux pass the inner OSC sequence to its containing terminal.  If you only run
termcopy directly inside tmux (locally or remotely), this should be sufficient
and you shouldn't need any extra setup.

If you run ssh inside tmux or you use nested tmux sessions, termcopy won't
always manage to get the OSC 52 sequences through to the outermost terminal.
Luckily, you can set a couple of options in `.tmux.conf` that will let it pass
through OSC 52 sequences almost perfectly:

```
set -g set-clipboard on
set -ga terminal-overrides ',xterm*:XT:Ms=\E]52;%p1%s;%p2%s\007'
set -ga terminal-overrides ',screen*:XT:Ms=\E]52;%p1%s;%p2%s\007'
```

`set-clipboard` tells tmux to accept and regenerate the OSC 52 clipboard
sequences.  The first `terminal-overrides` makes tmux think that your terminal
has `Ms` in its terminfo database (assuming `TERM` starts with `xterm`), which
makes tmux emit the same OSC 52 sequence to set the clipboard.  The second
`terminal-overrides` covers nested tmux sessions, since tmux typically sets
its inner `TERM` to `screen` or `screen-256color`.  The two terminal overrides
may not be needed, or you may need an additional entry for your own `TERM`
value, depending on the exact contents of your terminfo database.  I don't
think it hurts to include these even if your terminfo already has them.

## kitty

[kitty](https://sw.kovidgoyal.net/kitty/) introduces another wrinkle.  By
default, it
[appends](https://sw.kovidgoyal.net/kitty/protocol-extensions.html#pasting-to-clipboard)
to the clipboard instead of replacing the contents.  termcopy works around
this by sending an invalid OSC 52 command to clear the clipboard before
setting it.  If you use kitty without tmux, you don't need to do anything.

If you use tmux with kitty, tmux will not pass through the invalid
clipboard-clearing sequence, so each termcopy invocation will append to the
clipboard.  Again, there is a setting you can change in
`~/.config/kitty/kitty.conf`:

```
clipboard_control write-clipboard write-primary no-append
```

With the tmux and kitty settings described above, termcopy should work with
any combination of local and remote sessions with tmux and ssh nested any way
you like.
