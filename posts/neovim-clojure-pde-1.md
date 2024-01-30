Title: Neovim as the Clojure PDE - I
Date: 2024-01-26
Tags: clojure, neovim, fennel, lua, text-editor

Hello, fellow text editor enthusiasts. Welcome to *Abstract Symphony*, meaning of which, nobody knows and has nothing to do with the post's agenda.

You may gently ask for the meaning of PDE, what the fuck does that mean?

Well, you don't have to be so rude. It's an abbreviation for *Personalized Development Environment*, coined by [TJ](https://twitter.com/teej_dv), Neovim's marketing head.

### Why Neovim?
You know, same old bullshit like speed, muscle memory, extension of your body/mind, spiritualism, nirvana, moksha and so on. Nothing serious.
Jokes aside, the real reason was seeing how fast [Prime](https://twitter.com/ThePrimeagen) was, in [giving up](https://youtu.be/SGBuBSXdtLY?si=0HJtUqZEIT3B3izX&t=69) on clojure.

### Why Clojure?
Because I love my high functioning parens. I can lisp down, 100s of reasons why, but that's beyond the scope. This emotion is hosted on a solid and reliable foundation. I am sure, a dynamic and immutable relationship is what keeps us tied together so strongly. Please forgive me, for my love for punning (nil?). [IYKYK](https://clojure.org/about/rationale).

### init.lua
Neovim is a professional's tool. You need to deserve it, earn its respect, you know? Like Thor and his hammer.

Or you can just [install](https://github.com/neovim/neovim/blob/master/INSTALL.md) neovim and get started. At first sight, it doesn't look like anything more than a trap that you can never get out of (trust me you can :q!). And no, that was not my failed attempt at putting an emojee.

Create an `init.lua` file in `$HOME/.config/nvim` directory. This will be the entrypoint for your neovim configuration. It's similar to the `main` method in a java/c/cpp projects, an entry-point for the program to run.

Go ahead and add a `print("Hello, world!")` to the file. Now, when you run `nvim`, you should see `Hello, world!` at the bottom-left of your screen. Congratulations, for your first configured neovim instance.

### Leader and LocalLeader
Just like how each country needs good leaders to function properly, neovim is no different. You should define a leader and localleader according to your convenience. People usually choose `<Space>` or `","` as their leader. I'll go ahead with `","` as that is what I am used to.

But wait, what is the purpose of the leaders, you ask? Well, the main reason is the `"plugins"`. Plugin writers are not aware of how you have configured your editor. They can't arbitrarily setup keybindings in their plugins, as they may conflict with your bindings.

Neovim API exposes options via `vim.g`, `vim.o`, `vim.opt`, `vim.bo` and `vim.wo`. We can control behaviour of Neovim by setting these options. `vim.g` is used for `global` scoped options. We can read more about them in Neovim's help text using `:help vim.g` or `:help vim.o` etc.

Let's set our globally scoped leader and localleader.

```lua
-- init.lua
vim.g.mapleader = ','
vim.g.maplocalleader = ','
```

**NSFW**: [This article](https://learnvimscriptthehardway.stevelosh.com/chapters/06.html) goes deeper into how leader/localleader is helpful.

### We'll use Fennel as our config language
Although, Neovim officially supports lua as its configuration language, we will use [Fennel](https://fennel-lang.org/). Because, we love our parens. And also, we like to struggle and learn.

Fennel transpiles into lua, so we need a Neovim plugin to transpile our fennel files into lua. Olical's [nfnl](https://github.com/Olical/nfnl) does exactly that. We will update our `init.lua` to download `nfnl`. We will also add nfnl's path to neovim's runtime path, so that it can find `nfnl` plugin.

```lua
-- init.lua
-- ...

local nfnl_path = vim.fn.stdpath 'data' .. '/lazy/nfnl'

if not vim.uv.fs_stat(nfnl_path) then
  print("Could not find nfnl, cloning new copy to", nfnl_path)
  vim.fn.system({'git', 'clone', 'https://github.com/Olical/nfnl', nfnl_path})
end

vim.opt.rtp:prepend(nfnl_path)

require('nfnl').setup()
```
We define a local variable `nfnl_path`. It holds the directory path where we will download `nfnl`.
* **vim.fn.stdpath 'data'** returns neovim's `data` directory.
> `:help stdpath` to know more about standard paths.
* "**..**" is lua's string concatenation operator. So essentially, we get neovim's standard data path and append `/lazy/nfnl` to it. We'll store `nfnl` here.
* **vim.uv.fs\_stat(nfnl_path)** returns information about file/directory. If the file doesn't exist, it returns nil.
* **vim.fn.system** allows us to execute shell commands. We can also use `vim.system(...)` instead. They are exactly the same.
* We ran `git clone` to download nfnl from its official github repository.
* Then we added `nfnl_path` to the neovim's runtime path.
* At last, we require the `nfnl` module and call `setup` on it.

Let us create a directory to store our fennel config files. And also a `core.fnl` file inside that directory.

```bash
# Creates the directory
mkdir -p $HOME/.config/nvim/fnl/user

# Creates the file
touch $HOME/.config/nvim/fnl/user/core.fnl
```

Let's add a simple `Hello, world!` print form in our `core.fnl`.
```clojure
;; core.fnl
(println "Hello, world! This is fennel config!")
```

When we restart our neovim instance, nothing happens. We should have seen a _hello world_ message. We're missing a key `nfnl` config.

`nfnl` looks for a `.nfnl.fnl` file in a directory for configuration about which files to compile and how. Create a `.nfnl.fnl` file with empty config in `$HOME/.config/nvim` directory.
```bash
echo {} > $HOME/.config/nvim/.nfnl.fnl
```

When we restart our instance again, we still don't see anything printed. Well, there are a couple of things pending.
* Fennel files haven't been transpiled yet. `nfnl` transpiles `fnl` files on save by default.
* In our `init.lua`, we haven't `require`d the `user.core`, so nvim doesn't know about it just yet.

Let's open `core.fnl` and just save it by `:write`. You'll notice a prompt asking you to allow nfnl to transpile the files. Go ahead and allow it. When you look at the `$HOME/.config.nvim` directory, you should have a `lua` directory with transpiled lua code corresponding to the `core.fnl`.

Let's require `user/core` module in `init.lua`
```lua
-- init.lua
-- ...
-- append this to the bottom of init.lua file.
require('user.core')
```

Now, when you restart neovim, it greets you with:
> Hello, world! This is fennel config!

#### To be continued!
