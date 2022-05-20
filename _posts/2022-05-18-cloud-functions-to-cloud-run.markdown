---
layout: post
title:  "Migrate Cloud Functions to Cloud Run"
date:   2022-05-18 12:45:10 +0800
categories: gcp cloudfunctions cloudrun
---

We'll use the [Functions Framework](https://cloud.google.com/functions/docs/functions-framework) and Functions Framework [buildpacks](https://github.com/GoogleCloudPlatform/buildpacks#functions-framework-buildpacks) to migrate code written for Cloud Functions to Cloud Run without making any code changes.

## Objectives

* Introduce the Functions Framework
* Test code written for Cloud Functions locally using the Functions Framework
* Deploy code to Cloud Functions
* Migrate Cloud Functions code to Cloud Run using the Functions Framework buildpack

## Using Cloud Shell

This tutorial uses the following tool packages:

* [`gcloud`](https://cloud.google.com/sdk/gcloud)
* [`pip3`](https://pip.pypa.io/en/stable/)
* [`pack`](https://buildpacks.io/docs/tools/pack/)
* [`docker`](https://www.docker.com/)

Because [Cloud Shell](https://cloud.google.com/shell) automatically includes these packages, you run the commands in this tutorial in Cloud Shell, so that you don't need to install these packages locally.

## Implementation steps

### Introduce the Functions Framework

The Functions Framework lets you write lightweight functions that run in many different environments, such as:

* Your local development machine
* Cloud Functions
* Cloud Run

Using the Functions Framework, you can write portable code that works across these environments without making additional code changes.

### Set the environment variables

1.  Set the required environment variables:

        PROJECT_ID=${GOOGLE_CLOUD_PROJECT}
        REGION=${REGION}
        FUNCTION_NAME=${FUNCTION_NAME}

### Enable the required APIs

1.  Enable the required APIs:

        gcloud services enable \
          cloudfunctions.googleapis.com \
          cloudbuild.googleapis.com \
          run.googleapis.com

### Isolate project dependencies

1. Create a project directory and move into it:

        mkdir my_project && cd my_project

1. Use the `venv` command to create a virtual copy of the entire Python installation. This tutorial creates a virtual copy in a folder named `venv`, but you can specify any name for the folder:

        python3 -m venv .venv

1. Set your shell to use the `venv` paths for Python by activating the virtual environment:

        source .venv/bin/activate

### Create your sample application

1. Create your application files:

        cat  << EOF > main.py
        def hello(request):
          return "Hello world!"
        EOF

        cat << EOF > requirements.txt
        functions-framework==2.2.1
        EOF

### Install the Functions Framework

1. Install the Functions Framework via `pip`:

        pip3 install -r requirements.txt

### Test your function locally with the Functions Framework

The Functions Framework makes it easy to test your code locally without needing to deploy it to Cloud Functions beforehand.
1. In your current terminal, start your function locally using the Functions Framework:

        functions-framework --target hello --debug

1. In another terminal, send a request to your local function:

        curl localhost:8080

1. You should get back:

        Hello world!

   Stop your function in the previous terminal by entering `Ctrl-c`.

### Deploy your function to Cloud Functions

1. Deploy the function on your local machine to Cloud Functions:

        gcloud functions deploy ${FUNCTION_NAME} \
          --entry-point hello \
          --region ${REGION} \
          --runtime python39 \
          --trigger-http \
          --allow-unauthenticated

1. Test your deployed function:

        FUNCTION_URL="$(gcloud functions describe $FUNCTION_NAME --region ${REGION} | awk '/url:/ {print $2}')"
        curl "${FUNCTION_URL}"

1. You should get back:

        Hello world!

### Understand what happens when a Cloud Function is deployed

When you deploy a Cloud Function, [two main things](https://cloud.google.com/functions/docs/running/overview#abstraction_layers) happen:

1. Cloud Functions uses the Functions Framework to unmarshal incoming HTTP requests into language-specific function invocations. This is what allows you to test your function as a locally-runnable HTTP service, and is what you have done in the previous steps.

1. Cloud Native buildpacks are then used to wrap the HTTP service created by the Function Framework and build them into runnable Docker containers, which then run on Cloud Functions' container-based architecture. You will learn how to do this using Cloud Native buildpacks next.

### Build a container image of your Cloud Functions code

1. Use the Functions Framework buildpack to build a container image of your Cloud Functions code:

        pack build \
          --builder gcr.io/buildpacks/builder:v1 \
          --env GOOGLE_FUNCTION_SIGNATURE_TYPE=http \
          --env GOOGLE_FUNCTION_TARGET=hello \
          ${FUNCTION_NAME}

### Test your container image locally

1. Test your Cloud Functions container image locally:

        docker run --rm -p 8080:8080 ${FUNCTION_NAME}

1. In another terminal, send a request to your local function:

        curl localhost:8080

1. You should get back:

        Hello world!

   Stop your function in the previous terminal by entering `Ctrl-c`.

### Publish your container image

1. Publish the build image to the cloud directly with `pack`:

        pack build \
          --builder gcr.io/buildpacks/builder:v1 \
          --publish gcr.io/${PROJECT_ID}/${FUNCTION_NAME} \
          --env GOOGLE_FUNCTION_SIGNATURE_TYPE=http \
          --env GOOGLE_FUNCTION_TARGET=hello

### Deploy container to Cloud Run

1. Deploy your Cloud Function to Cloud Run:

        gcloud run deploy ${FUNCTION_NAME} \
          --image gcr.io/${PROJECT_ID}/${FUNCTION_NAME} \
          --platform managed \
          --region ${REGION} \
          --allow-unauthenticated

1. Visit your service:

        SERVICE_URL="$(gcloud run services describe $FUNCTION_NAME --region ${REGION} | awk '/URL:/ {print $2}')"
        curl "${SERVICE_URL}"

1. You should get back:

        Hello world!