# Quasar Electron App Template

A minimal cross-platform desktop app built with [Electron](https://www.electronjs.org/) via the [Quasar Framework](https://quasar.dev/).
Includes a basic CI/CD pipeline via GitHub Actions and GitHub Releases, as well as automatic updates via [electron-updater](https://yarn.pm/electron-updater).

## Tutorial

To reproduce this repository from scratch for yourself, you can follow the instructions below.

### Prerequisites

Install the latest LTS version of [Node.js](https://nodejs.org/en/).
I also recommend installing [Yarn](https://yarnpkg.com/) for package management.

```bash
npm install -g yarn
```

Then, install the Quasar CLI.

```bash
yarn global add @quasar/cli
# or
npm install -g @quasar/cli
```

### Create a New Quasar Project

For this template app, all default options were selected when creating the project.

```bash
quasar create quasar-electron-template
cd quasar-electron-template
```

If you want to initialize Git and commit the project at this point, make sure to also specify the remote repository, which is required for Quasar to build the app.

```bash
git init
git add -A
git commit -m "Initial Quasar project"
git branch -M master # if the default branch is not master
git remote add origin git@github.com:edward-ly/quasar-electron-template.git # required
git push -u origin master
git checkout -b develop # create a separate branch for development
```

### Edit Quasar Configuration

Open `quasar.conf.js` and make the following changes to the `electron` options object:

-   Replace `'packager'` with `'builder'` as the value of the `bundler` property (since we are using `electron-builder` to build the app).
-   In the configuration object for the `builder` property, add the following lines:

```js
win: {
  target: [
    {
      target: 'nsis',
      arch: ['x64', 'arm64', 'ia32']
    }
  ]
},
mac: {
  target: [
    {
      target: 'dmg',
      arch: ['x64', 'arm64']
    }
  ]
},
linux: {
  target: [
    {
      target: 'AppImage',
      arch: ['x64', 'arm64', 'ia32', 'armv7l']
    }
  ]
},
publish: {
  provider: 'github'
}
```

Now create a test build for the app.
This will also enable Electron mode for the project, adding Electron-specific dependencies and source code.

```bash
quasar build -m electron
```

### Add Automatic Updates

Install `electron-updater` as an app dependency.

```bash
yarn add electron-updater
```

Open `src-electron/electron-main.js` and make the following changes:

```js
// Add the following line near the top of the file
import { autoUpdater } from 'electron-updater'

// Replace this line ...
app.on('ready', createWindow)
// ... with this code
app.on('ready', () => {
  autoUpdater.checkForUpdatesAndNotify()
  createWindow()
})
```

### Configure GitHub Actions

#### From Quasar

Add the following scripts to `package.json`:

```json
"build": "quasar build -m electron -P never",
"release": "quasar build -m electron -P onTagOrDraft",
```

Create a new file in `.github/workflows/main.yml` with the following code:

```yml
name: build

on:
  push:
    branches:
      - master
      - develop
  pull_request:
    branches:
      - develop

jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os:
          - windows-latest
          - macos-latest
          - ubuntu-latest
      max-parallel: 1
      fail-fast: false
    steps:
      - name: Clone repository
        uses: actions/checkout@v2
      - name: Setup Node
        uses: actions/setup-node@v2
        with:
          node-version: 14.x # use the latest LTS version of Node
      - name: Install dependencies
        run: yarn --frozen-lockfile
      - name: Build app
        if: github.ref != 'refs/heads/master'
        run: yarn build
      - name: Build & deploy app
        if: github.ref == 'refs/heads/master'
        env:
          GH_TOKEN: ${{ secrets.GH_TOKEN }}
        run: yarn release
```

#### From GitHub

1.  From your GitHub account settings, go to **Developer settings > Personal access tokens** and click **Generate new token**.
2.  Give the new token a name, but more importantly, check the `repo` box to enable repository access for the token.
    Then click **Generate token** and copy the value of the new token.
3.  From the project repository settings, go to **Secrets** and click **New repository secret**.
    Type `GH_TOKEN` into the name field, and paste the value of the personal access token into the value field, then click **Add secret**.

## Continuous Workflow

After the above steps are taken, GitHub Actions will automatically build the app for all platforms each time you push new commits to the repository (or a pull request targeting the `develop` branch is created).
If new commits are pushed to the `master` branch, GitHub Actions will also upload the artifacts to GitHub Releases where the app will be ready for publishing.

#### Recommended Steps

The following steps are taken from [the default `electron-builder` workflow](https://www.electron.build/configuration/publish#recommended-github-releases-workflow), but modified to accommodate Git branching development models such as [this one](https://nvie.com/posts/a-successful-git-branching-model/).

1.  [Draft a new release](https://help.github.com/articles/creating-releases/).
    Set the "Tag version" to some version after the current `version` in your application `package.json`, and prefix it with `v`.
    Make sure the tag targets the `master` branch.
    "Release title" can be anything you want.
2.  Push some commits to the `develop` branch.
    Each successful CI build confirms that the app can be compiled on all platforms.
3.  Create a release branch, add some commits that will prepare the app for release (e.g. increasing the `version` in `package.json`), then merge the release branch into `master`.
4.  Push the new commits to the `master` branch.
    Confirm that the artifacts from this CI build have been uploaded to the release draft.
4.  Add a description (preferably release notes) to the release, and publish the release.
5.  Merge the release branch back into `develop`, and delete the release branch.
