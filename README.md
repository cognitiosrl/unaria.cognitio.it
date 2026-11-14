# Unaria install repository

This repository is one install of Unaria, for one company. It is private, and it holds:

- `unaria-umbrella/`: the umbrella repository as a git submodule, pinned at a release, a commit on the umbrella's `main` carrying exactly one tag `vX.Y.Z`. The umbrella carries its modules as submodules of its own.
- `.gitmodules`: the submodule's declaration; the deploy reads the umbrella's repository from it.
- `wrangler.jsonc`: the install's Worker configuration, with its reviewed values committed in it and the placeholders the deploy fills left as they are.
- `.github/workflows/deploy.yml`: the deploy workflow.
- `.gitignore`: the files a deploy or a local run writes and this repository never tracks.

The umbrella's `docs/INSTALL.md` (here `unaria-umbrella/docs/INSTALL.md`) is the full guide: creating this repository, the read-only token, filling `wrangler.jsonc`, the repository variables and secrets, the Cloudflare API token, the Google sign-in client, the sending domain, the first deploy, the `/health` checks, upgrades, rollback and restore. This repository was created from the umbrella's `install-template/`; a change to that template reaches this repository only when copied in by hand.

## Setting it up

1. **The read-only token.** A fine-grained personal access token: resource owner `cognitiosrl`, only the repositories `unaria-umbrella` and `unaria-hr`, Contents: Read-only and nothing else. Set it on this repository and on the umbrella, whose CI reads `modules/hr` with it; each command asks for the value:

   ```sh
   gh secret set SUBMODULES_TOKEN --repo <install repository>
   gh secret set SUBMODULES_TOKEN --repo cognitiosrl/unaria-umbrella
   ```

2. **`wrangler.jsonc`.** Replace once by hand, and commit, the placeholders of the install's reviewed values: `REPLACE_AUTH_PROVIDERS`, `REPLACE_BOOTSTRAP_OWNER_EMAIL`, `REPLACE_ALLOWED_EMAIL_DOMAINS`, `REPLACE_ALLOWED_EMAILS`, `REPLACE_MODULES`, `REPLACE_DEFAULT_TIME_ZONE`, `REPLACE_DEFAULT_LOCALE`, `REPLACE_MAIL_FROM`, `REPLACE_ALERT_EMAIL_TO`, and the rate limiters' `REPLACE_RATE_LIMIT_AUTH_NAMESPACE_ID`, `REPLACE_RATE_LIMIT_WRITE_NAMESPACE_ID` and `REPLACE_RATE_LIMIT_UPLOAD_NAMESPACE_ID`. Leave the others: the deploy fills `WORKER_NAME`, `HOSTNAME` and `GOOGLE_CLIENT_ID` from the repository variables, `RELEASE` from the release, and `D1_DATABASE_NAME`, `D1_DATABASE_ID`, `R2_BUCKET_NAME`, `JOBS_QUEUE_NAME` and `JOBS_DEAD_LETTER_QUEUE_NAME` from the resources it creates.

3. **The five repository variables, then the secrets**, which `gh secret set` asks for. There is no variables file:

   ```sh
   R=<install repository>
   gh variable set CLOUDFLARE_ACCOUNT_ID --repo "$R" --body "<account ID>"
   gh variable set WORKER_NAME --repo "$R" --body "<worker name>"
   gh variable set HOSTNAME --repo "$R" --body "<install hostname>"
   gh variable set GOOGLE_CLIENT_ID --repo "$R" --body "<client ID>"
   gh variable set TOFU_STATE_BUCKET --repo "$R" --body "<state bucket>"
   gh secret set CLOUDFLARE_API_TOKEN --repo "$R"
   gh secret set TOFU_STATE_PASSPHRASE --repo "$R"
   gh secret set GOOGLE_CLIENT_SECRET --repo "$R"
   ```

   The optional ones, only if you use them:

   ```sh
   gh secret set SPARKPOST_API_KEY --repo "$R"
   gh secret set CHAT_WEBHOOK_URL --repo "$R"
   ```

   Push notifications need a key pair: run `pnpm vapid:keys` once in the umbrella, then store its two values:

   ```sh
   gh secret set VAPID_PUBLIC_KEY --repo "$R"
   gh secret set VAPID_PRIVATE_KEY --repo "$R"
   ```

   `TOFU_STATE_BUCKET` may name a bucket shared with other products or installs: this install's state key in it is `unaria/<install hostname>/terraform.tfstate`.

4. **Plan, then deploy:**

   ```sh
   gh workflow run deploy.yml --repo "$R" --ref main -f plan_only=true
   gh run watch --repo "$R"
   gh workflow run deploy.yml --repo "$R" --ref main
   ```

## Deploying

A push to `main`, or Actions → deploy → Run workflow on `main`, deploys the release `unaria-umbrella/` is pinned at; a run started on another branch is skipped, and one deploy runs at a time. A run started by hand with `plan_only` shows OpenTofu's plan and stops there. The workflow:

1. checks out this repository without submodules;
2. reads the umbrella's repository from `.gitmodules` and its commit from the `unaria-umbrella` pointer, and checks out the umbrella at that commit with the read-only `SUBMODULES_TOKEN`, with its modules at the umbrella's pointers, one level deep and nothing beyond;
3. refuses the pin unless it is a release (`check-release-tag`); its tag is the release deployed;
4. creates or updates the install's D1 database, R2 bucket, queue and dead-letter queue, in the EU jurisdiction, and the generated `HEALTH_TOKEN`, with OpenTofu from the umbrella's `iac/install/` (`infra`), stopping on a plan that deletes or replaces a resource;
5. fills in `wrangler.jsonc` what the repository variables, the release and the resources give (`fill-config`), writes the secrets file, and checks both against the umbrella's configuration schema and cron registry and the release's `install-template/wrangler.jsonc` (`check-config`);
6. builds the app, applies the D1 migrations, and deploys, uploading the secrets with the version;
7. accepts the deploy only when the uploaded version alone serves all traffic and `/health`, asked with `HEALTH_TOKEN`, reports the release and the latest migration (`verify-deploy`).

Those tools live in the umbrella's `scripts` package and run from the pinned umbrella, so they always match the code they deploy.

## Upgrading

One commit moving the pointer, then its deploy. In a clone made with `git clone --recurse-submodules`:

```sh
git -C unaria-umbrella fetch --tags origin
git -C unaria-umbrella checkout vX.Y.Z
git add unaria-umbrella
git commit -m "chore: deploy vX.Y.Z"
git push
```

When the configuration check refuses the install after an upgrade, compare `wrangler.jsonc` with `unaria-umbrella/install-template/wrangler.jsonc` at the new release and bring its bindings, variables and cron schedule to the template's, keeping your reviewed values.

## Updating an action

The workflow's actions are pinned to commit SHAs. To update one, resolve the new tag with `git ls-remote https://github.com/<owner>/<action> 'refs/tags/<tag>' 'refs/tags/<tag>^{}'` (the peeled `^{}` line, when present, is the commit) and keep the tag in the comment.
