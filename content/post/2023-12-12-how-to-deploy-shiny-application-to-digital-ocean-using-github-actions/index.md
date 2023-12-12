---
title: How to deploy Shiny application to Digital Ocean using GitHub Actions
author: 'teo'
date: '2023-12-12'
slug: how-to-deploy-shiny-application-to-digital-ocean-using-github-actions
categories: [R, Shiny, GitHub Actions, Digital Ocean, ShinyProxy]
tags: []
subtitle: ''
summary: 'A walkthrough on setting up GitHub Actions for automatic deployment 
of Shiny application to DigitalOcean server running ShinyProxy'
authors: [teo]
lastmod: '2023-12-12T12:01:07-06:00'
featured: yes
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---

## ShinyProxy on DigitalOcean

For the remainder of this walkthrough, I'll assume that a DigitalOcean droplet
running ShinyProxy is already set up. The great people at [Analythium](https://analythium.io/) have made setting up an encrypted ShinyProxy server seamless with their 1-click (https://marketplace.digitalocean.com/apps/shinyproxy) application solution. Please follow their tutorials to set this up.

Having said that, this is in no way limited to the Analythium 1-click solution.
You can set up a new DigitalOcean server running [ShinyProxy from scratch](https://www.shinyproxy.io/documentation/getting-started/), or adapt the below protocol to different cloud service provider.

## GitHub Actions and DigitalOcean setup

In this section we'll set up SSH access on Digital Ocean servers for GH actions:

1. Set up a user for GHActions on the server

We can use `root` to login to the remote server, but its better for GH Actions
to run things without root priveledges. So, login to the droplet as user:

```
ssh root@ip.addr.of.droplet
```

and run

```
useradd ghactions
```

This will create our user and should not prompt for password. The only way to
access the server with this user is through `ssh keys` (see below).

Because the user will most likely require to pull docker images, its also good
to add it to the docker user group. This will bypass the requirement for sudo
when running docker commands:

```
sudo usermod -aG docker ghactions
newgrp docker
```

2. Create ssh keys for the `ghactions` user on your local computer

On your local machine, we want to create a _private-public ssh key pair_ for
our ghactions user. We _don’t want to use the personal ssh keys_. To do this,
we generate the keys in a temporary location:

```
ssh-keygen -C ghactions -f /tmp/ghactions-keys
```

3. Upload the public key to the server

- log in as root: `ssh root@ip.addr.of.droplet`

- create `.ssh` folder for user `ghactions`: `mkdir /home/ghactions/.ssh`

- create `.ssh/authorized_keys` file for user `ghactions`: `touch /home/ghactions/.ssh/authorized_keys`

- change ownership for the ssh config files: `chown ghactions:ghactions -R /home/ghactions/.ssh`

- change permissions for the files: `chmod 700 -R /home/ghactions/.ssh`

- copy paste the public key from your local computer (`/tmp/ghactions-keys.pub`)
  to the `/home/ghactions/.ssh/authorized_keys` file

- disconnect from the server (`exit` or `ctrl+d`)

- try connecting as the ghactions user: `ssh -i /tmp/ghactions-keys ghactions@ip.addr.of.droplet`

- if you can log in, the ssh setup should be good to go

- test that you can run `docker` without `sudo`: `docker run hello-world`

4. Store the private key to the GitHub repository secret

This requires _admin_ priveledges on the github repo.

Go to `Settings -> Secrets and Variables -> Actions` and click `New repository secret`.
Paste the entire contents of your temporary private key, file `/tmp/ghactions.keys`
(including the first and last line

`-----BEGIN OPENSSH PRIVATE KEY-----`

and

`-----END OPENSSH PRIVATE KEY-----`
).

Call the secret `SSH_PRIVATE_KEY`.

5. Test your github action. A simple action to verify that the GitHub Action
   runner can access the droplet could be as follows:

```
  on:
    push:
      branches: - dev

  jobs:
    test_gh_to_do_ssh:
      name: test gh to do ssh connection
      runs-on: ubuntu-latest

      steps:
        - name: Create a dummy file on server
          uses: appleboy/ssh-action@v1.0.0
          with:
            host: ip.addr.of.droplet
            username: ghactions
            key: ${{ secrets.SSH_PRIVATE_KEY }}
            script: touch github-actions-made-this-file
```

If you got no errors, login to the server as user ghactions and check if the
file is there. Then delete it.

6. Delete your local copy of the _ssh keys for ghactions user_

If everything went well, you can delete the temporary ssh files you created for
the `ghactions` user: `rm /tmp/ghactions-keys /tmp/ghactions-keys.pub`.

The private key is safe in github secrets. For added security you could periodically
change update the key.

## Docker container registry on DigitalOcean

One of the components of the deployment workflow is that we have to host our `docker`
images on a remote repository and pull them to the DigitalOcean server. To do this,
one can use any Docker registry, including [Docker hub](hub.docker.com), [GitHub's docker registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-docker-registry). For this post, I'll use [DigitalOcean's container
registry solution](https://www.digitalocean.com/products/container-registry).

To set this up we simply create a registry for our account or team. Then, to access
the registry, we need to generate an access token. For example, by following this
[help page](https://docs.digitalocean.com/reference/api/create-personal-access-token/)

Finally, we need provide this access token as a repository secret to GitHub Actions,
because it is needed for the runners to be able to push and pull images. To do this,
go `Settings -> Secrets and Variables -> Actions` and click `New repository secret`.
Paste the access token string and name the secret `DIGITALOCEAN_ACCESS_TOKEN`.

We are now ready to use the token in our workflow.

## GH Actions workflow for deployment

The workflow below is fairly straighforward. Mostly calling `docker` to
login, push and pull images. The `appleboy/ssh-action@v1.0.0` is used to
access the DigitalOcean droplet via ssh following our earlier setup of the
`ghactions` user.

In sequence, the steps are:

1. The workflow will run on push to dev. For example after merging a PR for example
2. Checkout the dev branch
3. Build a docker image out of it using the Dockerfile included in the repo.
   For this I used `golem::add_dockerfile_shinyproxy()`
4. Push the docker image to the DO registry under a tag `latest`. One can also use
   a SHA string to tag the image specifically
5. `ssh` to the droplet using USERNAME@HOST with key `SSH_PRIVATE_KEY` (as we set it up earlier),
   The user name and host are hard-coded here, but if necessary they can be set as
   secrets or variables.
6. Pull the image using the same tag (the DigitalOcean access token is passed as
   an environmental variable so the runner has access to the container registry after `ssh`-ing)
7. Next time you log in the app ShinyProxy will automatically serve the latest `docker` image

```
on:
  push:
    branches:
      - dev
jobs:
  deploy_to_dev:
    name: deploy to dev
    runs-on: ubuntu-latest

    steps:
      - name: Checkout dev
        uses: actions/checkout@v4
        with:
          ref: dev

      - name: Build container image
        run: docker build -t registry.digitalocean.com/myregistry/testgolem:latest .

      - name: Log in to DigitalOcean Container Registry with short-lived credentials
        run: echo ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }} | docker login registry.digitalocean.com/myregistry -u $(echo ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}) --password-stdin

      - name: Push image to DigitalOcean Container Registry
        run: docker push registry.digitalocean.com/myregistry/testgolem:latest

      - name: Pull image on DigitalOcean ShinyProxy Server
        uses: appleboy/ssh-action@v1.0.0
        env:
          DO_REGISTRY_TOKEN: ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}
        with:
          host: ip.addr.do.droplet
          username: ghactions
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          envs: DO_REGISTRY_TOKEN
          script: |
            echo $DO_REGISTRY_TOKEN | docker login registry.digitalocean.com/myregistry -u $(echo $DO_REGISTRY_TOKEN) --password-stdin
            docker pull registry.digitalocean.com/myregistry/testgolem:latest
```

## Gist

For quick access to the main files visit this gist https://gist.github.com/teofiln/d52241797dfc055ed9b9bc96f0c0cb70

## Summary

This article includes a step-by-step tutorial on setting up automatic deployment
of a Shiny application to a DigitalOcean server running ShinyProxy via GitHub Actions.
