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

* **Site Technology**, The site is built with Hugo, an open-source static site generator written in Go. It allows me to host and build the site using GitHub, write articles in Markdown, and maintain a flat file architecture.
* **CI/CD Setup**, Continuous Integration and Continuous Deployment (CI/CD) are handled by GitHub Actions, which automate the site’s build process and publish the container to GitHub’s container repository.
* **Hosting**, The hosting is managed on my homelab, using Docker Compose to deploy the site. WatchTower ensures the container is updated automatically whenever there’s a new release.
* **Proxy Setup**, Traefik manages the reverse proxy and SSL certificates, while Cloudflare provides an additional layer of security and acts as a proxy for my public-facing site. 

# The deployements (aka hands on)

With the architecture defined, it’s time to dive into the deployment process. Here’s a step-by-step guide to how I set up the blog on my homelab. You can find everything in my [TheCyberSociety github repo](https://github.com/vincentbenzo/TheCyberSociety).  

## 1. Setting Up the Hugo Site
   You first need to create a hugo site locally, then add the theme you want and to finish upload this site on Github.
   1. Install Hugo on your local machine by following the official documentation.
   2. Initialize a new Hugo site:
   ```bash
   hugo new site my-blog
   ```
   3. Add the PaperMod Theme:
   Navigate to your site’s directory and add the PaperMod theme as a submodule:
   ```bash
   cd my-blog
   git init
   git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
   ```
   4. Update Your `config.toml`:
   Set the theme to PaperMod and configure it to your preferences. Example:
   ```toml
   baseURL = "https://yourdomain.com/"
   languageCode = "en-us"
   title = "My Blog"
   theme = "PaperMod"
   paginate = 10

   [params]
     author = "Your Name"
     showReadingTime = true
  ```
  5. **Add Some Content:**  
   Create a sample post:
   ```bash
   hugo new posts/my-first-post.md
   ```
   Edit the Markdown file in `content/posts/` to include your article content.

6. **Test Locally:**  
   Run the local server to preview your site:
   ```bash
   hugo server -D
   ```
## 2. Set Up GitHub Actions for CI/CD

1. **Push Your Site to GitHub:**  
   Initialize a Git repository, add your files, and push them to a new GitHub repository.

2. **Set up the Dockerfile to build the site with Github Actions**
The idea is to copy build the and package our hugo site with a nginx docker in a multi-stage container deployment:
```
#####################################################################
#                            Build Stage                            #
#####################################################################
FROM hugomods/hugo:exts as builder

# Base URL
ARG HUGO_BASEURL=https://thecybersociety.io
ENV HUGO_BASEURL=${HUGO_BASEURL}

# Copy source
COPY . /src

# Debug: List contents
RUN ls -la /src && \
    echo "Theme directory contents:" && \
    ls -la /src/themes/ || echo "No themes directory" && \
    echo "Config directory contents:" && \
    ls -la /src/config/hugo/

# Build site with config path specified
RUN hugo --minify --enableGitInfo --logLevel debug --config /src/config/hugo/config.yml

# Debug: Show what was built
RUN ls -la /src/public

#####################################################################
#                            Final Stage                            #
#####################################################################
FROM hugomods/hugo:nginx

# Copy the generated files to the correct nginx path
COPY --from=builder /src/public /usr/share/nginx/html

# Copy custom nginx configuration if needed
COPY config/nginx/default.conf /etc/nginx/conf.d/default.conf

# Debug: Verify files are in the correct location
RUN ls -la /usr/share/nginx/html

# Add labels for container metadata
LABEL org.opencontainers.image.source="https://github.com/vincentbenzo/thecybersociety"
LABEL org.opencontainers.image.description="The Cyber Society Blog"
LABEL org.opencontainers.image.licenses="MIT"
```
Verify that the options are set for your usecase and the files you need for the deployment are in place.

4. **Create a GitHub Actions Workflow:**  
   Add a `.github/workflows/deploy.yml` file with the following content:
```yaml
name: Build and Deploy
 on:
  push:
    branches: [ main ]
  workflow_dispatch:
  schedule:
    # Check for updates every day at 00:00 UTC
    - cron: '0 0 * * *'
  
jobs:
  check-docker-updates:
    runs-on: ubuntu-latest
    outputs:
      images-updated: ${{ steps.check-updates.outputs.images-updated }}
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Check for image updates
        id: check-updates
        run: |
          # Pull the latest images and get their digests
          EXTS_DIGEST=$(docker pull hugomods/hugo:exts | grep -i digest | awk '{print $2}')
          NGINX_DIGEST=$(docker pull hugomods/hugo:nginx | grep -i digest | awk '{print $2}')
          
          # Create marker files with current digests
          echo "$EXTS_DIGEST" > /tmp/current_exts_digest
          echo "$NGINX_DIGEST" > /tmp/current_nginx_digest
          
          # Compare with previous digests if they exist
          if [ -f .github/digests/exts_digest ] && [ -f .github/digests/nginx_digest ]; then
            OLD_EXTS_DIGEST=$(cat .github/digests/exts_digest)
            OLD_NGINX_DIGEST=$(cat .github/digests/nginx_digest)
            
            if [ "$EXTS_DIGEST" != "$OLD_EXTS_DIGEST" ] || [ "$NGINX_DIGEST" != "$OLD_NGINX_DIGEST" ]; then
              echo "images-updated=true" >> $GITHUB_OUTPUT
            else
              echo "images-updated=false" >> $GITHUB_OUTPUT
            fi
          else
            echo "images-updated=true" >> $GITHUB_OUTPUT
          fi

      - name: Update digest files
        if: steps.check-updates.outputs.images-updated == 'true'
        run: |
          mkdir -p .github/digests
          mv /tmp/current_exts_digest .github/digests/exts_digest
          mv /tmp/current_nginx_digest .github/digests/nginx_digest
          
          git config --global user.name "github-actions[bot]"
          git config --global user.email "41898282+github-actions[bot]@users.noreply.github.com"
          
          git add .github/digests/*
          git commit -m "Update image digests [skip ci]"
          git push

  build-and-push:
    needs: check-docker-updates
    if: ${{ github.event_name == 'push' || github.event_name == 'workflow_dispatch' || needs.check-docker-updates.outputs.images-updated == 'true' }}
    runs-on: ubuntu-latest
    permissions:
      contents: write
      packages: write
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0
          
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        
      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./dockerfile-thecybersociety
          push: true
          tags: ghcr.io/vincentbenzo/thecybersociety:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
          labels: |
            org.opencontainers.image.source=https://github.com/${{ github.repository }}
            org.opencontainers.image.visibility=public
   ```

## 3. Deploy with Docker Compose

1. **Create a `docker-compose.yml`:**  
   Add the following content to deploy your site:
```yaml
version: '3'

networks:
  traefik-public-net:
    external: true

services:
  web:
    image: ghcr.io/vincentbenzo/thecybersociety:latest
    labels:
      - traefik.enable=true
      - traefik.docker.network=traefik-public-net
      - traefik.http.routers.thecybersociety-http.rule=Host(`www.thecybersociety.io`) || Host(`thecybersociety.io`)
      - traefik.http.routers.thecybersociety-http.entrypoints=http
      - traefik.http.middlewares.httptohttps.redirectscheme.scheme=https
      - traefik.http.middlewares.httptohttps.redirectscheme.permanent=true
      - traefik.http.routers.thecybersociety-http.middlewares=httptohttps
      - traefik.http.routers.thecybersociety-https.rule=Host(`www.thecybersociety.io`) || Host(`thecybersociety.io`)
      - traefik.http.routers.thecybersociety-https.entrypoints=https
      - traefik.http.middlewares.www-to-non-www.redirectregex.regex=^https?://www.thecybersociety.io/(.*)
      - traefik.http.middlewares.www-to-non-www.redirectregex.replacement=https://thecybersociety.io/$${1}
      - traefik.http.middlewares.www-to-non-www.redirectregex.permanent=true
      - traefik.http.routers.thecybersociety-https.middlewares=www-to-non-www
      - traefik.http.routers.thecybersociety-https.tls.certresolver=dnschallengecloudflare
      - traefik.http.routers.thecybersociety-https.tls.domains[0].main=*.thecybersociety.io
      - traefik.http.routers.thecybersociety-https.tls.domains[0].sans=thecybersociety.io
      - traefik.http.routers.thecybersociety-https.service=thecybersociety
      - traefik.http.services.thecybersociety.loadbalancer.server.port=80
      - "com.centurylinklabs.watchtower.enable=true"
    networks:
      - traefik-public-net
    healthcheck:
      test: ["CMD", "nginx", "-t"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
 ```

2. **Deploy the Stack:**  
   Run the following command to deploy:
   ```bash
   docker-compose up -d
   ```

## 4. Set Up Traefik and Cloudflare

1. **Configure Traefik:**  
   Ensure your Traefik instance is set up to handle SSL and route traffic to your blog. Use a `traefik.yml` or labels in your Docker Compose. I'll go in further details in coming articles.

2. **Update Cloudflare DNS:**  
   Point your domain’s DNS to Cloudflare, and configure the settings to proxy traffic to your server.
