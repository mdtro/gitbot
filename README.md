# Sentry Deploy Sync Hook

This repo contains a hook for updating the reference in getsentry to sentry automatically. There's two ways that this happens.

If a push happens on Sentry's master, this will clone getsentry and call `bin/bump-sentry` in order to update
the Sentry's sha on getsentry.

If a PR is opened/synchronized on Sentry and `#sync-getsentry` appears in the first message of the PR, the bot will try to bump the version on getsentry for a branch with the same name as the one on Sentry. This keeps both PRs synchronized and is useful for staging deployments. More details [here](https://www.notion.so/sentry/sync-getsentry-95a32dabe03b467bb3ec5fa0e20491e5).

## Deployment

Production deployment: `https://sentry-deploy-sync-hook-dwunkkvj6a-uc.a.run.app`
Staging deployment: `https://sentry-deploy-sync-hook-staging-dwunkkvj6a-uc.a.run.app`

The CI uploads the Docker image and deploys it once changes are merged on `master`.

If you want to make deployments to production or a PR follow the next steps.

### Configure Google Cloud locally

- [Install gcloud](https://cloud.google.com/sdk/docs/install)
- Install the Docker GCR extension with `gcloud components install docker-credential-gcr`
- Authenticate with `gcloud auth login`
- Configure Docker to authenticate with GCloud `gcloud auth configure-docker`

### Deploy manually via workflow

Once you're ready to deploy to production, visit the [Deploy workflow](https://github.com/getsentry/sentry-deploy-sync-hook/actions/workflows/deploy.yml) and dispatch it with the value `production`.

If you want to deploy a PR, you can follow the same process but choose the branch associated to that PR. It only allows to deploy to the staging service.

### GCR configuration

The GCR instances have these environments defined:

- DEPLOY_REPO:
  - `getsentry/getsentry` for production
  - `getsentry/getsentry-test-repo` for staging
- ENV: staging or production
- Images deployed from `gcr.io/sentry-dev-tooling/sentry-deploy-sync-hook`

The GCR instances can fetch the authentication token and Github webhook secret from Google Secrets without any env variables since they have a service account associated to the service.

## Repositories set up and testing

By default, the development set up will push changes to [getsentry-test-repo](https://github.com/getsentry/getsentry-test-repo) (which is only available to the productivity team members).

To test against another repo you can use the env variable `DEPLOY_REPO` in order to point to a different repo you have write access to. In order for this to work, such repo needs `bin/bump-sentry` and `cloudbuild.yaml` from the getsentry repo.

## Testing changes

Testing pushes:

- Create a branch named `test-branch` on Sentry (or your fork)
  - If using a fork, make sure you set the upstream of your branch to your fork
- From here on, pushes to that branch (or `master`) will be processed by the backend
- You can use this command: `echo "$(date)" > file && git add file && git commit -m "Foo" --no-verify && git push` to trigger a push
  - Check the output of the backend to see if it succeeds

Testing PR syncs:

- On your sentry repo and the `getsentry-test-repo`(or a repo you define with `DEPLOY_REPO`) create a branch named `test-pr` (name it anything but `test-branch`)
- Push both branches to your Sentry fork and your getsentry test repo
- On Sentry (or your fork), open a PR with the word `#sync-getsentry`
  - Any subsequent pushes to that Sentry branch will trigger a bump on the `DEPLOY_REPO`

Testing that it can fetch Google Secrets:

- Download a key associated to the GC staging service account
  - Place the file in your source checkout as `gcr-key.json` (it needs to be within the mount)
- Run `docker-compose run -e GOOGLE_APPLICATION_CREDENTIALS="gcr-key.json" backend`

Check if the production set up starts up (GCR logs can sometimes fail to show the issue):

```shell
docker run \
  -e GOOGLE_APPLICATION_CREDENTIALS=gcr-key.json \
  -v `pwd`:/app --rm -ti sentry-deploy-sync-hook
```

## Rotate secret

Steps:

- This [Notion page](https://www.notion.so/sentry/Bot-Accounts-beea0fc35473453ab50e05e6e4d1d02d) has information as to who has access to the bot account.
- Request a new personal access token
- Visit [Google Secrets](https://console.cloud.google.com/security/secret-manager?project=sentry-dev-tooling) and add a new version of the secret
- Update the version of the secret in the code, commit to `master`, build and deploy the app
- Once you see things working you can request for the original one to be deleted
- In Google Secrets you can disable or destroy the previous version of the secret

## Requirements

- Docker

## Development

Create [a new personal access token](https://github.com/settings/tokens/new) for this project with `repo` access and run the commands below.

```shell
echo DEPLOY_SYNC_PAT=<value> > .env
echo DEPLOY_SYNC_USER=<your_github_user> >> .env
```

**NOTE**: Docker Compose reads by default variables defined in that file. This will will _not_ be included as part of the Docker image or your Github history.

**NOTE**: It is super important you understand that this token will be able to commit anywhere the associated Github user can. It is encouraged you delete this token from Github (or disk) as soon as you're done doing development.

We use docker compose to help with live code reloading (since we mount a volume to the source checkout):

```shell
docker-compose up --build
```

To test the push API you can use `curl`:

```shell
curl \
    --header "Content-Type: application/json" \
    --header 'X-GitHub-Event: push' \
    --request POST \
    --data '{"ref":"refs/heads/master","repository":{"full_name":"getsentry/sentry"},"head_commit":{"id":"438cb62a559889b5ae68ce3494c1034c60e50f4a","author":{"name":"wmak","email":"william@wmak.io"}}}' \
    http://0.0.0.0:5000
```

To test the Github PR API you can type this:

```shell
curl \
    --header "Content-Type: application/json" \
    --header 'X-GitHub-Event: pull_request' \
    --request POST \
    --data '{}' \
    http://0.0.0.0:5000
```

### Running the pipeline locally

**NOTE**: The development set up will not commit code unless you set `DRY_RUN` env to False.

**NOTE**: We assume you have the backend running on your localhost (see steps on section above).

In order to test Github changes through your local set up you need to follow these steps:

- Run `docker-compose up --build -e DRY_RUN=False`
  - Verify the output says dry run mode to be off and which repo it will push to
- Set up [Ngrok](https://ngrok.io/) to redirect Github calls to your localhost
  - `ngrok http 5000` --> Grab the URL ngrok gives you (e.g. `https://6a88fe29c5cc.ngrok.io`)
- Fork sentry and create a webhook under the repo's settings
  - Point it to the URL that ngrok gives you
  - Choose `application/json` for `Content type`
  - For a production set-up you will want to define a [Secret](https://docs.github.com/en/developers/webhooks-and-events/creating-webhooks#secret)
    - TODO: Verify we have code that checks `X-Hub-Signature` and `X-Hub-Signature-256`
  - Choose `Let me select individual events` and select: `Pull requests` and `Pushes`

**NOTE**: You can inspect the contents of Github webhook events in the sample place where you edit the webhook. You can re-deliver and see the contents of the response.

**NOTE**: You can test that your pipeline works by following the steps in the section "Testing changes"