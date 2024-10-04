# docker compose内のあるコンテナに直接sshできるようにする

既にあるコンテナにホストを通さず直接sshしたい。docker composeでホストのsshとは別にポートを開けて、ホストのパケットフィルタを変えて、ssh踏み台を通して、とすごくめんどくさいし設定の管理がしんどいので、tailscaleを使ってみる。

ひとまず最低限の疎通を試す。サイドカー的にtailscaleを立てられる方が関心が分離できてよいが、ssh接続時に元のコンテナに直送するような方法が見つからなかったので、元のコンテナにアドオンする。

```yaml
services:
  tailscale:
    # image: tailscale/tailscale:stable
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
# docker compose execしたときなどに (デフォルトでは) non-rootで接続したいが、tailscaledはrootで実行必要なので
# sudo NOPASSWDできるようにしておく
RUN apt-get update && apt-get install -y curl sudo -y \
 && curl -fsSL https://tailscale.com/install.sh | sh \
 && gpasswd -a debian sudo \
 && echo debian ALL=NOPASSWD: /usr/sbin/tailscaled >> /etc/sudoers

USER debian

ENV TS_EXTRA_ARGS=--ssh
ENTRYPOINT ["sudo", "/usr/sbin/tailscaled"]
```
