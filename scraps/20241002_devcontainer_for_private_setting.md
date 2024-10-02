# 個人設定のためにdevcontainerを使う
workspace固有かつ個人専用の設定 (themeなど) をdevcontainer.json内で設定してgitignoreするとやりやすい。

## 実例
こんなdevcontainer.jsonを作る。

```json
{
	"name": "Debian",
	"image": "mcr.microsoft.com/devcontainers/base:bullseye",

	// Configure tool-specific properties.
	"customizations": {
		"vscode": {
            "settings": {
                "workbench.colorTheme": "Solarized Dark"
			}
            "extensions": [
                "${containerWorkspaceFolder}/.devcontainer/yaml-todo-v0.0.2-231226200b0afa0165ffc3c562530cf9b9633763.0.vsix"
            ],
		}
	}
}
```

プロジェクト自体が既にdevcontainerを使っている場合 (それがgit管理対象になっている場合) は、そのdevcontainer.jsonをコピーしたものから上記の個人設定を足す必要があるはず。そしてそのdevcontainer_copy.jsonだけをgitignoreする感じ。

## ユースケース
### このworkspaceだけ拡張機能の設定を変えたい
WSL2本体に大量にextensionが入っている場合に、それとの干渉を避け隔離された環境を作りたい、など。
devcontainerを使うと自然と環境が隔離されるので、

### このworkspaceだけ手元で別テーマで表示したい
.vscode/settings.jsonにworkbench.colorThemeを設定することでテーマを適用することは可能だが、それ以外の設定がsetting.jsonにあってそれがgit管理対象だと困る。
そこで、上記の例のようにdevcontainer.json内のsettingsに記載すると、必然的にworkspace専用になる。

### 独自の拡張機能をインストールする
予めvisxファイルまでできていれば、それをローカルにおいて、上記のようにdevcontainer.json内のextensionsで設定すればよい。

https://www.kenmuse.com/blog/implementing-private-vs-code-extensions-for-dev-containers/

> You may see a warning in Visual Studio, since the schema file does not include the pattern for referencing files, but the feature works and is supported:
