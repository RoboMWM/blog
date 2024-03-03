Seeing that there's a lot of notes for a lot of things I've done over the years, I've decided to formalize this knowledge, whether it be for future me or for future others.

One of my latest adventures was hosting my own Mastodon instance. I initially tried following the install guide, but ran into various issues that assumed I had some prerequisites installed that weren't fully detailed in the guide. Things like yarn and such, which I later found out I had to do extra things after pasting in errors to Bing AI (which I guess I should now call "Copilot"). I continued to struggle with this for a few hours, until deciding to ditch it and have a go at trying to run it in docker with docker-compose. I figured it'd be a good way to get some docker/container experience outside of work.

I followed [this guide](https://www.bentasker.co.uk/posts/blog/general/running-mastodon-in-docker-compose.html) which helped significantly, but I had to change a few things. As such, I'll try to recreate that guide here (albeit more simplified).

## Assumptions

I ran these commands with a user account that has access to running docker. `sudo usermod -a -G docker $USER`

## Getting docker-compose.yml

Cloning the repo is primarily used to obtain its docker-compose file. Not sure if I used the repo directly for anything beyond this. In addition to cloning, we then create and switch to a new local branch based on the latest tag. We then copy the `docker-compose.yml` out of the repo, so volumes and other docker-compose shenagins occur outside of the git repo.

```bash
git clone https://github.com/mastodon/mastodon.git
cd mastodon
export latest=$(git describe --tags `git rev-list --tags --max-count=1`)
git checkout $latest -b ${latest}-branch

cd ..
cp mastodon/docker-compose.yml ./
```

Then you edit `docker-compose.yml` and perform the following operations:
- comment out lines starting with `build:` (I suppose cuz we're pulling these images from dockerhub instead of building from source)
- Change `image:` value to a different, tagged version. (I ended up using `tootsuite/mastodon:v4.1.0`. While troubleshooting, I found that the default-specified image, even though it has a prompt where it asks if it's being installed in Docker, won't set and print out environment variables to save to finish setup properly.)

Here's how my `docker-compose.yml` looks (don't worry about the extra `http` container for now):

```yml
version: '3'
services:
  db:
    restart: always
    image: postgres:14-alpine
    shm_size: 256mb
    networks:
      - internal_network
    healthcheck:
      test: ['CMD', 'pg_isready', '-U', 'postgres']
    volumes:
      - ./postgres14:/var/lib/postgresql/data
    environment:
      - 'POSTGRES_HOST_AUTH_METHOD=trust'

  redis:
    restart: always
    image: redis:6-alpine
    networks:
      - internal_network
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
    volumes:
      - ./redis:/data

  # es:
  #   restart: always
  #   image: docker.elastic.co/elasticsearch/elasticsearch-oss:7.10.2
  #   environment:
  #     - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
  #     - "cluster.name=es-mastodon"
  #     - "discovery.type=single-node"
  #     - "bootstrap.memory_lock=true"
  #   networks:
  #      - internal_network
  #   healthcheck:
  #      test: ["CMD-SHELL", "curl --silent --fail localhost:9200/_cluster/health || exit 1"]
  #   volumes:
  #      - ./elasticsearch:/usr/share/elasticsearch/data
  #   ulimits:
  #     memlock:
  #       soft: -1
  #       hard: -1

  web:
   # build: .
    image: tootsuite/mastodon:v4.1.0
    restart: always
    env_file: .env.production
    command: bash -c "rm -f /mastodon/tmp/pids/server.pid; bundle exec rails s -p 3000"
    networks:
      - external_network
      - internal_network
    healthcheck:
      # prettier-ignore
      test: ['CMD-SHELL', 'wget -q --spider --proxy=off localhost:3000/health || exit 1']
    ports:
      - '127.0.0.1:3000:3000'
    depends_on:
      - db
      - redis
      # - es
    volumes:
      - ./public/system:/mastodon/public/system

  streaming:
   # build: .
    image: tootsuite/mastodon:v4.1.0
    restart: always
    env_file: .env.production
    command: node ./streaming
    networks:
      - external_network
      - internal_network
    healthcheck:
      # prettier-ignore
      test: ['CMD-SHELL', 'wget -q --spider --proxy=off localhost:4000/api/v1/streaming/health || exit 1']
    ports:
      - '127.0.0.1:4000:4000'
    depends_on:
      - db
      - redis

  sidekiq:
   # build: .
    image: tootsuite/mastodon:v4.1.0
    restart: always
    env_file: .env.production
    command: bundle exec sidekiq
    depends_on:
      - db
      - redis
    networks:
      - external_network
      - internal_network
    volumes:
      - ./public/system:/mastodon/public/system
    healthcheck:
      test: ['CMD-SHELL', "ps aux | grep '[s]idekiq\ 6' || false"]

  ## Uncomment to enable federation with tor instances along with adding the following ENV variables
  ## http_proxy=http://privoxy:8118
  ## ALLOW_ACCESS_TO_HIDDEN_SERVICE=true
  # tor:
  #   image: sirboops/tor
  #   networks:
  #      - external_network
  #      - internal_network
  #
  # privoxy:
  #   image: sirboops/privoxy
  #   volumes:
  #     - ./priv-config:/opt/config
  #   networks:
  #     - external_network
  #     - internal_network
  http:
    restart: always
    image: openresty/openresty
    container_name: openresty
    networks:
      - external_network
      - internal_network
    ports:
        - 443:443
        - 80:80
    volumes:
        - ./nginx/tmp:/var/run/openresty
        - ./nginx/conf.d:/etc/nginx/conf.d
        - /etc/letsencrypt/:/etc/letsencrypt/
        - ./nginx/lebase:/lebase

networks:
  external_network:
  internal_network:
    internal: true
```

## Postgres setup

I followed the guide and set a user named "mastodon" and randomly generated a password. I'm not sure if either of these are necessary, as I didn't need to do either of these with the non-docker setup instructions.

```bash
# generate a random password
cat /dev/urandom | tr -dc "a-zA-Z0-9" |fold -w 24 | head -n 1

# spin up the postgres container
# <image name> is the `image:` value for the postgres container, e.g. postgres:14-alpine
docker run --rm --name postgres \
-v <volume path>:/var/lib/postgresql/data \
-e POSTGRES_PASSWORD=<password> \
-d <image name>
```

Then you'll `docker exec` into the container with `psql` to create the user:

```bash
docker exec -it postgres psql -U postgres
```

You'll be in a `psql` shell

```bash
CREATE USER mastodon WITH PASSWORD '<password>' CREATEDB;
exit
```

Then when done, stop the postgres container
```bash
docker stop postgres
```

## Mastodon setup script

Apparently some environment variables need to be set in a file named `.env.production` in the same directory as `docker-compose.yml` for Mastodon setup to work properly. If you've been using the `mastodon` username, create a `.env.production` file that looks like this:

```
DB_HOST=db
DB_PORT=5432
DB_NAME=mastodon
DB_USER=mastodon
DB_PASS=<password>
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=
```

Now we can launch mastodon's interactive setup script:

```bash
docker-compose run --rm web bundle exec rake mastodon:setup
```

If you plan to use a [different domain than the one you'll use to access Mastodon](https://docs.joinmastodon.org/admin/config/#web_domain), make sure you set the `LOCAL_DOMAIN` when asked for the `Domain name` in the setup script. You can then manually set the `WEB_DOMAIN` as another entry in the `.env.production` file.

During setup, make sure to specify the PostgreSQL username and password, as these will not be default if you have followed the postgres setup of this guide earlier.

Once you're done with the SMTP step, it will print out environment variables that you should save into your `.env.production` file (which you should be able to overwrite your previous values). I opened another session to my server to do this, as mentioned in the guide, just in case these were needed for the creation of the admin user.

Also, if you're troubleshooting and need to run through Mastodon's setup script again, add this line to your `.env.production` file to allow setup to clear your database: `DISABLE_DATABASE_ENVIRONMENT_CHECK=1`

## File permissions

I'm still a linux noob, so idk how important these file permission changes are for the volumes, but I did 'em anyway.

```bash
# Briefly run containers to create volumes
docker-compose up -d
docker-compose down

# Change permissions. Iirc, I had to run these as root.
sudo chown -R 70:70 postgres
sudo chown -R 991:991 public/
```

## Now we're ready to start Mastodon

`docker-compose up -d`

## Adding a web server - Nginx

Now we need a way to access Mastodon. The install guide used Nginx, so I was already doing a little bit of that. The guide however adds it as a container to the `docker-compose.yml`, which I like. He uses a variant of Nginx named `openresty`, but given I don't care about georestrictions, vanilla Nginx would be fine too.

Add something like this to `docker-compose.yml` under the `services:` key

```yml
services:

...

  http:
    restart: always
    image: openresty/openresty
    container_name: openresty
    networks:
      - external_network
      - internal_network
    ports:
        - 443:443
        - 80:80
    volumes:
        - ./nginx/tmp:/var/run/openresty
        - ./nginx/conf.d:/etc/nginx/conf.d
        - /etc/letsencrypt/:/etc/letsencrypt/
        - ./nginx/lebase:/lebase
```

Create some of its directories so we can add a configuration straight away:

```bash
mkdir -p nginx/conf.d nginx/tmp nginx/certs
```

Create `nginx/conf.d/mastodon.conf`:

```
server {
        listen 80;
        listen   [::]:80;

        root /lebase;
        index index.html index.htm;

        server_name mastodon.yourdomain.here; # Replace with your domain name

        location ~ /.well-known/acme-challenge {
            try_files $uri $uri/ =404;
        }

        location / {
                return 301 https://$server_name$request_uri;
        }
}
```

Then start the container with `docker-compose up -d`

At this point, I believe the site should be accessible via insecure http. I did not test this.

#### Setting up https

I had installed [certbot with the mastodon install guide](https://docs.joinmastodon.org/admin/install/#acquiring-a-ssl-certificate), so I skipped these steps. If I recall correctly, I ran the following command as the root user: `certbot certonly --nginx -d mastodon.yourdomain.here`

My mistake here during troubleshooting was combining the https server declaration with the prior server declaration. I did not realize I had to have a second `server {` declaration in the nginx config. (I spent a few hours trying to find how to get logs for the nginx container and troubleshoot the various nginx messages I received.) You will add this second https configuration to your `nginx/conf.d/mastodon.conf` file, having the full resulting file look as such:

```
server {
        listen 80;
        listen   [::]:80;

        root /lebase;
        index index.html index.htm;

        server_name mastodon.yourdomain.here;

        location ~ /.well-known/acme-challenge {
            try_files $uri $uri/ =404;
        }

        location / {
                return 301 https://$server_name$request_uri;
        }
}
server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        root /mnt/none;
        index index.html index.htm;

        server_name mastodon.yourdomain.here; # Replace with your domain name


        #ssl on;

        # Replace your domain in these paths
        ssl_certificate      /etc/letsencrypt/live/mastodon.yourdomain.here/fullchain.pem;
        ssl_certificate_key  /etc/letsencrypt/live/mastodon.yourdomain.here/privkey.pem;

        ssl_session_timeout  5m;
        ssl_prefer_server_ciphers On;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;


        absolute_redirect off;
        server_name_in_redirect off;

        error_page 404 /404.html;
        error_page 410 /410.html;


        location / {
            proxy_set_header Host $http_host;
            proxy_set_header X-Forwarded-Proto https;
            proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

            proxy_pass http://web:3000;
        }

        location ^~ /api/v1/streaming {
            proxy_set_header Host $http_host;
            proxy_set_header X-Forwarded-Proto https;
            proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

            proxy_pass http://streaming:4000;

            proxy_buffering off;
            proxy_redirect off;
            proxy_http_version 1.1;
            tcp_nodelay on;
        }
}
```

If your container is already running, you can restart it with the new configuration via `docker restart openresty`, or you can be like me and just restart everything:

```bash
docker-compose down
docker-compose up -d
```

## Addendum

[Lamp](https://lamp.wtf) helped me configure a few things while we were troubleshooting connecting to his fediverse instance. Thank you Lamp!

### Set a webfinger if using a different web domain

As per [Mastodon instructions](https://docs.joinmastodon.org/admin/config/#web_domain), you'll need to set a webfinger on the site serving the `LOCAL_DOMAIN` if that differs from your `WEB_DOMAIN.` If your site is hosted with Github Pages, [these instructions](https://blog.netnerds.net/2022/11/alias-mastodon-github-pages/) make it easy to do so without having to setup 301 redirects on your webserver.

#### Github Pages

Create a `_config.yml` at repo root if not already created, and add
```yml
include: [".well-known"]
```

Then create the file `.well-known/webfinger/index.json` and copy the contents from your mastodon instance. E.g., if your `WEB_DOMAIN` is `mastodon.yourdomain.here` and your `LOCAL_DOMAIN` is `yourdomain.here`, copy contents from `https://mastodon.yourdomain.here/.well-known/webfinger?resource=acct:robomwm@yourdomain.here` and paste it into the `index.json` file you're creating.

Do note that this would only be for a single-instance mastodon.

#### Cloudflare page rule

As I continued troubleshooting with Lamp, he pointed out that I could create 301 redirects with Cloudflare's page rules for free.

![Cloudflare Page Rule](https://i.imgur.com/xOfxGtr.png)
