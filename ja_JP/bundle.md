# ファイルシステムバンドル

## コンテナ形式

この節では、あるコンテナを一定の方法でまとめたファイルの集合である「ファイルシステムバンドル」として固めるフォーマットを定義します。また、どのOCF準拠ランタイムも標準的なオペレーションをそれに対して実行できるデータとメタデータを含んでいるように、バンドルを定義します。
>This section defines a format for encoding a container as a *filesystem bundle* - a set of files organized in a certain way, and containing all the necessary data and metadata for any compliant runtime to perform all standard operations against it.

「バンドル」を似たような意味で使っている例としては、[OS X アプリケーションバンドル](http://en.wikipedia.org/wiki/Bundle_%28OS_X%29)が挙げられます。
>See also [OS X application bundles](http://en.wikipedia.org/wiki/Bundle_%28OS_X%29) for a similar use of the term *bundle*.

バンドルの定義をするにあたっては、どのようにコンテナとコンテナの設定データをローカルファイルシステムに保存し、OCF準拠ランタイムで使えるようにするかのみを考えます。
>The definition of a bundle is only concerned with how a container, and its configuration data, are stored on a local file system so that it can be consumed by a compliant runtime.

「標準コンテナ」はコンテナをロードし実行するのに必要な情報をすべて含んでいます。これは下記に示す3つの情報がローカルファイルシステムの同じディレクトリになければならないことを意味しています。
>A Standard Container bundle contains all the information needed to load and run a container.
>This includes the following three artifacts which MUST all reside in the same directory on the local filesystem:

1. `config.json` : ホストによらない設定情報
このファイルは`config.json`という名前でなければなりません。このファイルには、動作させるホストに依存しないアプリケーション固有の情報、例えばセキュリティのパーミッションや、環境変数、実行時の引数などを記述します。
バンドルをディストリビューション向けにパッケージ化する場合には、このファイルを必ず含みます。より詳細には[`config.json`](config.md)を参照してください。
>1. `config.json` : contains host-independent configuration data.
>This REQUIRED file, which MUST be named `config.json`, contains settings that are host-independent and application-specific such as security permissions, environment variables and arguments.
>When the bundle is packaged up for distribution, this file MUST be included.
See [`config.json`](config.md) for more details.

2. `runtime.json` : ホスト特有の設定情報
このファイル`runtime.json`という名前でなければなりません。このファイルには、マウントソースやフックといったホスト特有の情報を記述します。
このファイルの目的は、バンドルが他のランタイムに移植できる最小単位となれるようにすることと、`runtime.json`が一度定義されれば、それに基づき同じアプリケーションが同じように動作できるようにすることです。
バンドルをディストリビューション向けにパッケージ化する場合には、このファイルを含まないようにする必要があります。より詳細には [`runtime.json`](runtime-config.md)を参照してください。
>2. `runtime.json` : contains host-specific configuration data.
>This REQUIRED file, which MUST be named `runtime.json`, contains settings that are host-specific such as mount sources and hooks.
>The goal is that the bundle can be moved as a unit to another runtime and run the same application once a host-specific `runtime.json` is defined.
>When the bundle is packaged up for distribution, this file MUST NOT be included.
>See [`runtime.json`](runtime-config.md) for more details.

3. コンテナのルートファイルシステムを表すディレクトリ
このディレクトリの名前は任意ですが、通例の`rootfs`という名前を使うべきです。
バンドルをディストリビューション向けにパッケージ化する場合には、このディレクトリを含むようにします。
このディレクトリは、`config.json`ファイルにより指定されなければなりません。
>3. A directory representing the root filesystem of the container.
>While the name of this REQUIRED directory may be arbitrary, users should consider using a conventional name, such as `rootfs`.
>When the bundle is packaged up for distribution, this directory MUST be included.
>This directory MUST be referenced from within the `config.json` file.

この3つの情報は、ローカルファイルシステムの一つのディレクトリに存在しなければならず、またそのディレクトリ自体はバンドルの一部にしてはなりません。
言い換えると、ひとつのバンドルのtarアーカイブはこれらの情報をアーカイブのrootに持っており、トップレベルのディレクトリは含んでいないということになります。
>While these three artifacts MUST all be present in a single directory on the local filesytem, that directory itself is not part of the bundle.
>In other words, a tar archive of a *bundle* will have these artifacts at the root of the archive, not nested within a top-level directory.
