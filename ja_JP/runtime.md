# ランタイムとライフサイクル

## 状態

ランタイムは外部ツールがその情報を使って動けるよう、必ずコンテナのメタデータをディスクに保存しなければなりません。
ただしこのデータはテンポラリディスクに保存し、システムの再起動時に削除されるようにすることを推奨します。
Linux/Unixベースのシステムでは、このメタデータは`/run/opencontainer/containers`以下に保存しなければなりません。
Linux/Unixベースでないシステムについては、メタデータの場所は今のところ未定義です。
このディレクトリには、コンテナごとにコンテナのIDを名前にしたディレクトリを作らなければなりません。
例えば、`173975398351`というIDのLinuxコンテナに対応するディレクトリは、`/run/opencontainer/containers/173975398351`となります。
各コンテナのディレクトリには、そのコンテナの実行状態を表す`state.json`というJSONファイルを配置する必要があります。例：`/run/opencontainer/containers/173975398351/state.json`

>Runtime MUST store container metadata on disk so that external tools can consume and act on this information.
>It is recommended that this data be stored in a temporary filesystem so that it can be removed on a system reboot.
>On Linux/Unix based systems the metadata MUST be stored under `/run/opencontainer/containers`.
>For non-Linux/Unix based systems the location of the root metadata directory is currently undefined.
>Within that directory there MUST be one directory for each container created, where the name of the directory MUST be the ID of the container.
>For example: for a Linux container with an ID of `173975398351`, there will be a corresponding directory: `/run/opencontainer/containers/173975398351`.

>Within each container's directory, there MUST be a JSON encoded file called `state.json` that contains the runtime state of the container.
For example: `/run/opencontainer/containers/173975398351/state.json`.

この`state.json`には、下記の全てのプロパティが必要です。

>The `state.json` file MUST contain all of the following properties:

* **`version`**: (string)は、このコンテナが作られた時点で使用したOCF仕様のバージョンです。
* **`id`**: このstringプロパティは、コンテナのIDを示します。
このIDはホスト上のすべてのコンテナに対して一意なものである必要がありますが、ホスト間で一意である必要はありません。
このIDは、実行状態に対して与えられます。なぜならフックは実行状態を対象にして実行されるためです。このことにより、フックがクリーンアップあるいは削除ロジックをランタイムがコンテナ自体を終了した後でも実行できるようにしています。

>* **`version`**: (string) is the OCF specification version used when creating the container.
> * **`id`**: (string) is the container's ID.
>This MUST be unique across all containers on this host.
>There is no requirement that it be unique across hosts.
>The ID is provided in the state because hooks will be executed with the state as the payload.
>This allows the hooks to perform cleanup and teardown logic after the runtime destroys its own state.

* **`pid`**: (int)は、ホスト上でのコンテナのメインプロセスのIDです。
* **`bundlePath`**: (string)は、コンテナのバンドルディレクトリの絶対パスです。
この情報により、ホスト上の他のプロセスがコンテナの設定とルートファイルシステムを見つけられるようにします。

>* **`pid`**: (int) is the ID of the main process within the container, as seen by the host.
>* **`bundlePath`**: (string) is the absolute path to the container's bundle directory.
>This is provided so that consumers can find the container's configuration and root filesystem on the host.

*例*

```json
{
    "version": "0.2.0",
    "id": "oc-container",
    "pid": 4422,
    "bundlePath": "/containers/redis"
}
```

## ライフサイクル

このライフサイクルはコンテナが生まれてから死ぬまでに発生したイベントの流れを表しています。

>The lifecycle describes the timeline of events that happen from when a container is created to when it ceases to exist.

1. OCI準拠のランタイムにバンドルのパスを引数に渡して実行する
2. config.jsonの設定に従い、コンテナの実行環境が作られる。この後config.jsonに加えられた変更は、実行中のコンテナには影響しない。
3. コンテナのstate.jsonファイルが /run/opencontainer/<runtime>/containers/<id>/以下に作成される。
4. ランタイムによってprestartフックが実行される。
   prestartフックが失敗した場合、コンテナは停止し、ライフサイクルのステップ8に飛ぶ。
5. ユーザが指定したプロセスがコンテナ上で実行される。
6. poststartフックがランタイムによって実行される。
   poststartフックが失敗した場合、コンテナは停止し、ライフサイクルのステップ8に飛ぶ。
7. コンテナの一時停止やレジューム、シグナル送信などの追加のアクションがランタイムのインタフェースから実行される。
   コンテナはまた、エラーで終了したり、クラッシュすることもある。
8. コンテナはステップ2の生成フェーズで行われた処理を取り消すことで削除される。
9. poststopフックがランタイムにより実行され、エラーがあれば記録される。
10. コンテナのstate.jsonファイルが削除され、コンテナ内のユーザが指定したプロセスの戻り値が返される、あるいは記録される。

>1. OCI compliant runtime is invoked by passing the bundle path as argument.
>2. The container's runtime environment is created according to the configuration in config.json.
>   Any updates to config.json after container is running do not affect the container.
>3. The container's state.json file is written to the filesystem under /run/opencontainer/<runtime>/containers/<id>/.
>4. The prestart hooks are invoked by the runtime.
>   If any prestart hook fails, then the container is stopped and the lifecycle continues at step 8.
>5. The user specified process is executed in the container.
>6. The poststart hooks are invoked by the runtime.
>   If any poststart hook fails, then the container is stopped and the lifecycle continues at step 8.
>7. Additional actions such as pausing the container, resuming the container or signaling the container may be performed using the runtime interface.
>   The container could also error out or crash.
>8. The container is destroyed by undoing the steps performed during create phase (step 2).
>9. The poststop hooks are invoked by the runtime and errors, if any, are logged.
>10. The state.json file associated with the container is removed and the return code of the container's user specified process is returned or logged.

注意: このライフサイクルは現在も策定中で、ユースケースが増えたり、これ以外の生成フェーズについての情報が出てくれば、更新されます。

>Note: The lifecycle is a WIP and it will evolve as we have more use cases and more information on the viability of a separate create phase.

## フック

[フックのランタイム設定](./runtime-config.md)を参照してください。

> See [runtime configuration for hooks](./runtime-config.md)
