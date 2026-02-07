# Commitizen Github Action

The goal if this repo is to create a github workflow, that whenever a PR is
merged, creates a release with commitizen. All commits have to conform to the
conventional commit specification.

To try it out, create a pull request with a title that conforms to the conventional commit specification. After merging, commitizen will to the magic. See also [.github/workflows/release.yml](.github/workflows/release.yml) for more details.

## 🤖 Automated Release Setup

This repository uses **Conventional Commits** and a custom **GitHub App** to automate versioning and releases. To prevent humans from accidentally pushing to `main` while allowing the Bot to do its job, we use GitHub **Rulesets**.

## 🛠️ How to Create the Release Bot (GitHub App)

A GitHub App is preferred over a Personal Access Token (PAT) because it is more secure, doesn't expire, and can be easily exempted from branch protection rules.

### 1. App Creation

1. Navigate to your **Personal Settings** (Profile icon > Settings).
2. On the left sidebar, click **Developer settings** > **GitHub Apps** > **New GitHub App**.
3. **App Name:** `Gierlinger-Infra-Bot` (or your preferred name).
4. **Homepage URL:** Your GitHub profile (e.g., `https://github.com/fgierlinger`).
5. **Webhook:** Uncheck **Active** (Not needed for this workflow).
6. **Permissions:**
    * **Repository permissions > Contents:** Select **Read & write**.
    * **Repository permissions > Metadata:** Select **Read-only** (Automatically selected).

7. **Where can this app be installed?** Select **Only on this account**.
8. Click **Create GitHub App**.

### 2. Authentication Setup

1. **App ID:** Note the "App ID" displayed on the General settings page. You will need this for your GitHub Secret.
2. **Private Key:** Scroll down to the "Private keys" section and click **Generate a private key**. A `.pem` file will download to your computer.
3. **Install the App:**
    * On the left sidebar of the App settings, click **Install App**.
    * Click **Install** next to your account.
    * Select **Only select repositories** and choose your `ansible-gierlinger.xyz` repo.
    * Click **Install**.

### 3. Repository Secrets

Go to your repository **Settings > Secrets and variables > Actions** and add the following two secrets:

* `RELEASE_APP_ID`: Paste the App ID from Step 2.1.
* `RELEASE_APP_PRIVATE_KEY`: Open the downloaded `.pem` file in a text editor and paste the **entire content** (including the `-----BEGIN RSA PRIVATE KEY-----` lines).

---

## 🔄 Final `release.yml` with App Logic

This is the production-ready workflow that uses the secrets above to bypass your Ruleset protections.

```yaml
name: Release

on:
  push:
    branches:
      - main

jobs:
  bump-version:
    # Ensures the bot doesn't trigger itself
    if: "!contains(github.event.head_commit.message, 'bump:')"
    runs-on: ubuntu-latest
    steps:
      - name: Generate Token
        uses: actions/create-github-app-token@v1
        id: app-token
        with:
          app-id: ${{ secrets.RELEASE_APP_ID }}
          private-key: ${{ secrets.RELEASE_APP_PRIVATE_KEY }}

      - name: Checkout Code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ steps.app-token.outputs.token }}

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install Commitizen
        run: pip install -U commitizen

      - name: Create bump and changelog
        run: |
          # Use the official bot email so the avatar appears in Git
          git config --local user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git config --local user.name "github-actions[bot]"
          cz bump --yes --changelog

      - name: Push changes
        run: |
          git push origin main --tags

```

### 1. The Ruleset Configuration

Go to **Settings > Rules > Rulesets** and create/edit a "Branch Ruleset" for `main`:

* **Bypass List:** * Add your GitHub App (e.g., `Gierlinger-Infra-Bot`).
* Set **Bypass Mode** to **Always**. This allows the bot to skip PR requirements and push directly.

* **Rules:**
* **Restrict updates:** **Enabled**. (This is the Ruleset version of "Restrict Pushes"). Only actors in the Bypass List can push directly.
* **Restrict deletions:** **Enabled**.
* **Require a pull request before merging:** **Enabled**. This ensures humans must use PRs.
* **Block force pushes:** **Enabled**.

### 2. Workflow Logic (`release.yml`)

The release workflow is designed to handle the "Bypass" logic and prevent infinite loops.

```yaml
# .github/workflows/release.yml
jobs:
  release:
    # 1. Prevent the bot from triggering its own release workflow
    if: github.actor != 'Gierlinger-Infra-Bot[bot]'
    runs-on: ubuntu-latest
    steps:
      # 2. Generate a short-lived token from the GitHub App
      - name: Generate Token
        id: generate_token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.BOT_APP_ID }}
          private-key: ${{ secrets.BOT_APP_PRIVATE_KEY }}

      # 3. Checkout using the App Token
      # This allows subsequent 'git push' commands to inherit the Bot's bypass permissions
      - name: Checkout
        uses: actions/checkout@v4
        with:
          token: ${{ steps.generate_token.outputs.token }}
          fetch-depth: 0

      # 4. Bump version and Push
      - name: Release
        run: |
          git config user.name "Gierlinger-Infra-Bot[bot]"
          git config user.email "Gierlinger-Infra-Bot[bot]@users.noreply.github.com"
          cz bump --yes
          git push origin main --tags

```

> **Note:** We use `Restrict updates` instead of `Restrict creations` because the `main` branch already exists. `Restrict updates` specifically targets pushes to an existing branch.

## Author

Frédéric Gierlinger

## License

See [LICENSE](./LICENSE)
