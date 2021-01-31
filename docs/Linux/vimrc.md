# VIM

## Redefining tabs to 4 spaces (keystroke + conversion)

> /etc/vim/vimrc.local
```conf vimrc.local
set tabstop=4
set softtabstop=0
set expandtab
set shiftwidth=4
set smarttab
set mouse=
set ttymouse=
```


---

```conf --NONAME--
tabstop
    The width of a hard tabstop measured in "spaces" -- effectively the (maximum) width of an actual tab character.

shiftwidth
    The size of an "indent". It's also measured in spaces, so if your code base indents with tab characters
    then you want shiftwidth to equal the number of tab characters times tabstop.
    This is also used by things like the =, > and < commands.

softtabstop
    Setting this to a non-zero value other than tabstop will make the tab key (in insert mode)
    insert a combination of spaces (and possibly tabs) to simulate tab stops at this width.

expandtab
    Enabling this will make the tab key (in insert mode) insert spaces instead of tab characters.
    This also affects the behavior of the retab command.

smarttab
    Enabling this will make the tab key (in insert mode) insert spaces or tabs to go to the next indent
    of the next tabstop when the cursor is at the beginning of a line (i.e. the only preceding characters are whitespace).

For more details on any of these see :help 'optionname' in vim (e.g. :help 'tabstop')
```
