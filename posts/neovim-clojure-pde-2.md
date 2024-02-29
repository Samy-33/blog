Title: Neovim as the Clojure PDE - II
Date: 2024-02-29
Tags: clojure, neovim, fennel, lua, text-editor

[Click here](/neovim-clojure-pde-1) if you missed the first part.

Welcome back, folks. It's been a while since the first post of this series came out.

Today's agenda is to install some basic plugins and configure them. While doing that, we'll learn some fennel and have our own config in parallel.

### Why Plugins?
Plugins enhance the raw editor by adding features that the raw Neovim doesn't provide out of the box. In VSCode world, we call them extensions.

We need a plugin that makes it easy for us to install and use other plugins. These are called `Plugin Managers`. There are many of them in Neovim/Vim world like [Packer](https://github.com/wbthomason/packer.nvim), [Vim Plug](https://github.com/junegunn/vim-plug), [Lazy](https://github.com/folke/lazy.nvim) et al. For the purpose of this post and because it's the new shiny thing, we will use **Lazy**.

### plugins.fnl
Just to recap, after our [last post](/neovim-clojure-pde-1), the config directory looks like the following
```sh
$ tree -L 3 ~/.config/nvim
├── fnl
│   └── user
│       └── core.fnl
├── init.lua
└── lua
    └── user
        └── core.lua
```

We are going to add the following plugins now:
* [lsp-config](https://github.com/neovim/nvim-lspconfig) - Language Server Protocol client for neovim
* [nvim-treesitter](https://github.com/nvim-treesitter/nvim-treesitter) - Allows to parse code into abstract syntax tree and use that in better syntax highlighting, code traversal et al.
* [nvim-cmp](https://github.com/hrsh7th/nvim-cmp) - Autocompletion for neovim
* [telescope.nvim](https://github.com/nvim-telescope/telescope.nvim) - Fuzzy finder for neovim
* [conjure](https://github.com/Olical/conjure) - Allows you to hook into REPl servers
* [nvim-paredit](https://github.com/Olical/conjure) - S-expression motions made easy
* [rainbow-delimiters](https://github.com/HiPhish/rainbow-delimiters.nvim) - Colorful parenthesis
* [autopairs](https://github.com/windwp/nvim-autopairs) - Allows working with pair characters easy

We'll put all the plugin installation related configuration in the `plugins.fnl` file. Let's start with installing `Lazy`, a lua based plugin manager for neovim.

```clojure
;; fnl/user/plugins.fnl

;; Path to directory where lazy itself will be installed.
;; `(vim.fn.stdpath :data)` returns standard directory where nvim's data will be stored.
;; `..` is string concatenation in fnl.
(local lazy-path (.. (vim.fn.stdpath :data)
                     :/lazy/lazy.nvim))

;; `(vim.uv.fs_stat lazy-path)` returns nil if the `lazy-path` doesn't exist.
;; If the directory exists, it returns info about the directory.
(local lazy-installed? (vim.uv.fs_stat lazy-path))

;; We'll store plugins we need to install in this variable.
(local to-install [])

;; `setup` installs lazy plugin manager if not already installed, and also downloads the plugins added to `to-install`.
;; This is an exported function that is called from the parent module.
(fn setup []
  (when (not lazy-installed?)
    (vim.fn.system [:git
                    :clone
                    "--filter=blob:none"
                    "https://github.com/folke/lazy.nvim.git"
                    :--branch=stable
                    lazy-path]))
  (vim.opt.rtp:prepend lazy-path)
  (let [lazy (autoload :lazy)]
    (lazy.setup plugins-to-install)))

;; Exporting the setup function.
{: setup}
```

Let's import `plugins.fnl` in `fnl/user/core.fnl` and call the `setup` function.

```clojure
;; fnl/user/core.fnl
(local plugins (require :user.plugins))

(fn setup []
  (plugins.setup))

{: setup}
```

In our `init.lua`, we need to call `setup` function of our `core` module.

```lua
-- init.lua
-- ...

require('user.core').setup()
```

Now, when we restart the neovim, the lazy plugin should be installed.

It is time for us to add the plugins we listed above.

```clojure
;; fnl/user/plugins.fnl

;; ...
(local to-install
  [{1 :neovim/nvim-lspconfig
    :dependencies [{1 :williamboman/mason.nvim :config true} ;; Mason handles lsp servers for us and others
                   :williamboman/mason-lspconfig.nvim ;; default lspconfigs for language servers
                   {1 :echasnovski/mini.nvim ;; For lsp progress notification
                    :version false
                    :config (fn [] 
                              (let [mnotify (require :mini.notify)]
                                (mnotify.setup {:lsp_progress {:enable true
                                                               :duration_last 2000}})))}
                   :folke/neodev.nvim]} ;; neodev for lua neovim dev support.

   ;; Autocompletion   
   {1 :hrsh7th/nvim-cmp
    :dependencies [:hrsh7th/cmp-nvim-lsp]}

   ;; Fuzzy Finder (files, lsp, etc)
   {1 :nvim-telescope/telescope.nvim
    :version "*"
    :dependencies {1 :nvim-lua/plenary.nvim}}
   ;; Fuzzy Finder Algorithm which requires local dependencies to be built.
   ;; Only load if `make` is available. Make sure you have the system
   ;; requirements installed.
   {1 :nvim-telescope/telescope-fzf-native.nvim
    ;; NOTE: If you are having trouble with this installation,
    ;; refer to the README for telescope-fzf-native for more instructions.
    :build :make
    :cond (fn [] (= (vim.fn.executable :make) 1))}

   ;; Treesitter
   {1 :nvim-treesitter/nvim-treesitter
    :build ":TSUpdate"}
  
   ;; Conjure and related plugins
   :Olical/conjure
   :PaterJason/cmp-conjure ;; autocomplete using conjure
   :tpope/vim-dispatch
   :clojure-vim/vim-jack-in
   :radenling/vim-dispatch-neovim
   :tpope/vim-surround

   ;; Paredit
   {1 :julienvincent/nvim-paredit
    :config (fn []
              (let [paredit (require :nvim-paredit)]
                (paredit.setup {:indent {:enabled true}})))}
   ;; Paredit plugin for fennel
   {1 :julienvincent/nvim-paredit-fennel
    :dependencies [:julienvincent/nvim-paredit]
    :ft [:fennel]
    :config (fn []
              (let [fnl-paredit (require :nvim-paredit-fennel)]
                (fnl-paredit.setup)))}

   ;; Rainbow parens
   :HiPhish/rainbow-delimiters.nvim

   ;; Autopairs
   {1 :windwp/nvim-autopairs
    :event :InsertEnter
    :opts {:enable_check_bracket_line false}}])
```
Restart neovim, and we see Lazy UI installing the plugins added above.

### Common configuration
Let's configure Neovim to solve some day-to-day pains eg. enabling relative line numbering, auto syncing system and neovim clipboards, some useful keymaps etc.

We'll create a `general.fnl` where we put all of this.
```clojure
;; fnl/user/config/general.fnl

(fn setup []
  ;; disables highlighting all the search results in the doc
  (set vim.o.hlsearch false)

  ;; line numbering
  (set vim.wo.number true)
  (set vim.wo.relativenumber true)

  ;; disable mouse
  (set vim.o.mouse "")

  ;; clipboard is system clipboard
  (set vim.o.clipboard :unnamedplus)

  ;; Some other useful opts. `:help <keyword>` for the help.
  ;; For example: `:help breakindent` will open up vimdocs about `vim.o.breakindent` option.
  (set vim.o.breakindent true)
  (set vim.o.undofile true)
  (set vim.o.ignorecase true)
  (set vim.o.smartcase true)
  (set vim.wo.signcolumn :yes)
  (set vim.o.updatetime 250)
  (set vim.o.timeout true)
  (set vim.o.timeoutlen 300)
  (set vim.o.completeopt "menuone,noselect")
  (set vim.o.termguicolors true)
  (set vim.o.cursorline true)

  ;; Keymaps
  (vim.keymap.set [:n :v] :<Space> :<Nop> {:silent true})

  ;; To deal with word wrap
  (vim.keymap.set :n :k "v:count == 0 ? 'gk' : 'k'" {:expr true :silent true})
  (vim.keymap.set :n :j "v:count == 0 ? 'gj' : 'j'" {:expr true :silent true})

  ;; Tabs (Optional and unrelated to this tutorial, but helpful in handling tab movement)
  (vim.keymap.set [:n :v :i :x] :<C-h> #(vim.api.nvim_command :tabprevious)
                  {:silent true})
  (vim.keymap.set [:n :v :i :x] :<C-l> #(vim.api.nvim_command :tabnext)
                  {:silent true})
  (vim.keymap.set [:n :v :i :x] :<C-n> #(vim.api.nvim_command :tabnew)
                  {:silent true}))

;; exports an empty map
{: setup}
```

### Plugin Configurations
It is time for us to configure each of these plugins to cater to our needs. These needs are plugin specific. We may want to add keymaps for commands we use often or tell Mason to make sure that particular LSP is always installed, or anything else that the plugin supports. These settings and configs can often be found on the plugin's github repository and neovim's documentation.

#### LSP configuration
```clojure
;; fnl/user/config/lsp.fnl

(local nfnl-c           (require :nfnl.core))
(local ts-builtin       (require :telescope.builtin))
(local cmp-nvim-lsp     (require :cmp_nvim_lsp))
(local mason-lspconfig  (require :mason-lspconfig))
(local lspconfig        (require :lspconfig))
(local neodev           (require :neodev))

;; This function will be executed when neovim attaches to an LSP server.
;; This basically sets up some widely used keymaps to interact with LSP servers.
(fn on-attach [_ bufnr]
  (let [nmap (fn [keys func desc]
               (vim.keymap.set :n keys func
                               {:buffer bufnr 
                                :desc   (.. "LSP: " desc)}))
        nvxmap (fn [keys func desc]
                 (vim.keymap.set [:n :v :x] keys func
                                 {:buffer bufnr :desc (.. "LSP: " desc)}))]
    (nmap :<leader>rn vim.lsp.buf.rename "[R]e[n]ame")
    (nmap :<leader>ca vim.lsp.buf.code_action "[C]ode [A]ction")
    (nmap :gd vim.lsp.buf.definition "[G]oto [D]efinition")
    (nmap :gr (fn [] (ts-builtin.lsp_references {:fname_width 60}))
          "[G]oto [R]eferences")
    (nmap :gI vim.lsp.buf.implementation "[G]oto [I]mplementation")
    (nmap :<leader>D vim.lsp.buf.type_definition "Type [D]efinition")
    (nmap :<leader>ds ts-builtin.lsp_document_symbols "[D]ocument [S]ymbols")
    (nmap :<leader>ws
          (fn [] (ts-builtin.lsp_dynamic_workspace_symbols {:fname_width 60}))
          "[W]orkspace [S]ymbols")
    (nmap :K vim.lsp.buf.hover "Hover Documentation")
    (nmap :<C-k> vim.lsp.buf.signature_help "Signature Documentation")
    (nmap :gD vim.lsp.buf.declaration "[G]oto [D]eclaration")
    (nmap :<leader>wa vim.lsp.buf.add_workspace_folder
          "[W]orkspace [A]dd folder")
    (nmap :<leader>wr vim.lsp.buf.remove_workspace_folder
          "[W]orkspace [R]emove folder")
    (nmap :<leader>wl
          (fn []
            (nfnl-c.println (vim.inspect (vim.lsp.buf.list_workspace_folders))))
          "[W]orkspace [L]ist folders")
    (nvxmap :<leader>fmt vim.lsp.buf.format
            "[F]or[m]a[t] the current buffer or range")))

;; This binding keeps a map from lsp server name and its settings.
(local servers
       {:clojure_lsp            {:paths-ignore-regex :conjure-log-*.cljc}
        :lua_ls                 {:Lua {:workspace {:checkThirdParty false}
                                       :telemetry {:enable false}}}
        :fennel_language_server {:fennel {:diagnostics  {:globals [:vim]}
                                          :workspace    {:library (vim.api.nvim_list_runtime_paths)}}}})

(local capabilities
       (cmp-nvim-lsp.default_capabilities (vim.lsp.protocol.make_client_capabilities)))

(fn setup []
  (neodev.setup)
  (mason-lspconfig.setup {:ensure_installed (nfnl-c.keys servers)})
  (mason-lspconfig.setup_handlers [(fn [server-name]
                                     ((. (. lspconfig server-name) :setup) 
                                        {: capabilities
                                         :on_attach on-attach
                                         :settings (. servers server-name)

                                         ;; This is required because clojure-lsp doesn't send LSP progress messages if the `workDoneToken` is not sent from client. It is an arbitrary string.
                                         :before_init
                                         (fn [params _]
                                           (set params.workDoneToken 
                                                "work-done-token"))}))]))

{: setup}
```

#### Autocompletion configuration
```clojure
;; fnl/user/config/cmp.fnl
(local cmp (require :cmp))

(fn setup []
  (cmp.setup {:mapping (cmp.mapping.preset.insert 
                         {:<C-d>     (cmp.mapping.scroll_docs -4)
                          :<C-f>     (cmp.mapping.scroll_docs 4)
                          :<C-Space> (cmp.mapping.complete {})
                          :<CR>      (cmp.mapping.confirm {:behavior cmp.ConfirmBehavior.Replace
                                                           :select false})
                          :<Tab>     (cmp.mapping (fn [fallback]
                                                    (if
                                                      (cmp.visible)
                                                      (cmp.select_next_item)

                                                      ;; else
                                                      (fallback))))
                          :<S-Tab>   (cmp.mapping (fn [fallback]
                                                    (if 
                                                      (cmp.visible)
                                                      (cmp.select_prev_item)
                                                      
                                                      ;; else
                                                      (fallback))))}                        
                         [:i :s])
              :sources [{:name :conjure}
                        {:name :nvim_lsp}]}))

{: setup}
```

#### Paredit configuration
```clojure
;; fnl/user/config/paredit.fnl

(local paredit      (require :nvim-paredit))
(local paredit-fnl  (require :nvim-paredit-fennel))

(fn setup []
  (paredit.setup {:indent {:enabled true}})
  (paredit-fnl.setup))

{: setup}
```

#### Rainbow delimiters configuration
```clojure
;; fnl/user/config/rainbow.fnl

(local rainbow-delimiters (require :rainbow-delimiters))

(fn setup []
  (set vim.g.rainbow_delimiters
       {:strategy {"" rainbow-delimiters.strategy.local}
        :query    {"" :rainbow-delimiters}}))

{: setup}
```

#### Telescope configuration
```clojure
;; fnl/user/config/telescope.fnl

(local telescope  (require :telescope))
(local builtin    (require :telescope.builtin))
(local themes     (require :telescope.themes))

(fn setup []
  (telescope.setup {:defaults {:mappings {:i {:<C-u> false :<C-d> false}}}})
  (telescope.load_extension :fzf)

  ;; keymaps
  (vim.keymap.set :n :<leader>? builtin.oldfiles
                  {:desc "[?] Find recently opened files"})
  (vim.keymap.set :n :<leader>fb builtin.buffers {:desc "[F]ind open [B]uffers"})
  (vim.keymap.set :n :<leader>/
                  (fn []
                    (builtin.current_buffer_fuzzy_find (themes.get_dropdown {:winblend 10
                                                                             :previewer false})))
                  {:desc "[/] Fuzzily search in current buffer"})
  (vim.keymap.set :n :<leader>ff builtin.find_files {:desc "[F]ind [F]iles"})
  (vim.keymap.set :n :<leader>fh builtin.help_tags {:desc "[F]ind [H]elp"})
  (vim.keymap.set :n :<leader>fw builtin.grep_string
                  {:desc "[F]ind current [W]ord"})
  (vim.keymap.set :n :<leader>fg builtin.live_grep {:desc "[F]ind by [G]rep"})
  (vim.keymap.set :n :<leader>fd builtin.diagnostics
                  {:desc "[F]ind [D]iagnostics"})
  (vim.keymap.set :n :<leader>fcf
                  (fn [] (builtin.find_files {:cwd (vim.fn.stdpath :config)}))
                  {:desc "[F]ind [C]onfing [F]iles"})
  (vim.keymap.set :n :<leader>fch builtin.command_history
                  {:desc "[F]ind in [C]ommands [H]istory"})
  (vim.keymap.set :n :<leader>fm builtin.marks {:desc "[F]ind in [M]arks"}))

{: setup}
```

#### Treesitter configuration
```clojure
;; fnl/user/config/treesitter.fnl

(local configs (require :nvim-treesitter.configs))

(local ensure-installed [:clojure :lua :vimdoc :vim :fennel])

(fn setup []
  (configs.setup {:ensure_installed       ensure-installed
                  ;; Add languages to be installed here that you want installed for treesitter
                  :auto_install           true
                  :highlight              {:enable true}
                  :indent                 {:enable  false}}))

{: setup}
```

### Gluing everything together
Let's actually call `setup` methods of all these modules in our `core.fnl`.
```clojure
;; fnl/user/core.fnl
(local plugins          (require :user.plugins))
(local u-general        (require :user.config.general))
(local u-telescope      (require :user.config.telescope))
(local u-treesitter     (require :user.config.treesitter))
(local u-lsp            (require :user.config.lsp))
(local u-cmp            (require :user.config.cmp))
(local u-paredit        (require :user.config.paredit))
(local u-rainbow        (require :user.config.rainbow))

(fn setup []
  (plugins.setup)
  (u-general.setup)
  (u-telescope.setup)
  (u-treesitter.setup)
  (u-lsp.setup)
  (u-cmp.setup)
  (u-paredit.setup)
  (u-rainbow.setup))

{: setup}
```

And that should be it. I know, this blog is long and contains a lot of code. Most of it should be self explanatory once you get familiar with the fennel syntax and vim docs.

The configuration is inspired by [kickstart.nvim](https://github.com/nvim-lua/kickstart.nvim). More extensive configuration can be found on my [dotfiles](https://github.com/Samy-33/dotfiles/tree/master/dotcom/.config/nvim).

> If there's a bug somewhere in here or you have a doubt, feel free to open an issue [here](https://github.com/Samy-33/blog/issues).
