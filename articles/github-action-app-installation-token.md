# Obtaining a GitHub App Installation Token


**Author: Daniel Einars**

**Date Published: 16.01.2025**

**Date Edited: 16.01.2025**


## 1. Intro
I've recently been through the process of setting up renovate-bot as a GitHub App, which required a GitHub App Installation Token (it didn't really, but I wanted the commits created by the bot to be signed because that's a constraint on all my repositories). I've tried using the [action](https://github.com/actions/create-github-app-token) linked in renovate's [documentation](https://docs.renovatebot.com/modules/platform/github/#running-as-a-github-app) but kept on getting cryptic errors. The Github Docs provide a small bash script to generate the JWT token and a curl post request for creating the App Installation Token. This article just puts the two of them together for use in a GitHub Workflow step.

## 2. Getting the JWT for the App

In order to get App Installation Token from the GitHub API, you first need [generate a JWT for the app](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-a-json-web-token-jwt-for-a-github-app). Thankfully, the Github API docs [provide](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-a-json-web-token-jwt-for-a-github-app#example-using-bash-to-generate-a-jwt) a small bash script for this (copy & pasted below). The script takes two arguments.

1. The Client ID of the GitHub App
2. The path to the private key of the GitHub App (you'll get this when you create the GitHub App)
```shell
#!/usr/bin/env bash

set -o pipefail

client_id=$1 # Client ID as first argument

pem=$( cat $2 ) # file path of the private key as second argument

now=$(date +%s)
iat=$((${now} - 60)) # Issues 60 seconds in the past
exp=$((${now} + 600)) # Expires 10 minutes in the future

b64enc() { openssl base64 | tr -d '=' | tr '/+' '_-' | tr -d '\n'; }

header_json='{
    "typ":"JWT",
    "alg":"RS256"
}'
# Header encode
header=$( echo -n "${header_json}" | b64enc )

payload_json="{
    \"iat\":${iat},
    \"exp\":${exp},
    \"iss\":\"${client_id}\"
}"
# Payload encode
payload=$( echo -n "${payload_json}" | b64enc )

# Signature
header_payload="${header}"."${payload}"
signature=$(
    openssl dgst -sha256 -sign <(echo -n "${pem}") \
    <(echo -n "${header_payload}") | b64enc
)

# Create JWT
JWT="${header_payload}"."${signature}"
printf '%s\n' "JWT: $JWT"

```

Once you have this, you can proceed to the next step.

## 2. Getting the Installation Access Token

Once you have the Token, you can use it to create a installation access token using [this curl request](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/authenticating-as-a-github-app-installation#authenticating-with-an-installation-access-token) (copy & pasted below)

Replace `$JWT` with the JWT you generated in the previous step and `$INSTALLATION_ID` with the installation ID of the GitHub App you want to create a token for. You can find the installation ID in the URL of the GitHub App's page.

```shell
curl --request POST \
--url "https://api.github.com/app/installations/INSTALLATION_ID/access_tokens" \
--header "Accept: application/vnd.github+json" \
--header "Authorization: Bearer $JWT" \
--header "X-GitHub-Api-Version: 2022-11-28"
```

This will return a JSON response with the installation access token..

## 3. Putting it all together in a Github Action.

I needed the App Installation Token to run renovate-bot as a GitHub App. Below is the result of putting the two steps above together in a single step aptly named `Generate Access Token`. Feel free to ignore the rest, it isn't relevant to the article.

The Step takes inputs for the `CLIENT_ID` and `PRIVATE_KEY` secrets. The output is storing the App Installation Token in the `RENOVATE_TOKEN` environment variable.

```yaml
name: Renovate-Bot

on:
  # Allows manual/automated ad-hoc trigger
  push:
    branches:
      - master
  workflow_dispatch:
    inputs:
      logLevel:
        description: "Override default log level"
        required: false
        default: "info"
        type: string
      overrideSchedule:
        description: "Override all schedules"
        required: false
        default: "false"
        type: string
  schedule:
    - cron: "0 0 * * *"

permissions:
  packages: write
  contents: write
  pull-requests: write
  id-token: write

concurrency: renovate
env:
  PEM: |
    ${{ secrets.PRIVATE_KEY }}
jobs:
  renovate:
    name: renovate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Generate Access Token
        run: |
          #!/usr/bin/env bash
          set -o pipefail
          client_id="${{ secrets.CLIENT_ID }}" # Client ID as first argument
          pem="${{ secrets.PRIVATE_KEY }}" # PEM content as second argument (string directly) 
          now=$(date +%s)
          iat=$((${now} - 60)) # Issues 60 seconds in the past
          exp=$((${now} + 600)) # Expires 10 minutes in the future
          b64enc() { openssl base64 | tr -d '=' | tr '/+' '_-' | tr -d '\n'; }
          header_json='{
            "typ":"JWT",
            "alg":"RS256"
          }'
          # Header encode
          header=$( echo -n "${header_json}" | b64enc )
            
          payload_json="{
            \"iat\":${iat},
            \"exp\":${exp},
            \"iss\":\"${client_id}\"
          }"
          # Payload encode
          payload=$( echo -n "${payload_json}" | b64enc )
            
          # Signature
          header_payload="${header}"."${payload}"
          signature=$(
            openssl dgst -sha256 -sign <(echo -n "${pem}") \
            <(echo -n "${header_payload}") | b64enc
          )
            
          # Create JWT
          JWT="${header_payload}"."${signature}"
          echo "RENOVATE_TOKEN=$(curl --request POST \
          --url "https://api.github.com/app/installations/${{ secrets.INSTALLATION_ID }}/access_tokens" \
          --header "Accept: application/vnd.github+json" \
          --header "Authorization: Bearer $JWT" \
          --header "X-GitHub-Api-Version: 2022-11-28" | jq -r '.token')" >> "$GITHUB_ENV"
      - uses: pnpm/action-setup@v4
        with:
          version: 9.15.2
      - uses: actions/setup-node@v4
        with:
          node-version: "22.9.0"
      - name: renovate
        uses: renovatebot/github-action@v41.0.8
        env:
          RENOVATE_CONFIG_FILE: ./renovate-config.js
          RENOVATE_FORCE: ${{ github.event.inputs.overrideSchedule == 'true' && '{''schedule'':null}' || '' }}
          LOG_LEVEL: ${{ inputs.logLevel || 'info' }}




```