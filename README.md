# docker_study

## Dockerの基礎

Dockerは2013年に誕生。

コンテナ型仮想化技術。OSのリソースを隔離し、仮想OSにする。この仮想OSをコンテナと呼ぶ。
仮想化ソフトウェアを利用し、演算によりハードウェアを再現してゲストOSを作りだす方式を、ホストOS型の仮想化という。

コンテナは軽量に作成、利用、破棄できるのが重要な特徴の1つ。

runCランタイム経由でKernelを操作する（かつてはLXC経由）。

Dockerは、コンテナにアプリケーションと実行環境（OS）を同梱してデプロイするスタイル。

MobyプロジェクトでDockerが開発され、Docker CE/EEへ反映されていく。
最新の開発状況を知りたい場合は、以下にアクセス。
https://github.com/moby/moby

LinuxKitの開発により、WindowsやMacでdockerを利用する際、VirtualBoxを利用する必要がなくなった
（dockerdは、Linux上でしか動作しない）。
AWS等でもLinuxKitは利用されている。
macOS上では、hyperkitを利用している。

IaC, Immutable Infrastructure（冪等性）の考え方を再現できるのがdocker。

Dockerはプロダクションでも利用されている。
わずかなオーバーヘッドの割に、スケールアウトのしやすさが強み。
例えばCAのAbema系サービス。ポケモンGOもそう。
ポータビリティの利点を最大限活用するために、積極的に本番環境でも使っていこう。

Docker for Windowsは環境構築がうまく行かないケースもある。
その場合は、Docker Toolboxがおすすめ。リソース面では劣るが、ゲストOS上で起動できる。
VirtualBoxを利用する（仮想化オプションはONにする）。Hyper-Vとは同時利用できないので、注意。

Dockerのバージョン番号は、20XX年Y月に出たバージョンを ver.XX.Y と表す。

## Dockerコンテナのデプロイ

ImageはContainerの設定。
Containerはファイルシステムとアプリケーション。

`ENTRYPOINT`を使えば、最初のコマンドを設定できる。

`docker run` は、 `docker container run`の略で、
`docker build` は、 `docker image build`の略。
近年では、省略しないほうが良いとされている。

`--pull=true`をつけてビルドすることで、キャッシュを利用せずにFROMのイメージを落としてこれる。

実際の運用では`latest`のタグのイメージを利用することはほとんどなく、タグ名を直接指定する。

`docker search`でリポジトリの検索をかけられる。
`mysql`などはオーナー名がついていないが、これは公式イメージだからである。
公式イメージには、`library/`という名前空間が切られており、`library/mysql`が正式名称。

`--name`オプションつきでrunすると、containerに名前をつけられる。
開発用途ではよく使うが、コンテナが頻繁に破棄・生成を繰り返す本番環境ではあまり使わない。

`--rm`オプションをつけてrunすると、終了時にcontainerが破棄される。

`-i`をつけると、標準入力をつなぎ続けることができ、`-t`をつけると疑似端末を起動できる。

`docker container ls -q` で、idの一覧を取得できる。よく使う。

`docker image --help`みたいにヘルプがサブコマンドについても確認可能。

`docker container ls --filter`で、検索ができる。

`docker container rm -f`で、停止と破棄を同時に行える。

```
echo '{"version":100}' | docker container run -i --rm gihyodocker/jq:1.5 '.version'
 100
```
みたいにパイプで入力を渡せる。`-i`をつけるのがポイント。

`docker container run --rm`オプションは、`--name`オプションと同時に利用されることが多い。
1つのコンテナ名に対して、複数のコンテナを立ち上げることができないため。

`docker container cp`で、コンテナ間・ホスト-コンテナ間でのファイルのコピーができる。

`docker image/container prune`で不要なデータを一括削除できる。
`docker system prune`で、コンテナやボリューム、イメージやネットワークを一括削除できる。

Dockerコンテナ = 単一のアプリケーション と考える。

docker-composeのlinksを利用すれば、コンテナ同士を簡単に接続できる。

## 実用的なコンテナのデプロイ

### 1コンテナ = 1プロセス は妥当？

Docker黎明期には一部でルール化されていた。
必要以上に複雑になる場合があるので、絶対条件としない認識で良い。

公式ドキュメントのBest Practices for writing Dockerfileによれば、
「1つのコンテナでは1つの関心ごと（concern）に集中すべき」とある。

### Dockerフレンドリーなアプリケーション

Dockerコンテナとして実行されるアプリケーションの挙動を制御するには以下の様な方法がある。

- 実行時引数
  - CMDやENTRYPOINTが煩雑になりがち
- 設定ファイル
  - 非常にポピュラーで、コンテナの流行前はオーソドックスな方法だった
- アプリケーションの挙動を環境変数で制御
  - 非常に有用。イメージの再ビルドが不要。
- 設定ファイルに環境変数を埋め込む
  - 設定ファイルの良さと環境変数の良さを両取りできるが、対応していないアプリケーションも多い


### Data Volumesコンテナ

Container上のデータは、Containerが破棄されると消えてしまう。
永続化のため、-vオプションを利用したりするが、ホストと密結合になって危険。

その際、データ保持の目的だけのコンテナを用意したりする。

`docker run --volumes-from`オプションで、別コンテナのVolumeを利用できる。
VOLUMEを設定していると、そのコンテナ用にvolume用のディレクトリが確保される（仮想マシン上の`/var/lib/docker/*`に作られる。macOS上ではないので注意）


### 複数コンテナの起動

`docker compose`でセットで起動したコンテナ群は、composeのデフォルトのネットワークに乗り、コンテナ名で名前解決ができる。

### docker service

```
docker container exec -it manager \
docker service create --replicas 1 --publish 8000:8080 --name echo registry:5000/example/echo:latest
```

```
docker container exec -it manager docker service scale echo=6
```

Serviceは1つのアプリケーションイメージしか扱うことができない。
複数のServiceを扱うため、Stackを利用する。


## Swarmによる実践的なアプリケーション構築


## Kubernetes入門

Google主導で開発が勧められたツール。
2017にDockerに公式に統合・サポート開始。

Dockerコンテナを扱うという点では、KubernetesはSwarmとほぼ同じ立ち位置。
Compose/Stack/Swarmの機能を統合しつつ、より高度に管理できるものと考える。

クラウドで動作させているKubernetesであれば、GCPならGCE、AWSならEC2がnodeになる。

PodはDockerコンテナの集合。
PodごとにIPアドレスが割り当てられ、Pod内のコンテナ同士は`localhost`経由で通信が可能。
他のPodのIPアドレスを叩けばアクセスできる。

replicasetでpodの情報を定義することで、複製が可能。

実運用では、Deploymentのマニフェストファイルを扱う運用にすることがほとんど。

Deploymentを利用すると、コンテナ情報を変更した時に、replicasetのrevisionが変わる。
rollbackができる点が良い。

Serviceでセレクタによるcontainerへのルーティングが可能。
serviceの名前解決が便利なのでよく使う。

ServiceはKubenetesクラスタからしかアクセスできない。
公開するためにIngressなどを用いる。

## Kubenetesのデプロイ・クラスタ構成

...to be continued