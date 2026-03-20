# Language-tool setup
This documents shows how to self-host language-tool and add it to different browsers/editors for what is effectively supposed to be a free, open-sourced, self-hosted grammarly replacement.

Assumption(s): Docker is installed, debian/ubuntu based OS.

## Language-tool
The [original language-tool repo](https://github.com/languagetool-org/languagetool) setup is not very straightforward. Luckily, [user-maintained dockerized version(s)](https://github.com/meyayl/docker-languagetool?tab=readme-ov-file) exists (and are also listed in the official repo.).

Choosing one of these dockerized version, we clone it:
```bash
git@github.com:meyayl/docker-languagetool.git
```
Note that this also comes with ngrams (context corrections like "breaks" vs "brakes") and fasttext (something with multi-language, idk, didn't test this).

Save the origina `docker-compose.yml` and create a new one:
```bash
mv docker-compose.yml docker-compose.yml.old
touch docker-compose.yml
```

Put your desired docker setup into the new `docker-compose.yml` file. The [dockerized language-tool repo](https://github.com/meyayl/docker-languagetool?tab=readme-ov-file) provides many options.  Here, we go with:
```YAML
---
services:
  languagetool:
    image: meyay/languagetool:latest
    container_name: languagetool
    restart: unless-stopped
    read_only: true
    tmpfs:
      - /tmp:exec
    cap_drop:
      - ALL
    cap_add:
      - CAP_CHOWN
      - CAP_DAC_OVERRIDE
      - CAP_SETUID
      - CAP_SETGID
    security_opt:
      - no-new-privileges
    ports:
      - 8081:8081
    environment:
      download_ngrams_for_langs: en
      MAP_UID: 783
      MAP_GID: 783
    volumes:
      - ./ngrams:/ngrams
      - ./fasttext:/fasttext
```

Compose and run the container:
```bash
sudo docker compose up -d
```

## Firefox
Assuming you did the first step, you setup language-tool as a Firefox extension by installing the following extension: [language-tool for firefox](https://addons.mozilla.org/en-US/firefox/addon/languagetool/).
After clicking through all the "confirm", "got it", "blah-blah" prompts. Click the settings wheel in the extension pop-up ... or get in the (advanced) settings any other way. Click on "Advanced settings (only for professional users)" and click on "Other server" in the "LanguageTool server" option. Here, you will want to enter `http://localhost:8081/v2.`, where the exact port depends on the port in your `docker-compose.yml`. Press save and profit.

Alternatively, [this website]( https://thecrow.uk/languagetool-is-an-ai-powered-grammar-checker-you-can-self-host-on-your-own-hardware/) contains a decent, yet slightly outdated, setup guide also.

## vscode
Install "LTeX – LanguageTool grammar/spell checking" by "Julian Valentin".
Go to settings (for this extension).
Put `http://localhost:8081/v2` into the "Ltex: Language Tool Http Server Uri" setting.
Reload the vscode window... that's it.

## neovim
NOTE: This setup was done using lazyvim, but if you are using some "fancier" version of nvim, you can probably figure it out from here.


Add the following extension setup:
```lua
return {
  {
    "neovim/nvim-lspconfig",
    opts = {
      servers = {
        ltex = {
          settings = {
            ltex = {
              languageToolHttpServerUri = "http://localhost:8081/v2",
              language = "en-GB",
              additionalRules = {
                enablePickyRules = true,
              },
            },
          },
        },
      },
    },
  },
}
```
into the `nvim ~/.config/nvim/lua/plugins/languagetool.lua` file. Make sure that vimtex is installed (via lang.tex in "Lazy" for lazyvim).

Note: Replace localhost by the actual address in case you are not running the languagetool locally.

## Accessing from different machines on the same local network
Simply use: `http://ip-of-host-here:8081/v2`, i.e. use the IP of the hosting machine instead of localhost.

## More advanced setup
To apply changes make sure to:
```bash
sudo docker compose down
sudo docker rmi container-id  # optional, to delete the old container
sudo docker compose up -d
```


### ngrams
This image does not download it automatically, but we still want to use it of course. Download the EN (or whatever language) file:
```bash
mkdir ngrams && cd ngrams
wget https://languagetool.org/download/ngram-data/ngrams-en-20150817.zip
```

Unzip it and clean up:
```bash
unzip ngrams-en-20150817.zip
rm ngrams-en-20150817.zip
```

Note that you will need to update your file accordingly, with something like:
```YAML
  services:
    environment:
      - langtool_languageModel=/ngrams
    volumes:
      - ./ngrams:/ngrams:ro
```

### Custom rules
Create a `languagetool.cfg` file:
```bash
touch languagetool.cfg
```

And paste your rules setup, e.g.:
```
# General English
enabledRules.en=IN_ORDER_TO,DUE_TO_THE_FACT,AT_THIS_POINT_IN_TIME
enabledCategories.en=PLAIN_ENGLISH,REDUNDANCY,STYLE

# American English
enabledRules.en-US=IN_ORDER_TO,DUE_TO_THE_FACT,AT_THIS_POINT_IN_TIME
enabledCategories.en-US=PLAIN_ENGLISH,REDUNDANCY,STYLE

# British English (The Missing Link!)
enabledRules.en-GB=IN_ORDER_TO,DUE_TO_THE_FACT,AT_THIS_POINT_IN_TIME
enabledCategories.en-GB=PLAIN_ENGLISH,REDUNDANCY,STYLE
```

Update your `docker-compose.yml` to the following:
```YAML

---
services:
  languagetool:
    image: meyay/languagetool:latest
    container_name: languagetool
    restart: unless-stopped
    read_only: true
    tmpfs:
      - /tmp:exec
    cap_drop:
      - ALL
    cap_add:
      - CAP_CHOWN
      - CAP_DAC_OVERRIDE
      - CAP_SETUID
      - CAP_SETGID
    security_opt:
      - no-new-privileges
    ports:
      - 8081:8081
    environment:
      download_ngrams_for_langs: en
      MAP_UID: 783
      MAP_GID: 783
      langtool_rulesFile: "/config/languagetool.cfg"
    volumes:
      - ./ngrams:/ngrams
      - ./fasttext:/fasttext
      - ./languagetool.cfg:/config/languagetool.cfg:ro
```

