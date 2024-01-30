Title: Self hosting personal blog
Date: 2024-01-10
Tags: tools, blog, babashka, clojure
Twitter-Handle: saak3t
Description: Hosting your personal blog

> This post is aimed at a fairly technical audience. It expects familiarity with *clojure*, *babashka*, *git*, *github*, *cloudflare* and *markdown* syntax.

**Prerequisites**: Install babashka and git

As mentioned in my previous [blog](/new-year-2024), I had a wish to start writing in public for the last couple of years. Mostly to frame my own thoughts in a structured manner and to keep a public record so others can relate or criticize.

Recently I found the amazing [quickblog](https://github.com/borkdude/quickblog), that helped me get up and running with my first post within two hours.

Without wasting any more word, let's get started.

## Outline
In this post, we will
* Set up a blog repository using quickblog
* Host it on github
* Write a github action config to generate static files on push to `main` branch
* Set up a CD pipeline on cloudflare to deploy the blog
* Point a custom domain to the blog site

## What is quickblog?
Quickblog is a light-weight static blog engine for clojure and babashka. It allows you to write your posts in markdown, provides live reloading and generates SEO optimized static files.

## Setting up quickblog
Create a directory that'll hold your blogging engine and blog posts. I will call it `blog`. Inside `blog` directory, create a `bb.edn` file. The structure looks like the following:

```bash
➜ tree blog 
blog
└── bb.edn

1 directory, 1 file
```

Let's add *quickblog* as a dependency in our `deps.edn`.

```clojure
;; deps.edn
{:deps {io.github.borkdude/quickblog {:git/tag "v0.3.6"
                                      :git/sha "d00e14b1176416b7d7b88e6608b6975888208355"}}}}
```
Replace the `:git/tag` and `:git/sha` with the latest ones in [quickblog repository](https://github.com/borkdude/quickblog).

Now we will add a babashka task that hooks into `quickblog` api and exposes it as a command line tool.

```clojure
{:deps {...} ;; folded deps map

 :tasks
 {:requires ([quickblog.cli :as cli])
  :init (def opts {:blog-title "<Your Blog Title>"
                   :blog-description "<Short Blog Description>"
                   :about-link "<Your Website Link>"
                   :twitter-handle "<Twitter Handle>"})
  quickblog {:doc "Start blogging quickly! Run `bb quickblog help` for details."
             :task (cli/dispatch opts)}}}
```
Replace all the options, with your own.

Execute `bb quickblog help`, it should print a detailed help text about using quickblog.

## First blog post
Now that we have quickblog setup, we will generate our first post. Quickblog command line has a `new` subcommand that allows us to do just that.

```bash
bb quickblog new --file my-first-blog-post.md --title "My first blog post"
```

The directory structure after the last command should look like the following
```bash
➜  blog tree .
.
├── bb.edn
└── posts
    └── my-first-blog-post.md

2 directories, 2 files
```
Run a live reloading server using `bb quickblog watch`, it spins up a http server on `localhost:1888` by default.

Write your blog post, initialize the `blog` directory as a git repository and commit your markdown changes. You would also want to add a `.gitignore` file that excludes some directories from committing like `.work`, `.lsp` and `.clj-kondo`. We will also exclude the `public` directory from the commit to keep the main branch clean.

## Setting up github action
Via github action, we will
* Switch to `deploy` branch
* Generate the static files
* Commit and push them to the repository

For this, we will need to allow github actions `write` permissions on the repository.

In your repository, go to the **Settings > Actions > General**. In the `Workflow permissions` select **Read and write permissions** for now. Later, you can spericy more granular permissions [documented here](https://docs.github.com/actions/reference/authentication-in-a-workflow#modifying-the-permissions-for-the-github_token).

Create the github action config file: `.github/workflows/deploy.yml`.
```yml
on:
  push:
    branches:
      - main
env:
  TZ: Asia/Kolkata # Replace this with your time zone

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: git config
        run: |
          git config --global user.email "<Github Email>"
          git config --global user.name "<Github Username>"
      - name: Switch to deploy branch
        uses: actions/checkout@v4
        with:
          ref: deploy
          fetch-depth: 0
      - name: Merge main into deploy
        run: git merge origin/main
      - name: Setup babashka
        uses: just-sultanov/setup-babashka@v2
        with:
          version: 1.3.186 # Update this babashka version to latest
      - name: clean
        run: bb quickblog clean
      - name: build
        run: bb quickblog render
      - name: git
        run: |
          export CURRENT_TIME=$(date)
          git add public
          git commit -m "deploy @ ${CURRENT_TIME}"
          git push origin -u deploy
```
Replace the placeholders for github email and username, and other parameters. Apart from that the file should be self explanatory.

We will keep it in our local for now, and push it later to github.

## Setting up cloudflare
Create a new cloudflare account if you don't have it already. Log in to the account and go to **Workers & Pages** in the left panel.

* We will *create an application*
* Go to **Pages** tab.
* And **Connect to Git**

A modal pops up, where you select your Github account and the repository to connect. You'll be redirected to github. Allow access to just the `blog` repository and continue.

Now we go to the next step, add the project name and set the `Production branch` to **deploy** and `Build output directory` to **public**. *Click and deploy*.

Cloudflare will consider changes on all the branches other than the `Production branch`, that we set in the previous step, as preview branches. This means that changes on these other branches will also trigger a build, consuming our precious build quota.

To disable the preview branches, we will go to the project we just setup **-> Settings tab -> Build & deployment -> Configure preview deployments -> None -> Save**. We are done with the setting up our cloudflare page. It should be visible at `<project-identifier>.pages.dev`.

> **Note**: Create the main and deploy branches and push them to our github repository. This will trigger github action build, after which cloudflare page build and you should see you first blog up on the url mentioned above.

## Setting up custom domain
Here I am assuming that you already have a domain bought and configured on cloudflare. If you don't, then there are a couple of platforms that I use to find and get best offer for domains:

* [Namecheap](https://www.namecheap.com/)
* [GoDaddy](https://www.godaddy.com/)
* [BigRock](https://www.bigrock.in)
* [Hostinger](https://www.hostinger.in/domain-name-search)

Once you get a domain, transfer it to cloudflare by following [these steps](https://developers.cloudflare.com/registrar/get-started/transfer-domain-to-cloudflare/).

Setting up a custom domain for your page is simple. Open our page project, and go to **Custom domains** tab. Click on **Set up a custom domain** and follow the steps.

> **PS**: There are multiple ways to host your blog, and probably most of them would be simpler than this setup. It was my first time setting up my blog, and can certainly be improved. Will post an update when I migrate to a better pipeline. A similar post: [Quickblog by Anders Means Different](https://www.eknert.com/blog/quickblog)

> If you find an error or have a question, open an [issue](https://github.com/Samy-33/blog/issues/new).
