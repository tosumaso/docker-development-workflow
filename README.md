## エラー: Failed to initialize watch plugin "node_modules/jest-watch-typeahead/filename.js
  - 原因: モジュールのjest-watch-typeheadが現バージョンだとESMで書かれておりエラーがでる
  - 解決策: ```npm i -D --exact jest-watch-typeahead@0.6.5``` : ESMモジュールと競合しないバージョンのjestを読み込む

# Docker、GithubActions、Elastic Beanstalkを使ったアプリケーションの開発ワークフロー

## 下準備
1. ローカルのnpm

## 開発環境と本番環境
- 開発環境と本番環境でDockerFileを分けた方がよい
- 開発環境ではDockerfile.dev、本番環境ではそのままDockerFileと名付けることで差別化できる
- `Dockerfile`以外の名前のファイルを読み込み時は`docker build -f ファイル名 .`のように`-f`をつける

## Dockerfile読み込み時間の短縮
- コンテナにコピーされたローカルの`node_modules`ファイルとコンテナでインストールされた`node_modules`の2つが存在し、容量を食う :ローカルの`node_modules`ファイルを削除する

# DockerVolume
  - ボリューム: 外部のデータ保存領域。コンテナに保存したデータはコンテナが破棄されると同時に削除されるため。
  - バインドマウント: コンテナ内のファイルがコンテナ外のファイルを参照し続けるように設定できる

 - マウント: ホスト側のリソースをコンテナの中から使えるようにすること。読み書き可能

例(バインドマウント): 
  - ローカル環境でコードに変更を加えた場合、読み込ませるために毎回`docker build`、`docker run`する必要がある。
  - docker内にコピーしたファイルが常にローカルのファイルの内容を参照するように設定することで手間を省略できる
```docker run -p 3000:3000 -v /app/node_modules -v ${pwd}:/app imageId```

- `-p 外部からのポート:docker内のポート`: ポートの紐付け
- `-v docker内のパス`: 指定したパスのリソースはローカルのリソースをマウント、参照しない。`npm install`で作成されたnode_modulesフォルダはローカルに存在しないため、バインドさせるとエラーになる.バインドマウントはローカル、コンテナ内に対応するリソースがある場合のみ可能。
- `-v ローカルのパス:docker内のパス`: ローカルのリソースとdocker内のリソースをマウントさせる。`:`が必要。

問題点: コマンドが複雑
改善策: docker-composeで簡単にdocker runを実行する

```
version: '3'
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - /app/node_modules
      - .:/app
```

```docker-compose -f docker-compose.ymlの別名 up (--build)```: docker-composeも`-f`をつけると指定した名前のcomposeファイルを検索し、実行する

```
build:
  context: パス
  dockerfile: dockerファイル名
```
dockerファイルのパス、名前が違う場合はオプションをつけて設定する

```
volumes:
  - コンテナ内のパス
  - ローカルのパス:コンテナ内のパス
```
1行目のパスはローカルにマウントしない設定、2行目のパスはローカルにマウントする設定

## テスト
  - ```docker run -it cdcfc83005ab npm run test```: コンテナを立ち上げて`npm run test`を実行する
  - コンテナが起動中の場合、```docker exec -it コンテナId or 名前 sh```でコンテナにアクセスして`npm run test`を実行する

## 実行環境
Webサーバーにデプロイする場合

例: Dockerfile
```
FROM node:14-alpine as builder
WORKDIR '/app'
COPY package.json .
RUN npm install
COPY . .
RUN npm run build

FROM nginx
COPY --from=builder /app/build /usr/share/nginx/html
```

ベースイメージを２つ指定している。alpineがインストールされたビルドフェーズとnginxがインストールされた実行フェーズの２つに分けている
2つのフェーズに分けることでnpm installされたモジュールの依存関係などの不要な生成物をカットできる。依存関係はビルドするときに必要だけどデプロイ時は必要ないため。

- `FROM イメージ as 別名`: イメージに別名を付けられる
- `RUN npm run build`: プロジェクトをビルドし、デプロイ用に1つの圧縮ファイルを作成する
- `FROM nginx`: 2つ目のベースイメージ
- `COPY --from=別のフェーズのイメージ名 コピー元 コピー先`: 1つ目のイメージで作成されたファイル、フォルダを別のパスにコピー。`/usr/share/nginx/html`はnginxがwebサイトをデプロイするときに配置される場所。1つ目のイメージで作成された`npm run build`の成果物だけが移される。

## CIツール
githubにpushしたコードのテスト、デプロイの自動化が可能。CircleCI,TravisCI,GitHubActionなど。

## AWSにデプロイ

### Elastic Beanstalk
VPC、サブネット、負荷分散、EC2、RDS、サーバー、言語のインストールなどを自動で行ってくれるAWSのサービス

## Github Actions

1. Githubリポジトリの`Settings` -> `Actions` -> `New repository secret`からGithub用の環境変数を設定
  - プロジェクト内で変数に参照する方法: ```${{ secrets.変数名 }}```
2. `AWS_ACCESS_KEY(IAMユーザーの認証情報)`, `AWS_SECRET_KEY(IAMユーザーの暗号キー)`, `DOCKER_USERNAME`, `DOCKER_PASSWORD`をGithubの環境変数として登録
3. プロジェクトのルートディレクトリに`.github`フォルダを作成
  - その中に`workflows`フォルダを作成
    - その中に`deploy.yml or yaml(任意の名前)`ファイルを作成
4. deploy.yamlファイルの編集

```
name: Deploy Frontend
on:
  push:
    branches:
      - master
 
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - run: docker login -u ${{ secrets.DOCKER_USERNAME }} -p ${{ secrets.DOCKER_PASSWORD }}
      - run: docker build -t cygnetops/react-test -f Dockerfile.dev .
      - run: docker run -e CI=true cygnetops/react-test npm test
 
      - name: Generate deployment package
        run: zip -r deploy.zip . -x '*.git*'
 
      - name: Deploy to EB
        uses: einaregilsson/beanstalk-deploy@v18
        with:
          aws_access_key: ${{ secrets.AWS_ACCESS_KEY }}
          aws_secret_key: ${{ secrets.AWS_SECRET_KEY }}
          application_name: docker-react
          environment_name: Dockerreact-env
          existing_bucket_name: elasticbeanstalk-ap-northeast-1-401084975383
          region: ap-northeast-1
          version_label: ${{ github.sha }}
          deployment_package: deploy.zip
```

`name`: Githubに表示させるワークフローの名前。  
`on`: 自動テスト、デプロイを発動させる条件  
`push`: Githubにpush時発動  
`branches`: 特定のブランチにpushしたときに発動

`jobs`: 複数のステップで構成された処理。  
`build`: `job`の名前  
`runs-on`: Github actionsが動作する環境を指定  
`steps`: `job`に紐づく動作の集合体  
`steps.uses`: `job`の`step`の一部として処理される動作。アクション。  
`actions/checkout@v2`: Github actionsが提供している機能。gitでcheckoutする。  
`steps.run`: コマンドを実行する。`name`を指定しないと実行するコマンドがstep名になる。  
`steps.uses: einaregilsson/beanstalk-deploy@v18`: AWSのBeanstalkを使ってデプロイする  
`steps.with`: `action`で定義されているパラメータの指定。

aws_access_key, aws_secret_key, application_name: docker-react, environment_nameを随時変更する


## jobとstepの違い
job: 並行実行される
step: 直列実行される