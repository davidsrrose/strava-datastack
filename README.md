# strava-datastack
<img src="https://github.com/user-attachments/assets/91f696f8-81a7-4352-b5e9-71215f9c91ad" alt="CompatibleWithStrava" style="height: 25px;" />

### Table of Contents
- [Overview](#Overview)
- [Setup](#setup)
  - [Strava Auth](#strava-auth)
- [Usage](##use)
  - [`dlt`](#dlt)

## Overview
WIP project to visualize strava data using open source tooling

🏃🚴 🤝 📊

🧰 Tools planned:
- Orchestration → [dagster](https://github.com/dagster-io/dagster)
- Warehouse → [DuckDB](https://github.com/duckdb/duckdb)
- Extract & Load → [dlt](https://github.com/dlt-hub/dlt)
- Transform → [dbt](https://github.com/dbt-labs/dbt-core)
- BI → [evidence](https://github.com/evidence-dev/evidence)

👋 Find me on &nbsp;<a href="https://www.strava.com/athletes/4901617"><img src="https://upload.wikimedia.org/wikipedia/commons/c/cb/Strava_Logo.svg" alt="Strava" style="height: 1em; max-height: 1.2em"/></a>

## Setup
Install required dependencies:
```bash
pip install -r requirements.txt
```
### Strava Auth
#### Create your own [Strava API Application](https://www.strava.com/settings/api)
This is necessary to generate the credentials (e.g. `client_id`, `client_secret`, etc.) you will supply to the strava resource in `strava.py`. Don't worry, creating an application is just a few clicks.

(*full details on the process of creating an app are supplied by strava [here](https://developers.strava.com/docs/getting-started/#account)*)

#### Review App
Once your app has been created, it will come with a refresh token and an access token. For the purposes of this setup (i.e. creating an automated `dlt` pipeline to export your strava data), these are utterly useless.

<div align="center">
<img width="514" alt="image" src="https://github.com/user-attachments/assets/00db977f-ecb3-4216-bdaf-7cef2b15c632">
</div><br>

- The initial Access Token expires after 6 hours and will need updating (this is how strava manages access tokens)
- The intial Refresh Token is minimally scoped with `read` access which does not allow viewing of activity data (there is a seperate `activity:read` scope that must be granted to a token to view activity data)

To get around these issues, we will create a new refresh token with the proper scopes that we can pass `dlt`.

`dlt` can then use this refresh token that has already been authorized to fetch an up-to-date access token whenever it runs.

#### Generating new refresh token with proper scopes:
(*these steps were ripped from the [Strava developer docs](https://developers.strava.com/docs/getting-started/#oauth)*)
1. Paste the Client ID from your app into this URL where `[REPLACE_WITH_YOUR_CLIENT_ID]` is, and specify the required scopes where `[INSERT_SCOPES]` is:
    ```text
    https://www.strava.com/oauth/authorize?client_id=[REPLACE_WITH_YOUR_CLIENT_ID]&response_type=code&redirect_uri=http://localhost/exchange_token&approval_prompt=force&scope=[INSERT_SCOPES]
    ```
    *note: in order to get full read access to both your public and private data, these are the necessary [scopes]([scopes](https://developers.strava.com/docs/authentication/#detailsaboutrequestingaccess))*
    ```text
    read_all,activity:read_all,profile:read_all
    ```

2. Paste the updated URL into a browser, hit enter, and follow authorization prompts.

3. After authorizaiton, you should get a "This site can't be reached" error. Copy the authorization code (i.e. the `code=` value) from the returned URL.

<div align="center">
<img width="417" alt="image" src="https://github.com/user-attachments/assets/82f33361-a87f-42c9-aff9-750145fead6b">
</div><br>

4. Run a cURL request to generate the refresh token that `dlt` will use:
    ```bash
    curl -X POST https://www.strava.com/oauth/token \
      -F client_id=YOURCLIENTID \
      -F client_secret=YOURCLIENTSECRET \
      -F code=AUTHORIZATIONCODE \
      -F grant_type=authorization_code
    ```
5. If successful, the response will return a json payload with a `refresh_token` (save this)

#### `secrets.toml` setup
`dlt` offers various methods for [storing connection credentials](https://dlthub.com/docs/general-usage/credentials/);

shown below is an example `secrets.toml` for the strava connection secrets:
```toml
[sources.strava.credentials]
access_token_url = "https://www.strava.com/oath/token"
client_id = "<your_client_id>"
client_secret = "<your_client_secret>"
refresh_token = "<your_refresh_token>"
```

## Usage
### `dlt`
With crendentials defined, the `strava.py` dlt pipeline can be run via:
```python
python strava.py
```