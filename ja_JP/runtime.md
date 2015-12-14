# ランタイムとライフサイクル

## 状態

ランタイムは必ずコンテナのメタデータをディスクに保存し、外部ツールがその情報を使って動けるようにしなければならない。
>Runtime MUST store container metadata on disk so that external tools can consume and act on this information.
ただしこのデータはテンポラリディスクに保存し、システムの再起動時に削除されるようにすることを推奨する。
>It is recommended that this data be stored in a temporary filesystem so that it can be removed on a system reboot.
Linux/Unixベースのシステムでは、このメタデータは`/run/opencontainer/containers`以下に保存されていなければならない。
>On Linux/Unix based systems the metadata MUST be stored under `/run/opencontainer/containers`.
Linux/Unixベースでないシステムについては、メタデータの場所は今のところ未定義である。
>For non-Linux/Unix based systems the location of the root metadata directory is currently undefined.
このディレクトリにはコンテナごとに1つのディレクトリがなければならない。このディレクトリの名前はコンテナのIDでなければならない。
>Within that directory there MUST be one directory for each container created, where the name of the directory MUST be the ID of the container.
例えば、`173975398351`というIDのLinuxコンテナに対応するディレクトリは、`/run/opencontainer/containers/173975398351`となる。
>For example: for a Linux container with an ID of `173975398351`, there will be a corresponding directory: `/run/opencontainer/containers/173975398351`.

各コンテナのディレクトリには、そのコンテナのランタイム状態を表す`state.json`というJSONファイルを配置しなければならない。例：`/run/opencontainer/containers/173975398351/state.json`
>Within each container's directory, there MUST be a JSON encoded file called `state.json` that contains the runtime state of the container.
For example: `/run/opencontainer/containers/173975398351/state.json`.

この`state.json`は、下記のすべてのプロパティを含んでいなければならない。
>The `state.json` file MUST contain all of the following properties:

* **`version`**: このstringプロパティは、このコンテナが作られた時点でのOCFのバージョンである。
>* **`version`**: (string) is the OCF specification version used when creating the container.
* **`id`**: このstringプロパティは、コンテナのIDを示す。
このIDはホスト上のすべてのコンテナに対して一意なものでなければならないが、ホスト間で一意である必要はない。
このIDは、実行状態の中で与えられる。なぜならフックはステートをペイロードとして実行されるからである。このことは、これらのフックがクリーンアップあるいは削除ロジックをランタイムがコンテナのステートを壊した後でも実行できるようにしている。
> * **`id`**: (string) is the container's ID.
>This MUST be unique across all containers on this host.
>There is no requirement that it be unique across hosts.
>The ID is provided in the state because hooks will be executed with the state as the payload.
>This allows the hooks to perform cleanup and teardown logic after the runtime destroys its own state.

* **`pid`**: (int) is the ID of the main process within the container, as seen by the host.
* **`bundlePath`**: (string) is the absolute path to the container's bundle directory.
This is provided so that consumers can find the container's configuration and root filesystem on the host.

*Example*

```json
{
    "version": "0.2.0",
    "id": "oc-container",
    "pid": 4422,
    "bundlePath": "/containers/redis"
}
```

## Lifecycle
The lifecycle describes the timeline of events that happen from when a container is created to when it ceases to exist.

1. OCI compliant runtime is invoked by passing the bundle path as argument.
2. The container's runtime environment is created according to the configuration in config.json.
   Any updates to config.json after container is running do not affect the container.
3. The container's state.json file is written to the filesystem under /run/opencontainer/<runtime>/containers/<id>/.
4. The prestart hooks are invoked by the runtime.
   If any prestart hook fails, then the container is stopped and the lifecycle continues at step 8.
5. The user specified process is executed in the container.
6. The poststart hooks are invoked by the runtime.
   If any poststart hook fails, then the container is stopped and the lifecycle continues at step 8.
7. Additional actions such as pausing the container, resuming the container or signaling the container may be performed using the runtime interface.
   The container could also error out or crash.
8. The container is destroyed by undoing the steps performed during create phase (step 2).
9. The poststop hooks are invoked by the runtime and errors, if any, are logged.
10. The state.json file associated with the container is removed and the return code of the container's user specified process is returned or logged.

Note: The lifecycle is a WIP and it will evolve as we have more use cases and more information on the viability of a separate create phase.

## Hooks

See [runtime configuration for hooks](./runtime-config.md)
