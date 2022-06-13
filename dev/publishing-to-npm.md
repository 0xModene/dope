# Publishing a Package to NPM

## Requirements

These will be needed before you can get started. Generate/grab these and copy them down.

- `NPM Personal Access Token`: a PAT from NPM. Can be found/created at `https://www.npmjs.com/settings/[USERNAME]/tokens`
- `Github PAT`: a PAT from Github. Can be found/created [here](https://github.com/settings/tokens).

## Creating the PATs

Go to your repo's page on Github, then navigate to `Settings > Secrets > Actions`

![repo secrets](https://user-images.githubusercontent.com/7647623/173412025-7b3f2452-e3b4-4513-a183-7d3a76374302.png)


Put your NPM PAT from earlier into `NPM_TOKEN`. (This can be done at the `Organization secrets` level, which is useful if you want to publish multiple packages). Do the same for your Github PAT and `GH_TOKEN`.

## Installing Tooling

There are some good packages required to make this tidy and easy to publish. You'll want to install those by running:

```bash
npm i cz-conventional-changelog semantic-release
```

These will make publishing the package easier, and the commit messages more standardized and readable.

## Updating Scripts

These tools will change how you commit code, so you'll want to update the commands in `package.json` to make things easier to remember for yourself. Add the below to the `scripts` object:

```json
"commit": "git-cz",
"semantic-release": "semantic-release --branches main"
```

`commit` will force you to use `npm run commit` when committing code, but will use the commitizen tool to do so. This standardizes commit messages for publishing.

`semantic-release` uses the semantic-release tool to actually publish to NPM. This will be used by the Github Workflow that you'll see later.

## Updating config settings

For metadata reasons, you may want to update your `package.json` with this information, although it's not strictly necessary.

```json
  "homepage": "https://github.com/[ORG_OR_USER]/[REPO]", // can be any homepage
  "repository": {
    "type": "git",
    "url": "https://github.com/[ORG_OR_USER]/[REPO]" // links to the repo on NPM
  },
  "bugs": {
    "url": "https://github.com/[ORG_OR_USER]/[REPO]/issues" // gives users of the package a place to log bugs
  }
```

These additions, however, _are_ required

```json
"config": {
  "commitizen": {
    "path": "./node_modules/cz-conventional-changelog"
  }
},
"publishConfig": {
  "access": "public", // required to be public, unless you are paying for NPM private packages
  "registry": "https://registry.npmjs.org/"
}
```

One final important change to `package.json` will be the package name. If you created, for instance, a repo named `tokenlist` and you want to publish it to NPM under `@indexcoop/tokenlist`, you'll need to adjust the package name in `package.json` to compensate

```json
// before
"name": "tokenlist", // tokenlist being the repo name and default value
// after
"name": "@indexcoop/tokenlist" // what you wish the package name to be
```

This is what will set up the repo to be downloaded with NPM like `npm i @indexcoop/tokenlist`

## Creating the Github Workflow

In your repo, you'll want to add a `publish` workflow for Github. In the repo's root, create `.github/workflows/publish.yml`

Inside `publish.yml`, you'll want something akin to the following:

```yml
# This workflow will do a clean install of node dependencies, cache/restore them, build the source code and run tests across different versions of node
# For more information see: https://help.github.com/actions/language-and-framework-guides/using-nodejs-with-github-actions

name: publish

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [12.x, 14.x, 16.x]
        # See supported Node.js release schedule at https://nodejs.org/en/about/releases/

    steps:
      - uses: actions/checkout@v2
      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v2
        with:
          node-version: ${{ matrix.node-version }}
          cache: "npm"
      - run: npm ci
      - run: npm run build --if-present
      - run: npm test

  publish:
    runs-on: ubuntu-latest
    if: ${{ github.ref == 'refs/heads/main' }}
    needs: [build]
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v1
        with:
          node-version: "16"
      - run: npm ci
      - run: npm test
      - name: Release
        env:
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
          GH_TOKEN: ${{ secrets.GH_TOKEN }}
        run: npx semantic-release --branches main
```

You can find out more about Github Workflows [here](https://docs.github.com/en/actions/using-workflows). You may want to adjust the build commands to whatever makes sense for your project. The above code assumes this is a straightforward javascript package.

## Publishing

If you performed all of the above, run the following:

```bash
git add . && npm run commit && git push
```

This should push everything up to Github, and automatically kick off the `publish.yml` workflow. You can watch the progress at `https://github.com/[ORG_OR_USER]/[REPO]/actions`, where you'll see logs that will help you debug any issues in deploying. If all is successful, you'll get green checkmarks across the board and will be able to find your package at `https://www.npmjs.com/package/[@ORG/PACKAGE NAME]`

NOTE: The `publish` workflow **_only_** triggers when merging to the `main` branch, but it will trigger on **_all_** merges to `main` that used the commitizen command `npm run commit`.
