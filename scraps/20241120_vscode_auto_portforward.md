# VSCodeの自動ポートフォワードがうまく行かなくて調べた

## 1. 状況
phpのbuilt in serverを使うdevcontainerで、launch.jsonに書いた設定に基づいてF5から[vscode-php-debug](https://github.com/xdebug/vscode-php-debug/tree/main)を立ち上げた際にVSCodeがポートフォワードしてくれない。

VSCode内で立ち上げたコンソール上で手でphp -S ...を実行した際は自動でポートフォワードしてくれる。

```
$ php -S 0.0.0.0:8888
PHP 7.3.33 Development Server started at Wed Nov 20 04:16:58 2024
Listening on http://0.0.0.0:8888
Document root is /prg/www/html
Press Ctrl-C to quit.
```

## 2. 確認項目1: remote.autoForwardPortsの設定
自分の場合、なぜか remote.autoForwardPortsSource を hybrid にしていて、php built in serverがポートを開いたのを見落としていた。つーかこれhybridとはあまり思えない…。

以下、 https://code.visualstudio.com/docs/getstarted/settings より。

```
  // When enabled, new running processes are detected and ports that they listen on are automatically forwarded. Disabling this setting will not prevent all ports from being forwarded. Even when disabled, extensions will still be able to cause ports to be forwarded, and opening some URLs will still cause ports to forwarded.
  "remote.autoForwardPorts": true,

  // The number of auto forwarded ports that will trigger the switch from `process` to `hybrid` when automatically forwarding ports and `remote.autoForwardPortsSource` is set to `process`. Set to `0` to disable the fallback.
  "remote.autoForwardPortsFallback": 20,

  // Sets the source from which ports are automatically forwarded when `remote.autoForwardPorts` is true. On Windows and macOS remotes, the `process` and `hybrid` options have no effect and `output` will be used.
  //  - process: Ports will be automatically forwarded when discovered by watching for processes that are started and include a port.
  //  - output: Ports will be automatically forwarded when discovered by reading terminal and debug output. Not all processes that use ports will print to the integrated terminal or debug console, so some ports will be missed. Ports forwarded based on output will not be "un-forwarded" until reload or until the port is closed by the user in the Ports view.
  //  - hybrid: Ports will be automatically forwarded when discovered by reading terminal and debug output. Not all processes that use ports will print to the integrated terminal or debug console, so some ports will be missed. Ports will be "un-forwarded" by watching for processes that listen on that port to be terminated.
  "remote.autoForwardPortsSource": "process",
```

というわけでデフォルトのprocessにすれば解決はしたが、それに気づかず↓のことをずっと調べてしまったのでこれも供養する。

## 3. 確認項目2: autoForwardPortsSourceがoutput/hybirdの場合のDEBUG CONSOLEの表示

autoForwardPortsSourceがoutput/hybirdの時は上記の通りterminal and debug outputからそれっぽい文字列を見つかった場合に限り自動ポートフォワードされる。
ところが、php built in serverをlaunch.json → F5で起動した場合、DEBUG CONSOLEタブには起動直後に何も表示されない。
DEBUG CONSOLEがstdoutを表示しない仕様かとも思ったが、起動直後以降のログは出ている。

途中で気づいたのは、VSCodeのデバッグとしてプロセスを立ち上げるということはttyが繋がっていないということ。
そこで、以下のようなスクリプトをlaunch.jsonの runtimeExecutable に設定してみる。

```bash
#!/bin/bash -e

trap 'kill $(jobs -p)' EXIT

# Force attach tty to php server to show start up logs like:
# This enables automatic port forward and popup notification when debbuging in vscode
QUOTED_ARGS=$(printf "'%s' " "$@") # php '-S' '0.0.0.0:8888' ...
script -q -c "php ${QUOTED_ARGS}" /dev/null
```

この場合、DEBUG_CONSOLEにphp built in server起動直後の`Listening on http://0.0.0.0:8888`が表示され、VSCodeがポートフォワードを見つけてくれるようになった。
