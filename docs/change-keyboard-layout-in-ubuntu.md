# Ubuntu 下修改键盘映射

习惯了把 `Ctrl` 键 和 `Caps Lock` 键交换，在 Linux 下如果也要修改这个映射的话。可以直接修改 `/usr/share/X11/xkb/keycodes/evdev` 这个键盘映射文件。



修改之前，记得备份。

打开 `/usr/share/X11/xkb/keycodes/evdev` 文件，找到 `<CAPS> = 66;` 和 `<LCTL>=37;` 这两行（你系统上的文件的值可能不一样）。

然后把这两个值对换，变成  `<CAPS> = 37;` 和 `<LCTL>=66;` 。然后注销重新登录就生效了。

