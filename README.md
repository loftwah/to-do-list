# To do list

This is an application for me to learn how to use Ruby and Rails with Docker for a production ready deployment. The application is a simple CMS (content management system) that allows the user to manage posts. It used a CRUD (create, read, update, delete) pattern to create the posts and also has some other non-default features.

This project aims to build a lean Docker image for use in production. Therefore it's based on the official Alpine Ruby image, uses multi-stage building and some [optimizations that I described in this blog post](https://ledermann.dev/blog/2018/04/19/dockerize-rails-the-lean-way/). This results in an image size of ~80MB.

## Features

- Auto refresh via [ActionCable](https://github.com/rails/rails/tree/master/actioncable): If a displayed post gets changed by another user/instance, it refreshes automatically using the publish/subscribe pattern
- Full text search via [Elasticsearch](https://www.elastic.co/products/elasticsearch) and the [Searchkick](https://github.com/ankane/searchkick) gem to find post content (with suggestions)
- Autocompletion with [autocompleter](https://github.com/kraaden/autocomplete)
- Editing HTML content with the WYSIWYG JavaScript editor [Trix](https://github.com/basecamp/trix)
- Uploading images directly to S3 with the [Shrine](https://github.com/janko-m/shrine) gem and [jQuery-File-Upload](https://github.com/blueimp/jQuery-File-Upload)
- Background jobs with [ActiveJob](https://github.com/rails/rails/tree/master/activejob) and the [Sidekiq](http://sidekiq.org/) gem (to handle full text indexing, image processing and ActionCable broadcasting)
- Cron scheduling with [Sidekiq-Cron](https://github.com/ondrejbartas/sidekiq-cron) to handle daily data updates from Wikipedia
- Permalinks using the [FriendlyId](https://github.com/norman/friendly_id) gem
- Infinitive scrolling (using the [Kaminari](https://github.com/kaminari/kaminari) gem and some JavaScript)
- User authentication with the [Clearance](https://github.com/thoughtbot/clearance/) gem
- Sending HTML e-mails with [Premailer](https://github.com/fphilipe/premailer-rails) and the [Really Simple Responsive HTML Email Template](https://github.com/leemunroe/responsive-html-email-template)
- Admin dashboards with [Blazer](https://github.com/ankane/blazer) gem
- JavaScript with [Stimulus](https://stimulusjs.org/)
- Bundle JavaScript libraries with [Yarn](https://yarnpkg.com)

## Multi container architecture

There is an example **docker-compose.production.yml**. The whole stack is divided into multiple different containers:

- **app:** Main part. It contains the Rails code to handle web requests (by using the [Puma](https://github.com/puma/puma) gem). See the [Dockerfile](/Dockerfile) for details. The image is based on the Alpine variant of the official [Ruby image](https://hub.docker.com/_/ruby/) and uses multi-stage building.
- **worker:** Background processing. It contains the same Rails code, but only runs Sidekiq
- **db:** PostgreSQL database
- **elasticsearch:** Full text search engine
- **redis:** In-memory key/value store (used by Sidekiq, ActionCable and for caching)
- **backup:** Regularly backups the database as a dump via CRON to an Amazon S3 bucket

## Environmental Variables

This application uses the following environmental variables:

```bash
ELASTICSEARCH_HOST=elasticsearch
REDIS_SIDEKIQ_URL=redis://redis:6379/0
REDIS_CABLE_URL=redis://redis:6379/1
REDIS_CACHE_URL=redis://redis:6379/2
SECRET_KEY_BASE=f0ee8496d1ddeb69fa0980d0580ff0f174262e61b53e8a318c990a193661f45f84b07a995d33ca8749eb6e0085d375201d319ed0fdaab9aa469e3b9b96f269d6
DB_HOST=db
DB_USER=postgres
DB_PASSWORD=foobar123
RAILS_MAX_THREADS
APP_HOST
APP_SSL
FRONTEND_HOST
APP_ADMIN_EMAIL=admin@example.org
APP_ADMIN_PASSWORD=secret
APP_EMAIL=reply@example.org
PIWIK_HOST=piwik.example.org
PIWIK_ID=42
SMTP_SERVER
SMTP_PORT
SMTP_DOMAIN
SMTP_USERNAME
SMTP_PASSWORD
SMTP_AUTHENTICATION
SMTP_ENABLE_STARTTLS_AUTO
AWS_ACCESS_KEY_ID=1234
AWS_SECRET_ACCESS_KEY=1234abcdef
AWS_BUCKET=mybucket
AWS_REGION=ap-southeast-2
BLAZER_DATABASE_URL
```

## Usage

To start up the application in your local Docker environment:

```bash
git clone https://github.com/loftwah/to-do-list.git
cd to-do-list
docker-compose build
docker-compose up
```

It will take some to come up, so you'll need to wait a little while but then you can navigate to [localhost:52243](http://localhost:52243).

Sign in to the admin account with the following credentials:

- Username: `admin@example.org`
- Password: `secret`

## Tests / CI

On every push, the test suite (including [RuboCop](https://github.com/bbatsov/rubocop) checks) is performed via [GitHub Actions](https://github.com/ledermann/docker-rails/actions). If successful, a production image is built and pushed to GitHub Container Registry.

## Production deployment

The Docker image build includes precompiled assets only (no node_modules and no sources). The [spec folder](/spec) is removed and the Alpine packages for Node and Yarn are not installed.

The stack is ready to host with [traefik](https://traefik.io/) or [nginx proxy](https://github.com/jwilder/nginx-proxy) and [letsencrypt-nginx-proxy-companion](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion).
