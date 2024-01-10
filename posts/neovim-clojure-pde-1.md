Title: Neovim as the Clojure PDE
Date: 2024-01-12
Tags: clojure, neovim, fennel, lua, text-editor

Hello, fellow text editor enthusiasts. Welcome to *Abstract Symphony*, meaning of which, nobody knows and has nothing to do with the post's agenda.

You may gently ask for the meaning of PDE, what the fuck does that mean?

Well, you don't have to be so rude. It's an abbreviation for *Personalized Development Environment*, coined by [TJ](https://twitter.com/teej_dv), Neovim's marketing head.

### Why Neovim?
You know, same old bullshit like speed, muscle memory, extension of your body/mind, spiritualism, nirvana, moksha and so on. Nothing serious. Jokes aside, the real reason was seeing how fast [Prime](https://twitter.com/ThePrimeagen) was, in [giving up](https://youtu.be/SGBuBSXdtLY?si=0HJtUqZEIT3B3izX&t=69) on clojure.

### Why Clojure?
Because I love my high functioning parens. I can lisp down, 100s of reasons why, but that's beyond the scope. This emotion is hosted on a solid and reliable foundation. I am sure, a dynamic and immutable relationship is what keeps us tied together so strongly. Please forgive me, for my love for punning (nil?). [IYKYK](https://clojure.org/about/rationale).

### init.lua
Neovim is a professional's tool. You need to deserve it, earn its respect, you know? Like Thor and his hammer.

Or you can just [install](https://github.com/neovim/neovim/blob/master/INSTALL.md) neovim and get started. At first sight, it doesn't look like anything more than a trap that you can never get out of (trust me you can :q!). And no, that was not my failed attempt at putting an emojee.

Create an `init.lua` file in `$HOME/.config/nvim` directory. This will be the entrypoint for your neovim configuration. It's similar to the `main` method in a java/c/cpp projects, an entry-point for the program to run.

Go ahead and add a `print("Hello, world!")` to the file. Now, when you run `nvim`, you should see `Hello, world!` at the bottom-left of your screen. Congratulations, for your first configured neovim instance.

### Leader and LocalLeader
Just like how each country need good leaders to function properly, neovim is no different. You should define a leader and localleader according to your own convenience. People usually choose `<Space>` or `","` as their leader. I'll go ahead with `","` as that is what I am used to.

But wait, what is the purpose of the leaders, you ask? Well, the main reason is the `"plugins"`. Plugin writers are not aware of how you have configured your editor. They can't arbitrarily configure keybindings in their plugins, as they may conflict with users bindings.
