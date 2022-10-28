---
layout: post
title: RTags
---

This is a guide to installing [RTags](https://github.com/Andersbakken/rtags) on linux, and then using it in [spacemacs](http://spacemacs.org/)

The distro I use in this post is Ubuntu 16.04, but most of the information should be general in nature.

We will install RTags' dependencies, build and install RTags itself, build an RTags tag database for
our project, and then enable the rtags layer in spacemacs.

## Dependencies

According to the [Installing rtags](https://github.com/Andersbakken/rtags#installing-rtags) section of the Readme,
we need to have the following dependencies (starred ones are optional):

- clang-3.3 or higher
- cmake-2.8 or higher
- pkg-config*
- bash-completion*
- lua*

I am using Ubuntu 16.04, so the default versions offered by the Ubuntu package manager are all sufficient

    sudo apt install clang libclang-dev cmake pkg-config bash-completion lua

## Download, build and install RTags

    cd /tmp
    git clone --recursive git@github.com:Andersbakken/rtags.git
    cd rtags
    mkdir build && cd build
    cmake ..
    make -j8
    sudo make install

## Configure a service in systemd

Create a systemd config directory for your user

    mkdir -p ~/.config/systemd/user

### Create the RTags daemon socket service config

    $ vim ~/.config/systemd/user/rdm.socket

Add the following:    

    [Unit]
    Description=RTags daemon socket

    [Socket]
    ListenStream=%h/.rdm

    [Install]
    WantedBy=multi-user.target

### Create the RTags daemon service config

    $ vim ~/.config/systemd/user/rdm.service

Add the following

    [Unit]
    Description=RTags daemon

    Requires=rdm.socket

    [Service]
    Type=simple
    ExecStart=/usr/local/bin/rdm --log-file=%h/.rtags/rdm.log --data-dir=%h/.rtags/rtags-cache --verbose --inactivity-timeout 300

Enable and start the socket service

    systemctl --user enable rdm.socket
    systemctl --user start rdm.socket

## Build an RTags database for your project

Since RTags uses clang to parse your code, it needs to know the compiler flags your project uses.

There are [multiple ways](https://github.com/Andersbakken/rtags#setup) to tell RTags what flags you use.

My project uses cmake, so I will use that.

Using cmake we can generate a `compile_commands.json` file which RTags will use to index the project

    cd ~/src/project
    mkdir build && cd build
    cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=1 ..

Now we can instruct RTags to index our code

    rc -J ./compile_commands.json

This takes a while to complete, we can watch the progress by looking at the log file we configured in `~/.config/systemd/user/rdm.service`

    tail -f ~/.rtags/rdm.log

## Enable RTags in spacemacs

We now need to set up RTags support in emacs, and configure some keybindings.

For this I'm using [mineo-rtags](https://github.com/mineo/dotfiles/tree/master/spacemacs/.emacs.d/private/layers/mineo-rtags)

I copied the 2 files `packages.el` and `keybindings.el` into a folder in my spacemacs config directory

    mkdir ~/.spacemacs.d/mineo-rtags && cd ~/.spacemacs.d/mineo-rtags
    wget https://raw.githubusercontent.com/mineo/dotfiles/master/spacemacs/.emacs.d/private/layers/mineo-rtags/packages.el
    wget https://raw.githubusercontent.com/mineo/dotfiles/master/spacemacs/.emacs.d/private/layers/mineo-rtags/keybindings.el

For posterity, here is the contents of these two files at the time of writing:

`packages.el`:

```
(defconst mineo-rtags-packages
  '(cmake-ide
    rtags))

(defun mineo-rtags/init-cmake-ide ()
  (use-package cmake-ide
    :config
    (cmake-ide-setup)))

(defun mineo-rtags/init-rtags ()
  (use-package rtags
    :config
    (setq rtags-autostart-diagnostics t
          rtags-completions-enabled t
          rtags-use-helm t)
    (push '(company-rtags)
          company-backends-c-mode-common)
    (rtags-enable-standard-keybindings)
    (add-hook 'c-mode-common-hook 'rtags-start-process-unless-running))
  (use-package flycheck-rtags
    :ensure rtags))
```

`keybindings.el`:

```
(defconst mineo-rtags-overrides
  '(("C-]" 'rtags-find-symbol-at-point)
    ("M-." 'rtags-find-symbol-at-point)))

(defun mineo-rtags-set-evil-keys ()
  (dolist (override mineo-rtags-overrides)
    (evil-local-set-key 'normal (car override) (cdr override))))

(add-hook 'c-mode-common-hook 'mineo-rtags-set-evil-keys)

;;; https://github.com/mheathr/spacemacs/blob/develop/contrib/!lang/c-c%2B%2B/packages.el

(dolist (mode '(c-mode c++-mode))
  (evil-leader/set-key-for-mode mode
    "g ." 'rtags-find-symbol-at-point
    "g ," 'rtags-find-references-at-point
    "g v" 'rtags-find-virtuals-at-point
    "g V" 'rtags-print-enum-value-at-point
    "g /" 'rtags-find-all-references-at-point
    "g Y" 'rtags-cycle-overlays-on-screen
    "g >" 'rtags-find-symbol
    "g <" 'rtags-find-references
    "g [" 'rtags-location-stack-back
    "g ]" 'rtags-location-stack-forward
    "g D" 'rtags-diagnostics
    "g G" 'rtags-guess-function-at-point
    "g p" 'rtags-set-current-project
    "g P" 'rtags-print-dependencies
    "g e" 'rtags-reparse-file
    "g E" 'rtags-preprocess-file
    "g R" 'rtags-rename-symbol
    "g M" 'rtags-symbol-info
    "g S" 'rtags-display-summary
    "g O" 'rtags-goto-offset
    "g ;" 'rtags-find-file
    "g F" 'rtags-fixit
    "g L" 'rtags-copy-and-print-current-location
    "g X" 'rtags-fix-fixit-at-point
    "g B" 'rtags-show-rtags-buffer
    "g I" 'rtags-imenu
    "g T" 'rtags-taglist
    "g h" 'rtags-print-class-hierarchy
    "g a" 'rtags-print-source-arguments))

(provide 'keybindings)
```

### Enable mineo-rtags layer in spacemacs

    dotspacemacs-configuration-layers
    '(
      (c-c++
        :variables
        c-c++-enable-clang-support t
      )
     syntax-checking
     mineo-rtags
     )

Restart spacemacs and it should install `rtags` and its dependencies.

The keybindings are available under `, g`
