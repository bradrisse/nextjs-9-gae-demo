# nextjs-9-gae-demo

Demonstration of [Next.js](https://nextjs.org) 9.x on [Google App Engine's Standard Environment for Node.js](https://cloud.google.com/appengine/docs/standard/nodejs/).

### Prerequisites

1. Install the [Google Cloud Platform SDK](https://cloud.google.com/sdk/).
2. Clone this repo.
3. Install Node.js (use homebrew, apt, whatever you choose).
4. Either run commands locally to build and deploy the app, or create a Cloud Build trigger to use the cloudbuild.yaml file.

### Install dependencies

```sh
npm install
```

Use if dependencies were not already specified in package.json.

```sh
npm install next react react-dom
```

### Test the App

Start the app to make sure it runs properly.

```sh
npm run dev
```

### Configuring a New GCP Project and Application

If you have not set up a project and application to run this demo, follow the steps below. Otherwise, skip to the next section.

First, authenticate to GCP if you have not already:

```sh
gcloud auth login
```

Using the GCP CLI, create a new project and application. Replace `PROJECT-NAME` with your own.

```sh
gcloud projects create PROJECT-NAME
gcloud config set project PROJECT-NAME
gcloud app create
```

Wait a few minutes while Google provisions resources. While you are waiting, enable the Cloud Build API for your project by visiting the Cloud Build API page for your project (ex: https://console.developers.google.com/apis/api/cloudbuild.googleapis.com/overview?project=PROJECT-NAME).

### Building and Deploying the Application (using local gcloud CLI)

```sh
npm run build
npm run deploy
```

### Building and Deploying the Application (using Cloud Build)

I need to add gcloud commands for this, but for now:
  1. Fork this repository into your own Github repo (or some other repo used by GCP), so you can authorize Cloud Build to use it.
  2. Create a Cloud Build trigger to point to this repo, and use the cloudbuild.yaml file.
  3. Just use a manual trigger for now and kick it off when ready.

Visit the link provided by GCP to test.

### Explanations

**next.config.js**

```
module.exports = {
  distDir: 'build',
}
```

- Google App Engine doesn't recognize .next folder so you need to change the folder from .next to build using the distDir parameter in the next.config.js

**app.yaml**

```
env: standard
runtime: nodejs12
service: default

handlers:
  - url: /.*
    secure: always
    script: auto
```

- The newest version of nextjs has dependencies that require nodejs12, so setting the runtime to nodejs12 fixes that issue
- the /.\* ensures access to all folders and sub folders within the build folder
- secure always ensures https is used
- script auto tells GAE to rely on nextjs to indicate what files to load instead of index.js

**package.json -> scripts**

`"start": "next start -p 8080"`

- GAE runs on port 8080 and uses the start script by default in the package.json file

**package.json -> scripts**

```
"build": "rm -rf ./build && NODE_ENV=production next build",
"start": "next start -p 8080",
"deploy": "npm run build && gcloud app deploy"
```

- the build script command removes the existing build folder and then runs a new build, this prevents nextjs making multiple versions and running into issues uploading many unnecessary previous builds during deploy

- the deploy script command runs the build script and the app deploy command
- if your fearless `"deploy": "npm run build && gcloud app deploy --quiet"` use the quiet tag so you don't have to type "y" to confirm deploy

**.gcloudignore**

You should only be upload the build folder and public folder to GAE. To prevent uploading source code you should add these lines to the .gcloudignore files. `/pages` is just the standard folder for basic next projects, if you have more like `/components/`, or `/store/` you should add those too.

```
# Next.js source code
/pages/
```

**Dynamic Catch All Route**

If you are using the dynamic catch all route `blog/[...slug].js` then you might get this error

`ERROR: (gcloud.app.deploy) INVALID_ARGUMENT: Filename cannot contain '.', '..', '\r', start with '-', '_ah/', or '\n': blog/[...slug]/index.js`

Basically GAE doesn't like when files names have `...` in them. To fix this your package.json scripts should now be

```
"gcp-predeploy": "find ./build -name '\\[...*' -exec bash -c 'mv \"$1\" \"${1/.../@@@}\"' -- {} \\;",
"gcp-build": "find ./build -name '\\[@@@*' -exec bash -c 'mv \"$1\" \"${1/@@@/...}\"' -- {} \\;",
"build": "rm -rf ./build && NODE_ENV=production next build",
"start": "next start -p 8080",
"deploy": "npm run build && npm run gcp-predeploy && gcloud app deploy"
```

So now the deploy script runs build which removes the existing build folder, builds a new one, then replaces the dynamic all route file names from [...slug].js to [@@@slug].js, deploys the app, and once the app is uploaded GAE runs gcp-build and renames them back to [...slug].js

This fix was taken from here and modified [https://github.com/vercel/next.js/issues/10556#issuecomment-636313552](https://github.com/vercel/next.js/issues/10556#issuecomment-636313552)
