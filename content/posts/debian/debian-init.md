+++
title = 'Debian env init'
date = 2024-02-22T12:00:05+08:00
updated = 2025-02-19T12:06:05+08:00
draft = false
tags = ['linux', 'debian']
+++

## add user

```sh
adduser vb
usermod -aG sudo vb
```

## ssh

```sh
mkdir ~/.ssh
echo 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEY8nuiqZYNWjJMC8I4StHzAcv8pJjHMUCwkvPMaVTWY mac
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIALgDy+FJMEy/UG/bYjnBEAYmVTLH6qOVJyXzpXADoFX pc' > ~/.ssh/authorized_keys
chmod 0700 ~/.ssh
chmod 0600 ~/.ssh/authorized_keys
```

## nvim

### install
```sh
sudo apt install wget -y
wget https://github.com/neovim/neovim/releases/latest/download/nvim-linux-x86_64.tar.gz -O nvim-linux-x86_64.tar.gz
tar -zxf nvim-linux-x86_64.tar.gz
sudo mv nvim-linux-x86_64 /usr/local/nvim
sudo ln -s /usr/local/nvim/bin/nvim /usr/bin/nvim 
```

### config
```sh
mkdir -p ~/.config/nvim
wget https://raw.githubusercontent.com/cubatic45/dotfiles/refs/heads/main/dot_config/nvim/init.lua -O ~/.config/nvim/init.lua
```


## tmux
```sh
apt install tmux -y
echo 'set -s set-clipboard on
set -g allow-passthrough on
set -g mouse on
set-option -g repeat-time 0
set -s escape-time 10

set -g default-terminal "tmux-256color"
set-option -sa terminal-overrides ",xterm-256color:RGB"


set -wg mode-keys vi
bind -T copy-mode-vi v send-keys -X begin-selection
bind -T copy-mode-vi c send-keys -X copy-selection
bind -T copy-mode-vi p send-keys -X copy-selection
bind -T copy-mode-vi r send-keys -X rectangle-toggle

bind -r h select-pane -L
bind -r j select-pane -D
bind -r k select-pane -U
bind -r l select-pane -R


set -g base-index 1     
set -g pane-base-index 1 ' > ~/.tmux.conf
```

## zsh
```sh
apt install zsh git -f

~/.zshrc config 
### Added by Zinit's installer
if [[ ! -f $HOME/.local/share/zinit/zinit.git/zinit.zsh ]]; then
    print -P "%F{33} %F{220}Installing %F{33}ZDHARMA-CONTINUUM%F{220} Initiative Plugin Manager (%F{33}zdharma-continuum/zinit%F{220})…%f"
    command mkdir -p "$HOME/.local/share/zinit" && command chmod g-rwX "$HOME/.local/share/zinit"
    command git clone https://github.com/zdharma-continuum/zinit "$HOME/.local/share/zinit/zinit.git" && \
        print -P "%F{33} %F{34}Installation successful.%f%b" || \
        print -P "%F{160} The clone has failed.%f%b"
fi

source "$HOME/.local/share/zinit/zinit.git/zinit.zsh"
autoload -Uz _zinit
(( ${+_comps} )) && _comps[zinit]=_zinit

# Load a few important annexes, without Turbo
# (this is currently required for annexes)
zinit light-mode for \
    zdharma-continuum/zinit-annex-as-monitor \
    zdharma-continuum/zinit-annex-bin-gem-node \
    zdharma-continuum/zinit-annex-patch-dl \
    zdharma-continuum/zinit-annex-rust

### End of Zinit's installer chunk
#

zinit light zsh-users/zsh-completions
zinit light zsh-users/zsh-autosuggestions
zinit light zsh-users/zsh-history-substring-search
zinit light zdharma/fast-syntax-highlighting

# oh my zsh 
zinit snippet OMZ::lib/completion.zsh
zinit snippet OMZ::lib/history.zsh
zinit snippet OMZ::lib/key-bindings.zsh
zinit snippet OMZ::lib/theme-and-appearance.zsh

# key binding
bindkey '^[[A' history-substring-search-up
bindkey '^[[B' history-substring-search-down


zinit ice pick"async.zsh" src"pure.zsh" # with zsh-async library that's bundled with it.
zinit light sindresorhus/pure

unset zle_bracketed_paste

zi for \
    atload"zicompinit; zicdreplay" \
    blockf \
    lucid \
    wait \
  zsh-users/zsh-completions

```
