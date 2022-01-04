---
title: "GitHubActionsでテストカバレッジ率を出力してコード品質を可視化する"
emoji: "📈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["php", "GitHubActions", "CI", "PHPUnit"]
published: true
---

Zenn初投稿！！自称コード品質保全委員会です。
GitHubActionsを利用して、以下のようにプルリクにテストカバレッジ率を出力しレビュアーにテストをやってることを可視化したいと思います。
![Test_Coverrage_Summary](/images/github-actions-test-summary/Test_Coverrage_Summary.png)

今自分が関わっているプロジェクトでは、テストコード書かれても実際にテストの実行率は現在どれくらいなのか、ぱっと見では分からないので数字で出すようにしています。世には以下のようなサービスでテストのカバレッジを計測して、可視化するものもありますが、今回やりたいのはもっと簡易的なもの。
GitHubを使用したプロジェクトであれば、どのプロジェクトでも導入できます。
※**Codecov**はprivateリポジトを扱うのは有料

[Codecovを使ってカバレッジを計測する](https://qiita.com/nasum/items/aff9bf09d49b136bbf94)
[Codecov](https://about.codecov.io/)

※当記事ではPHPとPHPUnitを使用していますが、GitHubActionsを使用したBotコメントの方法などは他の言語とかでも流用出来るので見てくださいまし

**この記事で説明すること**

- カバレッジ率のサマリーの出し方
- PullRequestへ自動コメントの投稿

**この記事で説明しないこと**
- UnitTestの説明
- GitHubActionsの初期設定など

**この記事で使用した環境など**
- PHP 8.0.13
- Laravel Framework 6.20.43
- PHPUnit 9.5.10
```shell
root@11c4d9a174c5:/var/www/test# php -v
PHP 8.0.13 (cli) (built: Nov 19 2021 22:11:30) ( NTS )
Copyright (c) The PHP Group
Zend Engine v4.0.13, Copyright (c) Zend Technologies
    with Xdebug v3.1.1, Copyright (c) 2002-2021, by Derick Rethans

root@11c4d9a174c5:/var/www/test# php artisan -V
Laravel Framework 6.20.43

root@11c4d9a174c5:/var/www/test# php vendor/phpunit/phpunit/phpunit --version
PHPUnit 9.5.10 by Sebastian Bergmann and contributors.
```


### 1: UnitTestを実行する

```yml
      - name: Execute Test
        run: |
          php artisan config:clear
          php vendor/phpunit/phpunit/phpunit --coverage-text --colors=never > storage/logs/coverage.log
          cat storage/logs/coverage.log
```

1. `php artisan config:clear`
    おまじない
2. `php vendor/phpunit/phpunit/phpunit --coverage-text --colors=never > storage/logs/coverage.log`
    ここが味噌
3. `cat storage/logs/coverage.log`
    あってもなくても良いけど、一応細かいテスト結果の確認用にコンソールに出力する

phpunitではテスト結果の出力形式を選ぶことができます。
出力形式は以下のようなものがありますが、今回はテキスト形式で出力

```bash
--coverage-clover <file>    Generate code coverage report in Clover XML format
--coverage-cobertura <file> Generate code coverage report in Cobertura XML format
--coverage-crap4j <file>    Generate code coverage report in Crap4J XML format
--coverage-html <dir>       Generate code coverage report in HTML format
--coverage-php <file>       Export PHP_CodeCoverage object to file
# これを使用
--coverage-text=<file>      Generate code coverage report in text format [default: standard output]
--coverage-xml <dir>        Generate code coverage report in PHPUnit XML forma
```

かつ、`--colors=never`を指定して、色出力しないようにします
ローカルで実行する時などは、色を指定した方が見やすいですが、特定の色コードのタグが埋め込まれてしまうので外します。

### 2: UnitTestの失敗理由を出力する
```yml
      - name: Cat Test Result
        run: |
          cat storage/logs/coverage.log
        if: ${{ failure() }}
```

先ほどのテスト実行でテスト結果をログファイルに書き出しました。
それをテストが失敗し、落ちてしまった時も出力するようにします。
これを指定しないと、テストが失敗してコケた場合に何も出力されないので。
```
# GitHubActions側でこの出力だけで終わる
Error: Process completed with exit code 2.
```

`if: ${{ failure() }}`でプロセスが失敗したときだけ実行してくれます。


## 3: UnitTestの結果をフォーマットする
```yml
      - name: Sed Coverage Report
        run: |
          sed -E "s/"$'\E'"\[([0-9]{1,2}(;[0-9]{1,2})*)?m//g" | \
          grep "Code Coverage Report:" -A6 storage/logs/coverage.log | sed -e "s/^ *//" | sed -e "s/ *$//" | sed -e "/^ *$/d" > storage/logs/coverage-summary.log
```
出力したい内容だけ抜き取ります。

### 4: UnitTestの結果を読み込む
```yml
      - name: Read coverage summary
        id: coverage-summary
        uses: juliangruber/read-file-action@v1
        with:
          path: ./src/storage/logs/coverage-summary.log
```
読み込むファイルを指定します。
上記で出力結果をフォーマットしたファイルがあるのでそちらを指定。
`id`はこの後のプロセスで使用するので、必ず指定してください。

### 5: UnitTestの結果(レポート)をGitHubで自動でコメントする

```yml
      - name: Comment Coverage Summary
        uses: marocchino/sticky-pull-request-comment@v2
        with:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          header: coverage-summary
          message: |
            ## Coverage Summary
            ${{ steps.coverage-summary.outputs.content }}
```

messageの部分が実際にコメントする内容。
` ${{ steps.先ほど指定したプロセスID.outputs.content }}`
上記の **3: UnitTestの結果を読み込む** で指定したIDを使用することで、こちらでテスト結果のファイルを出力することが可能です。

このActionの便利なところは、新規にコメントを行うのではなく、コメントを上書きしてくれるところ。履歴も残ります。
![GitHub_Comment_Rivision](/images/github-actions-test-summary/GitHub_Comment_Rivision.png)

なので、テストが実行される度に上書きを行ってくれるので以下のように不必要にコメントが複数生成されることがありません。
![UnitTest_Comment_Collection](/images/github-actions-test-summary/UnitTest_Comment_Collection.png)


### 6: UnitTestの結果をアップロードする
```yml
      - name: Archive code coverage results
        uses: actions/upload-artifact@v2
        with:
          name: code-coverage-report
          path: ./src/storage/logs/coverage-summary.log
```
必要な人だけ向け。
テスト結果をファイル形式でアップロードしておきます。（テスト成果物として？）
実際にアップロードするのであれば、HTML形式の方が見やすいと思うのでそこは適宜。


### UnitTest Workflow 全体図

```yml
name: UnitTest
on: [pull_request]

jobs:
  unit-test:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        ports:
          - 3306:3306
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_PASSWORD: root
          MYSQL_DATABASE: database
          MYSQL_USER: user

        options: >-
          --health-cmd "mysqladmin ping -h localhost"
          --health-interval 20s
          --health-timeout 10s
          --health-retries 10

    env:
      DB_CONNECTION: mysql
      MYSQL_ALLOW_EMPTY_PASSWORD: yes
      DB_HOST: 127.0.0.1
      DB_PORT: 3306
      DB_DATABASE: root
      DB_USERNAME: database
      DB_PASSWORD: user

    defaults:
      run:
        working-directory: ./src

    steps:
      - uses: actions/checkout@v2

      - name: SetUp PHP
        uses: shivammathur/setup-php@master
        with:
          php-version: '8.0'
          extension-csv: mbstring, xdebug

      - name: Cache vendor
        id: cache
        uses: actions/cache@v1
        with:
          path: ./vendor
          key: ${{ runner.os }}-composer-${{ hashFiles('./composer.lock') }}
          restore-keys: |
            ${{ runner.os }}-composer-

      - name: Composer install
        if: steps.cache.outputs.cache-hit != 'true'
        run: composer install -n --prefer-dist

      - name: Composer Dump Autoload
        run: composer dump-autoload -q

      - name: Laravel Settings
        run: |
          cp .env.CI .env
          php artisan key:generate
          chmod -R 777 storage bootstrap/cache        

      - name: Laravel Migrate
        run:
          php artisan migrate:fresh --seed
      
      - name: Execute Test
        run: |
          php artisan config:clear
          php vendor/phpunit/phpunit/phpunit --coverage-text --colors=never > storage/logs/coverage.log
          cat storage/logs/coverage.log

      - name: Cat Test Result
        run: |
          cat storage/logs/coverage.log
        if: ${{ failure() }}

      - name: Sed Coverage Report
        run: |
          sed -E "s/"$'\E'"\[([0-9]{1,2}(;[0-9]{1,2})*)?m//g" | \
          grep "Code Coverage Report:" -A6 storage/logs/coverage.log | sed -e "s/^ *//" | sed -e "s/ *$//" | sed -e "/^ *$/d" > storage/logs/coverage-summary.log

      - name: Read coverage summary
        id: coverage-summary
        uses: juliangruber/read-file-action@v1
        with:
          path: ./src/storage/logs/coverage-summary.log

      - name: Comment Coverage Summary
        uses: marocchino/sticky-pull-request-comment@v2
        with:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          header: coverage-summary
          message: |
            ## Coverage Summary
            ${{ steps.coverage-summary.outputs.content }}

      - name: Archive code coverage results
        uses: actions/upload-artifact@v2
        with:
          name: code-coverage-report
          path: ./src/storage/logs/coverage-summary.log
```

#### 最後に

テストの可視化大事。（自動化するのも大事）
テストが大事なのは言わずもがなですが、「このプロジェクトの前からあるけど、品質ってどうなの？」と偉い人に言われたときに、テスト自体(成果物)とその報告できる数字(成果物のエビデンス)があると、諸々便利だなってことが分かってきました。

偉い人からしたり、プロジェクトから関わっていない人からすると、数字的な指標がないと「動いてるからとりあえずは平気なものなんか？」「品質は大丈夫なんか？」と不安材料が残ります。

ただ、こういったテスト結果を数字で見せられるところに残し続けておけば、偉い人も今後プロジェクトに関わる人も諸々安心材料として動きやすくなるのかなあと。改善あるのみ〜。

**※Follow&❤️してくれると励みになります**
#### 参考記事

- [GitHub Actionsでカバレッジを可視化する
](https://zenn.dev/katzumi/articles/995df5abebc91e312167)