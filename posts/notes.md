Title: Notes
Date: 2024-11-26
Tags: til,findings,keeps

## Why?
Keeping a list of public notes, that I found interesting or useful in my day to day work.

## Content
* [Exposing local server to Internet](#exposing_local_server_to_internet)

### Exposing local server to Internet
Cloudflare makes `cloudflared` available for tunneling purpose.

- Install by `sudo pacman -Sy cloudflared`
- Run to expose a http server running on port 3000 by `cloudflared tunnel --url http://localhost:3000`
- We get a public url like: `https://pollution-lmao-occasions-dracony.trycloudflare.com`
