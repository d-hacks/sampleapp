# Flask Sample Application "Imasugudemo"

Flaskを使ったデモを作成するための雛形コードです。
Ajaxを使ったテキストと画像ファイルのPOSTができます。

実行環境
Ubuntu 16.04.5
Python 3.5.2

[![Image from Gyazo](https://i.gyazo.com/8acb45c938f575b9d2d26528fb7e9dae.gif)](https://gyazo.com/8acb45c938f575b9d2d26528fb7e9dae)


## 導入

- 新しく作るFlaskアプリのディレクトリ名`newapp`を自分の作りたいアプリの名前に変更してください
- `/home/hirono/projects` を自分がFlaskアプリを置くpathに変更してください
- `hirono` を自分のユーザー名に変更してください


## 1. 雛形Flaskアプリのclone

1. カレントディレクトリ以下の任意の場所に雛形Flaskアプリをclone
```
$ git clone git@github.com:d-hacks/imasugudemo.git newapp ## newappの部分を書き換えてください
```

2. アプリ名を変更

```
$ mv newapp/imasugudemo.py newapp/newapp.py
```

3. Nginx、uWSGI、systemd serviceの設定ファイルを作成
```
$ cd /home/hirono/projects/newapp/src
$ python create_config_files.py --appname 'newapp' --appdirpath '/home/hirono/projects' --user 'hirono'
```

## 2. virtualEnvの作成

1. 対象のFlaskアプリのディレクトリの中でvenvを立ち上げる

```
$ cd /home/hirono/projects/newapp
$ python -m venv newappenv
$ source newappenv/bin/activate
```

2. 使用しているライブラリ全てと、Flask、uWSGIをvenv内にインストール
```
(newappenv) $ pip install uwsgi flask
```

3. venvから出る

```
(newappenv) $ deactivate
```

## 3. systemd serviceの設定
1. /etc/systemd/system/直下に `newapp.service` を移動
```
$ sudo mv /home/hirono/projects/newapp/newapp.service /etc/systemd/system/newapp.service
```

2. serviceの起動 + statusの確認
```
$ sudo systemctl start newapp
$ sudo systemctl enable newapp
$ sudo systemctl status newapp
```

## 4. Nginxの設定

1. /etc/nginx/sites-available/flaskappに `newapp.conf` の中身を追記

- newapp.confは `newapp/` 直下に作られています

```
$ sudo vim /etc/nginx/sites-available/flaskapp
```

```
（/etc/nginx/site-avalable/flaskappの中身）

server {
  listen 443;
  server_name ***;

  ssl on;
    ssl_certificate ***; # managed by Certbot
    ssl_certificate_key ***; # managed by Certbot

  location / {
          index index.html index.htm index.nginx-debian.html;
  }

  ## 他の人のlocationの記述

  ## ↓↓↓この最後の部分にnewapp.confの中身を以下のように貼り付ける
  ## newapp - hirono
  location ~ ^/newapp(.*)$ {
    root /home/hirono/projects/newapp;
    include uwsgi_params;
    uwsgi_pass unix:/home/hirono/projects/newapp/newapp.sock;
    uwsgi_param SCRIPT_NAME /newapp;
    uwsgi_param PATH_INFO /$1;
  }
  location ^~ /newapp/static/js   { root /home/hirono/projects/; }
  location ^~ /newapp/static/css  { root /home/hirono/projects/; }
  location ^~ /newapp/static/img  { root /home/hirono/projects/; }
  ### ↑↑↑
}

```

2. Nginxの設定にエラーがないかテスト
```
$ sudo nginx -t
```

3. Nginxをrestart
```
$ sudo systemctl restart nginx
```

4. https://bacchus.ht.sfc.keio.ac.jp/newapp にアクセス


## エラー対処方法

### とりあえずやってみること

- キャッシュの消去とハード再読み込み

- serviceの再起動→nginxの再起動

```
$ sudo systemctl stop newapp
$ sudo systemctl start newapp
$ sudo systemctl restart nginx
```

- nginxのログを見る
```
$ sudo tailf /var/log/nginx/access.log
$ sudo tailf /var/log/nginx/error.log
```


### 画像、CSS、JSが読み込まれない
- アプリ内の`<script>` `<img>` `<link>`タグ内のリンクを書き換える

```
（例）
<img src="/static/img/logo.img">
<link rel="stylesheet" href="/static/css/mystyle.css">
<script src="/static/js/myscript.js"></script>
↓
<img src="/imasugudemo/static/img/logo.img">
<link rel="stylesheet" href="/imasugudemo/static/css/mystyle.css">
<script src="/imasugudemo/static/js/myscript.js"></script>
```

※ locationの評価条件にimg, js, cssを明記しないと、nginxのデフォルトの設定で/usr/www/html以下にファイルを探しに行ってしまう。

またGETでリクエストされるurlからどのアプリのファイルをリクエストしているのかを判断しなければならないため、タグ内のリンクにアプリ名を含めなければならない（Flaskの基本設定には反する書き方になります）。詳しくは https://www.digitalocean.com/community/questions/nginx-image-folder-and-declaration

### AjaxのGET/POSTができない
- jsファイルの中のajaxのurlを書き換える
```
$.ajax({
         url: '/upload'

$.ajax({
         url: '/newapp/upload_image',
```

### 参考URL
独自に作った場合はこの記事を参考にすると良いです!
- https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-uswgi-and-nginx-on-ubuntu-18-04

Ajax
- https://chaika.hatenablog.com/entry/2014/04/12/001012

Nginxが再起動できない
- https://teratail.com/questions/49145
- https://scribble.washo3.com/linux/nginx%E3%81%A7%E5%81%9C%E6%AD%A2%E3%83%BB%E5%86%8D%E8%B5%B7%E5%8B%95%E3%81%8C%E5%87%BA%E6%9D%A5%E3%81%AA%E3%81%84%E3%81%A8%E3%81%8D.html
