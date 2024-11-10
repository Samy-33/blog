Title: Clojure Automation #1
Date: 2024-11-07
Tags: clojure, fun, automation

In this post, I share how I automated checking for passport application status using clojure from scratch.

## Content
* [Why?](#why?)
* [100 ft View](#100_ft_view)
* [Project Setup](#project_setup)
* [Scraping Information](#scraping_information)
  * Figuring out the API
  * Parsing Information
* [Telegram Bot Setup](#telegram_bot_setup)
  * Creating the bot and channel
  * Sending message
* [Deployment](#deployment)
* [Conclusion](#conclusion)

## Why?
My whole family has applied for new passports, and I want to know the application status. But every time I open the [website](https://www.passportindia.gov.in/AppOnlineProject/welcomeLink), I feel like giving up on life.

What could be worse, you say? Well, doing it four times.

Now, I use `Check Application Status` option on the homepage. It's an optimization by storing `File Numbers` and `Date of Births` in my BitWarden extension. That helps avoid having to fill the Captcha and logging in multiple times. It still hurts, albeit a bit less.

I abhor the 2 minutes, I spend each time I have to check the status. So, I'll spend next 5 hours setting up an automation script to notify me every 3 hours.

## 100 ft View
I will need the following:
* A job running every 3 hours
* A way to fetch passport application status
* A way to notify me

This is how it's going to work:
* At the onset of each 3rd hour, run a script
* Script will scrape the information from passport status webpage
* Parse out required information from the scraped page
* Send a notification to a chat app (whatsapp or telegram)

## Project Setup
I start by defining the `deps.edn` file in the project root. This is the beginning of the project. `deps.edn` will contains the project dependencies and configuration for build and release.

Which dependencies will I need?
* To scrape information
  * HTTP Client
  * HTML Parser and Selector
  * JSON encoder and decoder to convert from clojure data structures to JSON
* To send messages to Telegram
  * Again the same dependencies, except HTML Parser
* For development a REPL server library

`deps.edn` file looks like the following:

```clojure
; deps.edn
{:paths ["src"]                                                     ; Where the source code lives
 :deps {clj-http/clj-http {:mvn/version "3.13.0"}                   ; HTTP Client
        org.clj-commons/hickory {:mvn/version "0.7.5"}              ; HTML Parser and Selector
        cheshire/cheshire {:mvn/version "5.13.0"}}                  ; JSON encoder/decoder
 :aliases {:dev {:extra-paths ["env/dev"]                           ; dev environment config
                 :extra-deps {nrepl/nrepl {:mvn/version "1.3.0"}}   ; nrepl server development
                 :main-opts ["-m" "nrepl.cmdline"]}                 ; Options for main function in `dev` alias
           :prod {:extra-paths ["env/prod"]}                        ; Prod config
           :passport {:exec-fn passport-status/execute!}}}          ; Alias to execute actual flow
```

Now I can start the REPL server by executing the following command in the terminal:
```sh
$ clj -M:dev
```

As an output of the command, I'll get the `nrepl-port` that I can use to connect to the REPL server from the editor. I can also use the `.nrepl-port` file created by the above command.

I also create the following directory structure:
```sh

➜  automations git:(master) ✗ tree .
.
├── deps.edn
├── env
│   ├── dev
│   │   └── config.clj
│   └── prod
│       └── config.clj
└── src
    ├── passport_status.clj
    └── telegram.clj

4 directories, 5 files
```

- `src/passport_status.clj` contains all the logic related to passport status fetching
- `src/telegram.clj` contains all the logic to interact with `telegram`

## Scraping Information
To find out how to extract passport application information, I manually go through the process.

### Figuring out the API
* Go to the [Passport Website FrontPage](https://passportindia.gov.in)
* Close the annoying popup.
* Oh, I see a `Track Application Status` button on the left.

<img style="align-self: center;" src="./assets/clj-automation-1/passport-main-page.png" alt="Last goodbye to Dadaji" height="300" width="300" />

* When I click on it, they ask to provide `File Number` and `Date of Birth`, and type of service.
* When I fill that info and click `Track Status`. I notice the following request being sent in the browser's network tab (Copied in the curl format):


```sh
curl 'https://passportindia.gov.in/AppOnlineProject/statusTracker/trackStatusInpNew' \
  -H 'Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7' \
  -H 'Accept-Language: en-US,en;q=0.9,hi-IN;q=0.8,hi;q=0.7' \
  -H 'Cache-Control: max-age=0' \
  -H 'Connection: keep-alive' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -H 'Cookie: JSESSIONID=<*******************>' \
  -H 'DNT: 1' \
  -H 'Origin: https://passportindia.gov.in' \
  -H 'Referer: https://passportindia.gov.in/AppOnlineProject/statusTracker/trackStatusInpNew' \
  -H 'Sec-Fetch-Dest: document' \
  -H 'Sec-Fetch-Mode: navigate' \
  -H 'Sec-Fetch-Site: same-origin' \
  -H 'Sec-Fetch-User: ?1' \
  -H 'Upgrade-Insecure-Requests: 1' \
  -H 'sec-ch-ua-mobile: ?0' \
  --data-raw 'apptUrl=&apptRedirectFlag=&optStatus=Application_Status&fileNo=<********>&applDob=<DD/MM/YYY>&rtiRefNo=&diplArnNo=&appealNo=&appealDob=&action%3AtrackStatusForFileNoNew=Track+Status'

```

Ahah, can I just send it over again and get the same response? Hell yeah. When I execute the above command on terminal, I get an HTTP response. (Thank God, for no Captcha)

Perfect. Now, let's also try without all those unnecessary headers and data in request body. Turns out we just need to send the request body:
```json
{
  "optStatus": "Application_Status",
  "fileNo": "ABC**********",
  "applDob": "DD/MM/YYYY",
  "action:trackStatusForFileNoNew": "Track Status"
}
```

That really simplifies the problem.

### Parsing Information
Let's start by fetching the response. This code goes in `src/passport_status.clj`.
```clojure
(ns passport-status
  (:require [clj-http.client :as http]))

;; Data for each passport application
(def details
  [{:file-num "ABC234324234"
    :dob "15/09/1992"}
   {:file-num "ABC234324234"
    :dob "24/11/1987"}
   {:file-num "ABC234324234"
    :dob "10/01/1971"}
   {:file-num "ABC234324234"
    :dob "19/02/2001"}])

;; URL to fetch passport application status
(def ^:private url
  "https://www.passportindia.gov.in/AppOnlineProject/statusTracker/trackStatusInpNew")

;; Function that sends a POST request to the above URL
;; And fetches the actual HTML response
(defn- fetch-resp [{:keys [file-num dob]}]
  (http/post url {:form-params {:optStatus "Application_Status"
                                :fileNo file-num
                                :applDob dob
                                :action:trackStatusForFileNoNew "Track Status"}}))
```
The above code, should help me fetch the response from the server. Now it's time to parse the response and extract the structured information.

I am inspecting the web page source in the browser's `developer tools`. The information is in the following structure:
```html
<div class="hd_blue">
<table>
  <tbody>
    <tr>
      <td>
        <table>
          <tbody>
            <tr>
              <td> <-- This is what we need
            <tr>
              <td> <-- This is what we need
            <tr>
              <td> <-- This is what we need
```

Hickory makes it super easy to select those elements with it's `hickory.select` functions.

```clojure
(ns passport-status
  (:require [clj-http.client :as http]
            [hickory.core :as h]
            [hickory.select :as s]))

;; Omitted code

;; Selecting required data from hickory format
(defn- select-data [hickory]
  (->> hickory
       (s/select (s/follow-adjacent (s/class "hd_blue") (s/tag :table))) ; selects element with class `hd_blue` and moves to the adjacent `table` element.
       first                                                             ; Gets the first such element
       (s/select (s/child (s/tag :table)                                 ; Traverses the heirarchy shown in the previous code snippet
                          (s/tag :tbody)
                          (s/tag :tr)
                          (s/tag :td)
                          (s/tag :table)
                          (s/tag :tbody)
                          (s/tag :tr)
                          (s/tag :td)))
       (mapcat (fn [td]                                                  ; From here on, we are just extracting the text from the `td`s (Unintended)
                 (:content td)))
       (partition-all 2)
       (take 7)))
```

## Telegram bot setup
Next step is to setup a bot for telegram, that will send up messages with data extracted in the previous steps. I chose `telegram` over `whatsapp` because of the simplicity of using its APIs.

### Creating the bot and channel
I need a bot that authenticates me to use telegram APIs and a channel where the bot will send me messages. I can have the bot send messages to me personally, but I'll have it send to a channel, so that all the members of the channel can see the updates.

Telegram has a special bot that creates other bots. It's rightly named the `@BotFather`. Sending a `/help` message lists all the available commands to interact with it. Most important for us is the `/newbot` command. This command triggers a flow to create new bot and asks a few questions related to it.

* Name of the new bot
* A globally unique username that ends with `bot`

Once the bot is created, it sends a welcome message with the bot token and a link to chat with the bot. Bot token will be used to authenticate requests to the telegram API.

I'll also create a private channel and add the bot to it as an admin.

### Sending message
Now, that I have the bot token and created the channel. I need the `channel_id` to send the message to the channel. It's hard to find it on the telegram app or website. The way we'll do it is by using [getUpdates](https://core.telegram.org/bots/api#getupdates) API endpoint. This endpoint returns the data that bot has received using the long polling method. The data contains the `channel_id` as well.

First step is to send a message in the channel I created, for the bot to receive it as update. So, I send a `hi` message in the telegram channel.
To get the updates, I'll use the repl:
```clojure
(require '[clj-http.client :as http])
(require '[cheshire.core :as json])

(def bot-token "<the-bot-token-from-previous-steps>")

(-> (str "https://api.telegram.org/bot" bot-token "/getUpdates")
    (http/get {:headers {:content-type "application/json"}})
    :body
    (json/parse-string))

; {"ok" true,
;  "result"
;  [{"update_id" 242976779,
;    "channel_post"
;    {"message_id" 8,
;     "sender_chat"
;     {"id" <chat-id>, "title" "Bot Devel", "type" "channel"},
;     "chat"
;     {"id" <chat-id>, "title" "Bot Devel", "type" "channel"},
;     "date" 1731041338,
;     "text" "hi"}}]}
```
The response is shown above. I get the `<chat-id>` from there, and save it for later use. Test sending the message by:
```clojure
(def chat-id 1234434) ; Replace this with <chat-id>

(-> (str "https://api.telegram.org/bot" bot-token "/sendMessage")
    (http/post {:headers {:content-type "application/json"}
                :body (json/encode {:chat_id chat-id
                                    :text "Hello"})}))
```
I get a `Hello` text on the telegram channel. Ready to assemble the `telegram.clj` file.

```clojure
(ns telegram
  (:require [clj-http.client :as http]
            [cheshire.core :as json]
            [config :as cfg]))

(def ^:private bot-url
  (str "https://api.telegram.org/bot" (-> cfg/telegram :bot-token)))

(defn- telegram-url [method]
  (case method
    :send-message (str bot-url "/sendMessage")
    :get-updates (str bot-url "/getUpdates")))

(defn- channel-id [channel]
  (or (get-in cfg/telegram [:channel-ids channel])
      (when (= cfg/env :dev)
        (get-in cfg/telegram [:channel-ids :default]))))

(defn send-message! [channel message]
  (http/post (telegram-url :send-message)
             {:headers {:content-type "application/json"}
              :body (json/encode {:chat_id (channel-id channel)
                                  :text message
                                  :parse_mode "HTML"})}))
```
`config` namespace comes from `env/dev/config.clj` file. The structure looks something like the following:
```clojure
(ns config)

(def telegram
  {:bot-token "<bot-token>"
   :channel-ids {:default <chat-id>}})

(def env :dev)
```
`env/prod/config.clj` will have the same structure with prod related config.

The final `passport-status` namespace looks like the following, with some missing pieces and them integrated.
```clojure

(ns passport-status
  (:require
   [clj-http.client :as http]
   [hickory.core :as h]
   [hickory.select :as s]
   [clojure.string :as str]
   [telegram :as tg]))

(def details
  [{:file-num "ABC234324234"
    :dob "15/09/1992"}
   {:file-num "ABC234324234"
    :dob "24/11/1987"}
   {:file-num "ABC234324234"
    :dob "10/01/1971"}
   {:file-num "ABC234324234"
    :dob "19/02/2001"}])

(def ^:private url
  "https://www.passportindia.gov.in/AppOnlineProject/statusTracker/trackStatusInpNew")

(defn- fetch-resp [{:keys [file-num dob]}]
  (http/post url {:form-params {:optStatus "Application_Status"
                                :fileNo file-num
                                :applDob dob
                                :action:trackStatusForFileNoNew "Track Status"}}))

(defn- select-data [hickory]
  (->> hickory
       (s/select (s/follow-adjacent (s/class "hd_blue") (s/tag :table)))
       first
       (s/select (s/child (s/tag :table)
                          (s/tag :tbody)
                          (s/tag :tr)
                          (s/tag :td)
                          (s/tag :table)
                          (s/tag :tbody)
                          (s/tag :tr)
                          (s/tag :td)))
       (mapcat (fn [td]
                 (:content td)))
       (partition-all 2)
       (take 7)))

(defn- format-message [data]
  (->> data
       (map (fn [[key val]]
              (format "<b>%s</b>: %s" (str/trim key) (str/trim val))))
       (str/join "\r\n")
       (str "<b><u>" (java.time.LocalDate/now)
            " " (-> (java.time.LocalTime/now)
                    str
                    (str/split #"\.")
                    first)
            "</u></b> \r\n")))

(defn ^:export execute! [& _]
  (->> details
       (pmap (fn [info]
               (-> (fetch-resp info)
                   (:body)
                   (h/parse)
                   (h/as-hickory))))
       (pmap select-data)
       (pmap format-message)
       (pmap #(tg/send-message! :infinite %))
       (doall)))
```
Understanding the above code is left as an exercise to the reader. From the `passport-status` ns, I just need to call the `telegram/send-message!` with the channel and [formatted message data](https://core.telegram.org/bots/api#formatting-options).

## Deployment
Now, I need a way to setup a job that runs every 3 hours or so. I'll use an EC2 instance on AWS that I already use to host my personal projects.

First, I copy the project to the EC2 machine. Then, I use systemd timers for job scheduling because of its simple syntax and configuration.

I create two files for systemd config.
* `passportstatus.service`: keeps the config about what commands to run that will execute the `passport-status/execute!` function.
* `passportstatus.timer`: keeps the timer configuration, when to run the job and how often.

#### passportstatus.timer 
```systemd
[Unit]
Description="Passport Fetch status"

[Service]
WorkingDirectory=/home/user/automation
ExecStart=/bin/bash -c "/usr/local/bin/clojure -X:prod:passport"
```

#### passportstatus.timer
```systemd
[Unit]
Description="Run a job to fetch passport status."

[Timer]
OnStartupSec=5seconds
OnBootSec=5min
OnUnitActiveSec=24h
OnCalendar=*-*-* 09/3:00:00 # Runs every three hours starting at 9 AM.
Unit=passportstatus.service

[Install]
WantedBy=multi-user.target
```

Create softlinks to these files in `/etc/systemd/system` directory. And enable the `passportstatus.timer` unit by:
```sh
$ sudo systemctl enable passportstatus.timer
```

## Conclusion
It was a fun little project, that would have taken a few days for me to finish, just a few months back. This exercise boosted my confidence in executing a project from scratch in clojure and getting to deployment. This is just the start of more automations, that'll come in the future.

Thanks for reading and stay tuned for more.
