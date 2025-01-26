---
title: "The Cyber Society's deployment or how to over engineer your selfhosted personal blog"
description: "The description of my blog deployement using Hugo, Github Actions, Docker and much more"
---

As the first article of my personal blog—and this blog being one of my many personal cyber projects—it feels fitting (and a bit meta) that my first post is about deploying this blog! What started as a simple blog idea quickly turned into a journey of over-engineering—let me walk you through how I turned it into a learning experience.

# The context and my requirements

## Personal context
Since I started to have some cyber personal projects, I came accross countless personal blogs to get some knowledge or inspiration. They come in multiple forms and flavors, each reflecting something unique about their authors. I often find myself wondering: have I ever encountered the authors away from keyboard. Personal blogs are such a cool part of the internet, and I’ve always wanted to create one. I told myself that I will have mine one day and today is the day!
I wanted to build something personal, technical, and cool. I see personal projects as opportunities to explore and learn new technologies while integrating them with other projects. For this blog, I decided to host it on my homelab—which is one of my biggest personal projects to date. It consists of a 42U rack with 2 pfSense routers in high availability, a 3-node Proxmox cluster, and a 10G backbone network in my basement (stay tuned for articles about it).

## Technical requirements
With that in mind, here are the technical goals I set for this project:
* Host the site on my homelab.
* Use GitHub to host and versionning the files.
* Automate the deployment, I want a one-click deployment to publish my articles - When my article is ready, I want to push it to GitHub and watch my site update itself.
* Use a proxy to protect my personnal IP.
* Use docker compose to deploy my site as this is my go-to for deploying services on my homelab.
* Have SSL set up
* Being transparent to the reader, make the entire process visible to readers so they can explore and learn how everything works under the hood.
* Use Markdown to write my articles.
* Use a flat file architecture to facilitate any future migration.
* A low maintenance set up, in general I prefer to spend a little more time building a solution that will be low maintenance than the other way around.

# The architecture
The solution I developed is a static site powered by Hugo, hosted and versioned on GitHub. I use GitHub Actions to build the site and publish it as a container in GitHub's container repository. My existing Docker Compose stack on the homelab pulls the updated container via WatchTower. The reverse proxy and SSL certificates are managed by Traefik, while Cloudflare acts as a proxy to shield my personal IP.

The architecture is divided into four main parts:

* Site TechnologyThe site is built with Hugo, an open-source static site generator written in Go. It allows me to host and build the site using GitHub, write articles in Markdown, and maintain a flat file architecture.
* CI/CD SetupContinuous Integration and Continuous Deployment (CI/CD) are handled by GitHub Actions, which automate the site’s build process and publish the container to GitHub’s container repository.
* HostingThe hosting is managed on my homelab, using Docker Compose to deploy the site. WatchTower ensures the container is updated automatically whenever there’s a new release.
* Proxy SetupTraefik manages the reverse proxy and SSL certificates, while Cloudflare provides an additional layer of security and acts as a proxy for my public-facing site. 

# The deployements (aka hands on)
