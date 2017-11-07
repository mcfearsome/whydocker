# why docker?
![Docker Logo](https://www.brianweet.com/assets/docker-blog-1/docker-logo.png)

---

# Becasue I Said So!

---

### Isolated and Explicit Environment
1. Dependecies are specific to the project
2. The OS your code runs in doesn't change unless you change it

---

### Consitent Playing field
1. Easier to rule out issues if people are always using the same OS
2. Once setup in a project, easier to get a project running for new devs (personal opinion?)

---

### Security
1. How many of you have failed to update your OS for fear of breaking your development env?

---

# Now the Scary Stuff!

---

### Prereqs

1. [Docker for Mac](https://www.docker.com/docker-mac)
2. [Docker Sync](http://docker-sync.io/)
3. [Powder](https://github.com/powder-rb/powder)

```bash
gem install docker-sync
# for the .dev resolver
gem install powder
```

---

### The Dockerfile

```Dockerfile
FROM ruby:2.3.3

MAINTAINER Jesse McPherson, jmcpherson@weblinc.com

ENV MAIN_PACKAGES build-essential libpq-dev git xvfb nodejs jpegoptim
ENV PHANTOM_JS_DEPENDENCIES\
  libicu-dev libfontconfig1-dev libjpeg-dev libfreetype6

ENV PHANTOM_JS_TAG 2.1.1

# Installing phantomjs
# Installing dependencies
RUN apt-get update
RUN apt-get install -fyq ${MAIN_PACKAGES} ${PHANTOM_JS_DEPENDENCIES}

# Downloading bin, unzipping & removing zip
WORKDIR /tmp
RUN wget -q http://cnpmjs.org/mirrors/phantomjs/phantomjs-${PHANTOM_JS_TAG}-linux-x86_64.tar.bz2 -O phantomjs.tar.bz2 \
  && tar xjf phantomjs.tar.bz2 \
  && rm -rf phantomjs.tar.bz2 \
  && mv phantomjs-* phantomjs \
  && mv /tmp/phantomjs/bin/phantomjs /usr/local/bin/phantomjs

RUN apt-get autoremove -yq \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

ENV app /weblinc-direct
RUN mkdir $app
WORKDIR $app

ENV BUNDLE_PATH $app/docker/bundle

RUN gem install bundler -v 1.15.1

EXPOSE 3000
CMD (bundle check || bundle install) && bundle exec puma -e development -p 3000
```

---

### docker-compose.yml
```YML
version: '2'
services:
  mongodb:
    image: mongo:3.2.9
    ports:
    - "27099:27017"

  redis:
    image: redis:3.2.3
    expose:
    - "6379"

  es:
    image: elasticsearch:1.5.2
    ports:
    - "9299:9200"

  app:
    build: .
    ports:
    - "3099:3000"
    links:
    - mongodb
    - redis
    - es
    env_file:
    - .docker.env
    - .secrets.env
    volumes:
      - ja-sync:/weblinc-direct:nocopy

volumes:
  ja-sync:
    external: true
```

---

### docker-sync.yml
```yml
version: "2"
options:
  verbose: false
syncs:
  ja-sync:
    notify_terminal: true
    src: '.'
    sync_excludes: ['tmp/*', 'public/*']
```

---

### Some Extras

#### .docker.env
```sh
WEBLINC_ELASTICSEARCH_URL=http://es:9200
WEBLINC_REDIS_HOST=redis
WEBLINC_MONGOID_HOST_0=mongodb:27017
DOCKER=true
```

#### .secrets.env
```sh
BUNDLE_GEMS__WEBLINC__COM=user:pass
```

#### spec/support/capybara.rb
```ruby
if ENV['DOCKER']
  Capybara.configure do |config|
    config.save_path = '/weblinc-direct/docker/capybara'
  end
end
```

---

### Putting it all together

#### Build the image
```
docker-compose build
```

#### Link the .dev
```
powder portmap {port from docker-compose.yml}
```

#### Start up the sync
```
docker-sync start
```

#### Start up the App
```
docker-compose up
```

#### Get a shell
```
docker-compose run app bash
```

---

### The Breaks

1. Can be resource heavy if running a lot of containers at once
2. Sometimes the sync needs to be cleaned and reset
3. Cannot use save_and_open_screenshot (just save_screenshot)
4. Not ready for v3 (having an issue running all tests)
