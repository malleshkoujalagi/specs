# コンテナ設定ファイル

コンテナのトップレベルディレクトリに、`config.json`という設定ファイルがなければなりません。
今のところ、標準的なスキーマは[config.go](../config.go)と[config_linux.go](../config_linux.go)に定義されていますが、しばらくすれば正式なJSONスキーマに置き換えられるでしょう。

>The container's top-level directory MUST contain a configuration file called `config.json`.
>For now the canonical schema is defined in [config.go](config.go) and [config_linux.go](config_linux.go), but this will be moved to a formal JSON schema over time.

この設定ファイルは、コンテナに対する標準的な操作を実装するのに必要なメタデータを含んでいます。
すなわち、実行すべきプロセスや、追加設定すべき環境変数、利用すべきサンドボックス機能等です。

>The configuration file contains metadata necessary to implement standard operations against the container.
>This includes the process to run, environment variables to inject, sandboxing features to use, etc.

下記に設定フォーマットの個々のフィールドの詳細を記述します。

>Below is a detailed description of each field defined in the configuration format.

## マニフェストのバージョン

* **`version`** (string, 必須)は[SemVer v2.0.0](http://semver.org/spec/v2.0.0.html)形式で記述されなければなりません。これはこのコンテナバンドルが準拠するOCF仕様のバージョンを示します。Open Container仕様はセマンティックによるバージョンづけと、同じメジャーバージョンにおける前方及び後方互換性を保持します。
例えば、ある実装がバージョン1.0.1の仕様に準拠しているなら、これは1.xシリーズの仕様にも互換性があります。

>* **`version`** (string, required) must be in [SemVer v2.0.0](http://semver.org/spec/v2.0.0.html) format and specifies the version of the OCF specification with which the container bundle complies. The Open Container spec follows semantic versioning and retains forward and backward compatibility within major versions. For example, if an implementation is compliant with version 1.0.1 of the spec, it is compatible with the complete 1.x series.

*例*

```json
    "version": "0.1.0"
```

## ルートの設定

各コンテナは*root*オブジェクトにより指定された、一つの*ルートファイルシステム*を確実に持ちます:

>Each container has exactly one *root filesystem*, specified in the *root* object:

* **`path`** (string, 必須)は、コンテナのルートファイルシステムのパスを指定します。このパスはマニフェストからの相対パスになります。この相対パスで表された場所には、必ずディレクトリがなくてはなりません。
* **`readonly`** (bool, オプション）は、もし真(true)であればルートファイルシステムはコンテナ内で書き込み不可でなければなりません。指定がない場合は偽(false)です。

>* **`path`** (string, required) Specifies the path to the root filesystem for the container, relative to the path where the manifest is. A directory MUST exist at the relative path declared by the field.
>* **`readonly`** (bool, optional) If true then the root filesystem MUST be read-only inside the container. Defaults to false.

*例*

```json
"root": {
    "path": "rootfs",
    "readonly": true
}
```

## マウントポイント

コンテナ内のマウントポイントを、`mounts`という配列として追加できます。
この配列の個々のレコードは、[ランタイム設定](runtime-config.md#mount-configuration)に設定されている必要があります。
（訳註：例でわかるように、ホスト側のランタイム設定の中で設定したマウントポイントの「名称」と、ここで指定した設定の「名称」を突き合わせて利用するため、このような記述になっています。）
ランタイムは必ずこれらのエントリを、リストされた順序でマウントしなければなりません。

>You can add array of mount points inside container as `mounts`.
>Each record in this array must have configuration in [runtime config](runtime-config.md#mount-configuration).
>The runtime MUST mount entries in the listed order.

* **`name`** (string, 必須) マウントポイントの名称。マウントポイントを見つけるために使う。
* **`path`** (string, 必須) マウント先。コンテナ内の絶対パスです。

>* **`name`** (string, required) Name of mount point. Used for config lookup.
>* **`path`** (string, required) Destination of mount point: path inside container.

*例*

```json
"mounts": [
    {
        "name": "proc",
        "path": "/proc"
    },
    {
        "name": "dev",
        "path": "/dev"
    },
    {
        "name": "devpts",
        "path": "/dev/pts"
    },
    {
        "name": "data",
        "path": "/data"
    }
]
```

## プロセス設定

* **`terminal`** (bool, オプション) は、ターミナルをこのプロセスに接続するかどうかを指定します。デフォルトではfalseです。
* **`cwd`** (string, オプション) は、実行時のワーキングディレクトリを指定します。
* **`env`** (array of strings, オプション) は、プロセスの実行前に設定される環境変数のリストです。この配列の要素は、"KEY=value"の形式で指定され、左辺は [IEEE Std 1003.1-2001](http://pubs.opengroup.org/onlinepubs/009695399/basedefs/xbd_chap08.html) で定義されているように、アルファベットで始まり、数字とアンダースコアを含む文字列です。
* **`args`** (string, 必須) は、コンテナの実行に実行可能バイナリとフラグの配列です。実行可能バイナリは、配列の最初であり、rootfs中の実行可能なパスでなければなりません。実行可能バイナリのパスが絶対パス出ない場合、$PATH環境変数で指定されたパス中から実行可能バイナリを探します。

>* **`terminal`** (bool, optional) specifies whether you want a terminal attached to that process. Defaults to false.
>* **`cwd`** (string, optional) is the working directory that will be set for the executable.
>* **`env`** (array of strings, optional) contains a list of variables that will be set in the process's environment prior to execution. Elements in the array are specified as Strings in the form "KEY=value". The left hand side must consist solely of letters, digits, and underscores `_` as outlined in [IEEE Std 1003.1-2001](http://pubs.opengroup.org/onlinepubs/009695399/basedefs/xbd_chap08.html).
>* **`args`** (string, required) executable to launch and any flags as an array. The executable is the first element and must be available at the given path inside of the rootfs. If the executable path is not an absolute path then the search $PATH is interpreted to find the executable.

プロセスのユーザ指定とは、プラットフォーム特有の構造体で、このコンテナプロセスを実行するときに制御するユーザを指定します。
Linuxベースのシステムでは、ユーザ構造体は以下のフィールドを持ちます。

>The user for the process is a platform-specific structure that allows specific control over which user the process runs as.
>For Linux-based systems the user structure has the following fields:

* **`uid`** (int, 必須) はユーザIDを指定します。
* **`gid`** (int, 必須) はグループIDを指定します。
* **`additionalGids`** (array of ints, オプション) はプロセスに追加されるグループIDを指定します。

>* **`uid`** (int, required) specifies the user id.
>* **`gid`** (int, required) specifies the group id.
>* **`additionalGids`** (array of ints, optional) specifies additional group ids to be added to the process.

*例 (Linux)*

```json
"process": {
    "terminal": true,
    "user": {
        "uid": 1,
        "gid": 1,
        "additionalGids": [5, 6]
    },
    "env": [
        "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
        "TERM=xterm"
    ],
    "cwd": "/root",
    "args": [
        "sh"
    ]
}
```


## ホスト名

* **`hostname`** (string, オプション) はコンテナ内で実行しているプロセスからアクセスできるホスト名です。Linuxではバンドルが新しい[UTS namespace][uts-namespace]を作成した時だけセットできます。

*例*

```json
"hostname": "mrsdalloway"
```

## プラットフォーム固有の設定項目

* **`os`** (string, 必須) はこのイメージが実行されるべきOSの種類を指定します。OSの値はGo言語のドキュメント [`$GOOS`](https://golang.org/doc/install/source#environment) で定義されたものでなければなりません。
* **`arch`** (string, 必須) はこのイメージの中にあるバイナリがコンパイルされた命令セットを示します。この値は、はGo言語のドキュメント [`$GOOS`](https://golang.org/doc/install/source#environment) で定義されたものでなければなりません。


>* **`os`** (string, required) specifies the operating system family this image must run on. Values for os must be in the list specified by the Go Language document for [`$GOOS`](https://golang.org/doc/install/source#environment).
>* **`arch`** (string, required) specifies the instruction set for which the binaries in the image have been compiled. Values for arch must be in the list specified by the Go Language document for [`$GOARCH`](https://golang.org/doc/install/source#environment).

*例*

```json
"platform": {
    "os": "linux",
    "arch": "amd64"
}
```

JSONファイルのplatformセクションの情報は、どのプラットフォーム特有のセクションが有効になるかを解釈するために使われます。
例えば、`os`が`linux`であれば、[Linux特有のスキーマ](config-linux.md)で示されるJSONオブジェクトが`config.json`中の`linux`というキーで探し出せるべきです。

>Interpretation of the platform section of the JSON file is used to find which platform-specific sections may be available in the document.
>For example, if `os` is set to `linux`, then a JSON object conforming to the [Linux-specific schema](config-linux.md) SHOULD be found at the key `linux` in the `config.json`.

[uts-namespace]: http://man7.org/linux/man-pages/man7/namespaces.7.html
