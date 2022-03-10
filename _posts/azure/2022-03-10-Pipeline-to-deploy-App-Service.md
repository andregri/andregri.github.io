---
layout: single
title: Pipeline to deploy Azure App Service
tags: azure github
---

For the [Microsoft Azure Trial Hackathon on DEV](https://dev.to/devteam/hack-the-microsoft-azure-trial-on-dev-2ne5)
I created a web application to basically perform CRUD operations on a Azure SQL Database. The source code is hosted on
[GitHub](https://github.com/andregri/AiFame) and I decided to use GitHub Actions to create a CICD pipeline.

The repository is based on two branches:
- `main` where to push code going to production
- `dev` where to push code to be tested but not deployed

The **continuous integration** workflow tries to install the Python requirements and runs both on `main` and `dev` branches. 
I have not written any test at the moment.
```
# This is a basic workflow to help you get started with Actions

name: ci

# Controls when the workflow will run
on:
  # Triggers the workflow on push or pull request events but only for the main branch
  push:
    branches: [ main, dev ]
  pull_request:
    branches: [ main, dev ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@v2

      - name: Setup Python
        uses: actions/setup-python@v2.3.1
        with:
          # Version range or exact version of a Python version to use, using SemVer's version range syntax.
          python-version: 3.9
          # The target architecture (x86, x64) of the Python interpreter.
          architecture: x64
          
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
```

The **continuous deployment** workflow build and push the Docker image to Azure Container Registry.
It runs only on the `main` branch.
```
on: [push]
name: cd

jobs:
    build-and-deploy:
        runs-on: ubuntu-latest
        steps:
        # checkout the repo
        - name: 'Checkout GitHub Action'
          uses: actions/checkout@main
          
        - name: 'Login via Azure CLI'
          uses: azure/login@v1
          with:
            creds: ${{ secrets.AZURE_CREDENTIALS }}
        
        - name: 'Build and push image'
          uses: azure/docker-login@v1
          with:
            login-server: ${{ secrets.REGISTRY_LOGIN_SERVER }}
            username: ${{ secrets.REGISTRY_USERNAME }}
            password: ${{ secrets.REGISTRY_PASSWORD }}
        - run: |
            docker build . -t ${{ secrets.REGISTRY_LOGIN_SERVER }}/aifame --file Dockerfile.prod
            docker push ${{ secrets.REGISTRY_LOGIN_SERVER }}/aifame
```

The Docker image is not tagged explicitly so that Docker assigns the default tag, *latest*.
I didn't follow any semantic versioning syntax for the images.
Rolling back to a previous version would be a painful operation.
However, the *latest* tag is used by default by Azure App Service to deploy the app.

To configure the Azure App Service I needed to set the following options:
- Registry: the container registry name
- Image: the image name
- Tag: the image tag (default is *latest*)
- Continuous Deployment: On

The overall diagram of the cicd pipeline is:
[![EhcwCJ.md.png](https://iili.io/EhcwCJ.md.png)](https://freeimage.host/i/EhcwCJ)