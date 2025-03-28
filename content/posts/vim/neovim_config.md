+++
title = 'Neovim_config'
date = 2025-03-28T11:34:07+08:00
draft = false
+++

## load plugin

`h 'runtimepath'`

```md
	List of directories to be searched for these runtime files:
	  filetype.lua	filetypes |new-filetype|
	  autoload/	automatically loaded scripts |autoload-functions|
	  colors/	color scheme files |:colorscheme|
	  compiler/	compiler files |:compiler|
	  doc/		documentation |write-local-help|
	  ftplugin/	filetype plugins |write-filetype-plugin|
	  indent/	indent scripts |indent-expression|
	  keymap/	key mapping files |mbyte-keymap|
	  lang/		menu translations |:menutrans|
	  lsp/		LSP client configurations |lsp-config|
	  lua/		|Lua| plugins
	  menu.vim	GUI menus |menu.vim|
	  pack/		packages |:packadd|
	  parser/	|treesitter| syntax parsers
	  plugin/	plugin scripts |write-plugin|
	  queries/	|treesitter| queries
	  rplugin/	|remote-plugin| scripts
	  spell/	spell checking files |spell|
	  syntax/	syntax files |mysyntaxfile|
	  tutor/	tutorial files |:Tutor|

```

`:help 'write-plugin'`


```md
10. Load the plugin scripts.					*load-plugins*
	This does the same as the command: >
		:runtime! plugin/**/*.{vim,lua}
<	The result is that all directories in 'runtimepath' will be searched
	for the "plugin" sub-directory and all files ending in ".vim" or
	".lua" will be sourced (in alphabetical order per directory),
	also in subdirectories. First "*.vim" are sourced, then "*.lua" files,
	per directory.

	However, directories in 'runtimepath' ending in "after" are skipped
	here and only loaded after packages, see below.
	Loading plugins won't be done when:
	- The |'loadplugins'| option was reset in a vimrc file.
	- The |--noplugin| command line argument is used.
	- The |--clean| command line argument is used.
	- The "-u NONE" command line argument is used |-u|.
	Note that using `-c 'set noloadplugins'` doesn't work, because the
	commands from the command line have not been executed yet.  You can
	use `--cmd 'set noloadplugins'` or `--cmd 'set loadplugins'` |--cmd|.

	Packages are loaded.  These are plugins, as above, but found in the
	"start" directory of each entry in 'packpath'.  Every plugin directory
	found is added in 'runtimepath' and then the plugins are sourced.  See
	|packages|.

	The plugins scripts are loaded, as above, but now only the directories
	ending in "after" are used.  Note that 'runtimepath' will have changed
	if packages have been found, but that should not add a directory
	ending in "after".

```

That mean you can write your plugin in plugin/ dictrcoty, neovim will load
it automatically

```txt
~/.config/nvim
❯ tree                                        
.
├── init.lua
└── plugin
    ├── 01_hi.lua
    ├── 10_options.lua
    ├── 11_mappings.lua
    ├── 12_funcations.lua
    ├── 20_mini.lua
    └── 21_plugins.lua

2 directories, 7 files

```
