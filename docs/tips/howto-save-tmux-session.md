---
title: How to save tmux session
description: Save tmux session and restore it later
tags: ["tmux"]
---

# How to save current tmux session

To preserve your `tmux` session across reboots, you can use a combination of `tmux` and a tool called `tmux-resurrect`. `tmux-resurrect` allows you to save and restore `tmux` sessions, including the state of each pane and window.

Here are the steps to install and use `tmux-resurrect`:

1. **Install `tmux-resurrect`**:

    First, you need to install `tmux-resurrect`. You can do this by cloning the repository and adding it to your `tmux` configuration.

    ```sh
    git clone https://github.com/tmux-plugins/tmux-resurrect ~/tmux-resurrect
    ```

2. **Configure `tmux` to use `tmux-resurrect`**:

    Add the following lines to your `~/.tmux.conf` file to enable `tmux-resurrect`:

    ```sh
    set -g @plugin 'tmux-plugins/tmux-resurrect'
    run-shell ~/tmux-resurrect/resurrect.tmux
    ```

    If you don't have a `~/.tmux.conf` file, you can create one.

3. **Reload `tmux` configuration**:

    After updating your `~/.tmux.conf`, reload the `tmux` configuration by running the following command inside a `tmux` session:

    ```sh
    tmux source-file ~/.tmux.conf
    ```

4. **Save the `tmux` session**:

    Before rebooting, save your `tmux` session by pressing the following key combination inside a `tmux` session:

    ```sh
    Prefix + Ctrl-s
    ```

    The default `Prefix` key is `Ctrl-b`. So, you would press `Ctrl-b` followed by `Ctrl-s`.

5. **Reboot your system**:

    You can now safely reboot your system.

    ```sh
    sudo reboot
    ```

6. **Restore the `tmux` session**:

    After the system reboots, start a new `tmux` session and restore the saved session by pressing the following key combination inside the new `tmux` session:

    ```sh
    Prefix + Ctrl-r
    ```

    Again, the default `Prefix` key is `Ctrl-b`. So, you would press `Ctrl-b` followed by `Ctrl-r`.


By following these steps, you can save and restore your `tmux` sessions across reboots, ensuring that you don't lose your work.
