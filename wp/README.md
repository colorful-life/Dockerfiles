# WordPress with Xdebug, MailCatcher

https://hub.docker.com/r/colife/wordpress-xdebug

From [WordPress Official Image](https://hub.docker.com/_/wordpress/)

---
## docker-compose.yml
```yaml
version: '3.4'

services:

  # MySQL
  # =======================================
  wpdb:
    image: mysql:5.7
    # Workbench等で接続出来る様にポート割り当て： Standard(TCP/IP)にて、127.0.0.1:{割り当てたポート} で接続可
    ports:
      - "${PORT_MYSQL:-3307}:3306"
    restart: always
    volumes:
      - ./db_data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:-wordpressroot}
      MYSQL_DATABASE: ${DB_NAME:-wordpress}
      MYSQL_USER: ${DB_USER:-wordpress}
      MYSQL_PASSWORD: ${DB_PASSWORD:-wordpress}


  # WordPress
  # =======================================
  wordpress:
    depends_on:
      - wpdb
    image: colife/wordpress-xdebug:latest
    # xdebugを含める為、wordpressのimageを直接使用せずに Dockerfileからインストールする
    # build:
    #   context: .
    #   dockerfile: Dockerfile
    ports:
      - "${PORT_WP:-8080}:80"
    restart: always
    volumes:
      - ./html:/var/www/html
      - ./wp-install.sh:/tmp/wp-install.sh
      # - ./php.ini:/usr/local/etc/php/php.ini
      - ./php.ini:/usr/local/etc/php/conf.d/zzz-php.ini
      - ./.tmp/log:/tmp/log
    working_dir: /var/www/html${WP_DIR}
    environment:
      WORDPRESS_DB_HOST: wpdb
      WORDPRESS_DB_NAME: ${DB_NAME:-wordpress}
      WORDPRESS_DB_USER: ${DB_USER:-wordpress}
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD:-wordpress}
      WORDPRESS_TABLE_PREFIX: ${WP_TABLE_PREFIX:-wp_}
      WORDPRESS_DEBUG: ${WP_DEBUG:-null}
      WORDPRESS_CONFIG_EXTRA: |
        define('JETPACK_DEV_DEBUG', true);
        define('SAVEQUERIES', false);


  # MailCatcher
  # ======================================
  # wordpressから mailcatcherを使用するには・・・
  #   - phpに mailcatcherをインストールし、php.iniの sendmailのパスにcatchmailを設定する
  #       * Dockerfile、php.ini に反映済み
  #       * 要 WP Multibyte Patch。 素のwp_mail()だと、送信の際にmailcatcherで文字コードの関係でエラーになる感じ
  #      OR
  #   - WP Mail SMTP by WPForms プラグインをインストールする
  #       --- WP Mail SMTP の設定 ---
  #       * Mailer : Other SMTP
  #       * SMTP Host : mailcatcher (下記サービス名)
  #       * SMTP Port : 1025        (下記 ports で割り当てたポート)
  #       * Auto TLS : OFF
  #       * Authentication : OFF
  #       --- /WP Mail SMTP の設定 ---
  mailcatcher:
    image: schickling/mailcatcher
    ports:
      - "${PORT_MC_UI:-1080}:1080"
      - "${PORT_MC_SMTP:-1025}:1025"


  # phpMyAdmin
  # =======================================
  # phpMyAdminを使用するなら、以下アンコメント
  # pma:
  #   depends_on:
  #     - wpdb
  #   image: phpmyadmin/phpmyadmin
  #   ports:
  #     - "8080:80"
  #   restart: always
  #   environment:
  #     PMA_HOST: wpdb


# volumes:
#   db_data: {}

```

---
## .env
```yaml
######### docker-compose用 設定 #########

# コンテナのサービス名の先頭に付けられるプロジェクト名。 デフォルトはディレクトリ名
COMPOSE_PROJECT_NAME=

# ==== localhostポート割り当て ========
# wordpress: 80
PORT_WP=8100
# MySql: 3306
PORT_MYSQL=4306
# MailCatcher（WebUI）: 1080
PORT_MC_UI=1080
# MailCatcher（SMTP）: 1025
PORT_MC_SMTP=1025


# ==== WordPress ========
# wordpressをサブディレクトリに配置する場合は指定する（先頭にスラッシュを付ける事）
# インストール後に サイトアドレスの変更等を行えば、ホームはルートディレクトリ、WPはサブディレクトリ、でも動作する
WP_DIR=/wp

# $table_prefix： default => wp_
WP_TABLE_PREFIX=wp_

# WP_DEBUG： 1 or null
WP_DEBUG=1


# ==== WordPress & MySQL ========
DB_NAME=wordpress
DB_USER=wordpress
DB_PASSWORD=wordpress


# ==== MySQL ========
MYSQL_ROOT_PASSWORD=wordpressroot

```

---
## php.ini
```ini
default_charset = UTF-8
short_open_tag = 0
post_max_size = 1024M
upload_max_filesize = 1024M

log_errors = 1
html_errors = 1
error_reporting = E_ALL
; error_log = /tmp/log/php_errors.log

date.timezone = UTC

mbstring.language = neutral
mbstring.internal_encoding = UTF-8

[mail function]
; sendmailのパスにcatchmailを設定する。 WP Mail SMTP を使用するならコメントアウト
sendmail_path = /usr/bin/env catchmail --smtp-ip mailcatcher --smtp-port 1025
; mail.log = /tmp/log/mail.log

[xdebug]
xdebug.remote_port = 9000
; ホスト側のIP - dockerのhostマシンのIPを解決
xdebug.remote_host = host.docker.internal
; xdebug.remote_connect_back を有効にすると xdebug.remote_host が無効になるので、 remote_connect_back は無効にする
xdebug.remote_connect_back = 0
xdebug.remote_enable = 1
xdebug.remote_autostart = 1
xdebug.collect_params = 0
xdebug.var_display_max_children = -1
xdebug.var_display_max_data = 1024
xdebug.var_display_max_depth = -1
; xdebug.remote_log = /tmp/log/xdebug.log

```
---
## wp-install.sh
```sh
#!/bin/bash
set -ex;

### WPをインストールしたパス： .env で設定したパスと合わせる ###
WP_INSTALL_DIR=/var/www/html/wp
### サイトURL： .env で設定したポート、サブディレクトリと合わせる ###
WP_URL=localhost:8080/wp
### インストールするプラグイン ###
PLUGINS=(
  # "duplicate-post"
  # "advanced-custom-fields"
  "classic-editor"
  # "tinymce-advanced"
  "wp-multibyte-patch"
)
###### WPのダウンロード（必要ならアンコメント） #####
# wp core download \
#   --locale=ja \
#   --version=4.9.1 \
#   --path=${WP_INSTALL_DIR}  \
#   --force \
#   --allow-root

###### WPのインストール： タイトル、ユーザー等の設定 #####
wp core install \
  --path=${WP_INSTALL_DIR} \
  --url=${WP_URL} \
  --title=テスト用WordPress \
  --admin_user=admin \
  --admin_password=admin \
  --admin_email=admin@docker.test \
  --allow-root

###### プラグインのインストール #####
for plugin in ${PLUGINS[@]}; do
  ### インストール＆有効化するなら ###
  # wp plugin install $plugin --activate --path=${WP_INSTALL_DIR} --allow-root
  ### 有効化はせずにインストールするのみ ###
  wp plugin install $plugin --path=${WP_INSTALL_DIR}  --allow-root
done
```

1. WP用のコンテナにbashで入る。 ※ 下記 `wordpress` は、WPのコンテナ名
   ```sh
   $ docker-compose exec wordpress bash
   ```
2. WP用のコンテナにマウントしておいた wp-install.sh のパーミッションを設定して、実行。
   ```bash
   $ chmod +x /tmp/wp-install.sh
   ```
   ```bash
   $ /tmp/wp-install.sh
   ```

---
## VS Code with PHP Debug - .vscode/launch.json
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Listen for XDebug",
      "type": "php",
      "request": "launch",
      "port": 9000,
      "pathMappings": {
        "/var/www/html": "${workspaceFolder}"
      }
    },

  ]
}
```
