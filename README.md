# WSL Docker Operational test

## 概要
WSL環境でDockerを動かし、コンテナでNginxを起動させる検証リポジトリです。
コンテナでNginxを起動させるまでに、いくつかのトラブルに衝突したので、そのトラブルの内容と解決できた方法を記載します。

## 検証環境
Windows 11
WSL2
Ubuntu
Docker Desktop

## 環境構築の主な手順
1, Docker Desktop インストール
2, WSL設定
3, docker compose 起動

## トラブル
Dockerが起動しない問題

## YAMLファイルの中身
今回はdockerの起動テストということで、「hello world」を確認する程度の簡素な内容です。
```
# ファイル名：docker-compose.yaml

services:
  nginx:
    image: nginx:latest           　# Docker Hub から最新のnginxイメージを使う
    container_name: nginx-test    　# コンテナに「nginx-test」という名前をつける
    ports:
     - "8080:80"                  　# PC の 8080番ポート → コンテナの 80番ポートに繋ぐ
    volumes:
    - ./html:/usr/share/nginx/html　# PC側のhtmlファイル(./html)を、コンテナ側(/usr/share/nginx/html)共有
    restart: unless-stopped       　# エラー時に自動再起動する設定
```

## トラブルその1：コンテナを起動しようとしてエラー
DockerでNginxコンテナを起動しようとした結果、下記エラーが検出された。
```
# docker-compose.yamlの内容を使ってコンテナ起動
docker compose up -d
[+] Running 0/1
 ⠋ nginx Pulling 0.0s 
error getting credentials - err: fork/exec /usr/bin/docker-credential-desktop.exe: 
exec format error, out: ``
```

これは、Linux環境の中で、Windows用のプログラムを使おうとしている状態。
Linuxは直接.exeを実行できないため、exec format errorが検出された。

### 解決方法
dockerの認証設定で、desktop.exeを使う形式が記載されているのでそれを削除する。

```
# docker認証設定ファイル（~/.docker/config.json）の中身をcatコマンドで確認
cat ~/.docker/config.json
{
  "credsStore": "desktop.exe",
}
```
```
# credential設定を削除し、auths,currentContextの設定内容をvimで記述
vim ~/.docker/config.json
{
  "auths": {},
  "currentContext": "desktop-linux"
}
```
credsStoreを削除したことで、Windows credentia helperを使用しなくなる。
ここで「docker compose up -d」でDockerを起動させてみたが、Docker contextが壊れているという問題にぶつかり、起動はできなかった。


## トラブルその2：Docker contextが壊れていた。
contextの記述に誤りがあった。
恐らくWSLとDokcer Desktopが連携時のバグで、「desktop-linuix *」という記述がスペルミスをした可能性があると思われます。
```
# 接続先確認 
docker context ls
NAME              DESCRIPTION                               DOCKER ENDPOINT                             ERROR
default           Current DOCKER_HOST based configuration   unix:///var/run/docker.sock                 
desktop-linuix *   Docker Desktop                            npipe:////./pipe/dockerDesktopLinuxEngine
```
この接続先確認において特に注視する点は「*」が付いている箇所。
この「*」が付いている箇所は、確認時点においてはアクティブなコンテキストである。


Dockerが、「desktop-linuixに繋いでね」と指示したが、desktop-linuxというコンテキストは存在せず、Dockerを正常に起動できなかった。

### 解決方法
正しいcontextに切り替えるため、下記のコマンドを実行した。
```
# 正しいcontextの内容に修正する
docker context use desktop-linux

# 再度contextの内容を確認する
docker context ls
NAME            DESCRIPTION                               DOCKER ENDPOINT                             ERROR
default       Current DOCKER_HOST based configuration   unix:///var/run/docker.sock                 
desktop-linux *   Docker Desktop                            npipe:////./pipe/dockerDesktopLinuxEngine
```

もう一度コンテナを起動させて正常に稼働できました。
```
# コンテナを起動
docker compose up -d
[+] Running 2/2
 ✔ Network nginx-study_default  Created
 ✔ Container nginx-test         Started 

# 起動確認
docker compose ps
NAME         IMAGE          COMMAND                  SERVICE   CREATED          STATUS          PORTS
nginx-test   nginx:latest   "/docker-entrypoint.…"   nginx     18 minutes ago   Up 18 minutes   0.0.0.0:8080->80/tcp, [::]:8080->80/tcp
```

## まとめ
今回のトラブルの大元は、WSLとwindows Dockerの連携ミスにありました。
Dockerインストール後、windows上で動くlinux(wsl)なら動くんじゃないかと思っていました。

やはり、OSが違う環境で実行する際は、dockerに限らず注意が必要だなと思いました。
