
FROM wordpress:4.9-php7.2-apache

# Xdebug
RUN pecl install xdebug-2.9.2 \
  && docker-php-ext-enable xdebug

# MailCatcher
RUN apt-get update -qq \
  && apt-get install -y \
    ruby \
    ruby-dev \
    rubygems \
    libsqlite3-dev \
  && apt-get autoremove \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*
RUN gem install mailcatcher

WORKDIR /var/www
