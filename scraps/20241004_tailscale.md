# docker compose内のあるコンテナに直接sshできるようにする

既にあるコンテナにホストを通さず直接sshしたい。docker composeでホストのsshとは別にポートを開けて、ホストのパケットフィルタを変えて、ssh踏み台を通して、とすごくめんどくさいし設定の管理がしんどいので、tailscaleを使ってみる。

## 前提

- サイドカー的にtailscaleを立てられる方が関心が分離できてよいが、ssh接続時に元のコンテナに直送するような方法が見つからなかったので、元のコンテナにアドオンする。このため、本家コンテナをそのまま立ち上げるのではなく、既存のDockerfileの最後でtailscaleを入れるような方針。
- 元コンテナはもともとnon-rootでログインして作業する想定だったので、docker compose execで (デフォルトで) non-rootアカウントに入れるようにする。※ セキュリティ目的ではなくあくまで利便性

## うまく行った例
以下が最低限の疎通が試せた時点での設定。

```yaml
services:
  tailscale:
    build:
      context: .

    volumes:
      - ./tailscale:/var/lib/tailscale # State data will be stored in this directory
      - /dev/net/tun:/dev/net/tun # Required for tailscale to work
    environment:
      - TS_AUTHKEY=tskey-auth-xxxxxxxxxxxxxxxxx-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
      - TS_HOSTNAME=console # Display name in tailscale console
      - TS_STATE_DIR=/var/lib/tailscale
    cap_add:
      - net_admin
      - sys_module
```

```
FROM debian:bullseye-slim

# このコンテナの元々のセットアップいろいろ

# このコンテナでは元々non-rootで通常作業する想定
RUN groupadd -g 1000 debian
RUN adduser --disabled-password --gecos "" --uid 1000 --gid 1000 --home /home/debian debian
WORKDIR /home/debian/

# tailscaleのための追加設定
## 本家コンテナで作られたcontainerboot tailscale tailscaledを持ってくる
COPY --from=tailscale/tailscale:stable /usr/local/bin/* /usr/local/bin/
## tailscaleだけをrootで実行できるように、non-rootからsudo -Eを使えるようにする
RUN apt-get update && apt-get install -y sudo \
 && echo debian ALL=NOPASSWD:SETENV: /usr/local/bin/containerboot >> /etc/sudoers
ENV TS_EXTRA_ARGS=--ssh
USER debian

ENTRYPOINT ["sudo", "-E", "/usr/local/bin/containerboot"]
```


## うまく行かなかった例
- 非コンテナでのセットアップ手順であるcurlで取得 → tailscaledという方法だと、daemonは普通に立ち上がるがloginなど連続してやりたい操作をうまく入れられない。むしろ本家コンテナがtailscaledとtailscaleの両方の操作を1つのentrypointでまとめてやってくれるとこがすごいようなので、それを活用すべき。

  ```
  RUN apt-get update && apt-get install -y curl
  RUN curl -fsSL https://tailscale.com/install.sh | sh
  ENTRYPOINT ["/usr/sbin/tailscaled"]
  ```

- non-rootで起動するためにSUIDをセットする方法 (`chmod +s /usr/local/bin/containerboot`) の場合、tailscaledは立ち上がるが、実際にssh接続する際にpermission deniedとなった。
