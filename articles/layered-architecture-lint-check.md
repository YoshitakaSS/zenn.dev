---
title: "レイヤードアーキテクチャのコード品質を守りたい！"
emoji: "🛡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CI", "PHP", "Laravel", "GitHubActions"]
published: false
---

自称コード品質保全委員会です！
いきなりですが、最近流行っているレイヤードアークテクチャ、クリーンアーキテクチャ、DDD..etc。
色々な知見を活かしてせっかく導入したは良いけど、その品質どうやって担保していますか？

- もし導入した人がいなくなってしまったら...
- 特にレイヤーを意識しない人が開発をしてしまったら...
- 他の仕事が忙しくて、レビューもされない状況になってしまったら...

などなどコードの品質自体が壊れる理由は数多あります。人間でのチェックは限界です。
であれば、レイヤー間の制約を定義してCIに組み込んでしまえ！っていうのが本記事のテーマです。


**この記事で説明すること**
- レイヤードアーキテクチャのLintCheck/静的解析をするツール紹介&基本的な記法など

**この記事で説明しないこと**
- レイヤードアーキテクチャの細かい説明/論争

**環境など**
- PHP 8.0.13
- Laravel Framework 6.20.43

```bash
root@11c4d9a174c5:/var/www/test# php -v
PHP 8.0.13 (cli) (built: Nov 19 2021 22:11:30) ( NTS )
Copyright (c) The PHP Group
Zend Engine v4.0.13, Copyright (c) Zend Technologies
    with Xdebug v3.1.1, Copyright (c) 2002-2021, by Derick Rethans

root@11c4d9a174c5:/var/www/test# php artisan -V
Laravel Framework 6.20.43
```

## レイヤードアーキテクチャの品質を守る「Deptrac」

紹介するツールはPHP専用となっています。誰か他の言語でも使えるように拡張してくれ。
https://github.com/qossmic/deptrac?ref=bestofphp.com

### Deptracとは？
>本文）Deptrac is a static code analysis tool for PHP that helps you communicate, visualize and enforce architectural decisions in your projects. You can freely define your architectural layers over classes and which rules should apply to them.
For example, you can use Deptrac to ensure that bundles/modules/extensions in your project are truly independent of each other to make them easier to reuse.
Deptrac can be used in a CI pipeline to make sure a pull request does not violate any of the architectural rules you defined. With the optional Graphviz formatter you can visualize your layers, rules and violations.

>訳）Deptrac は PHP 用の静的コード解析ツールで、プロジェクトにおけるアーキテクチャの決定を伝達、可視化、強制するのに役立ちます。クラスに対するアーキテクチャの階層を自由に定義し、どのルールを適用させるかを決めることができます。
例えば、Deptracを使用して、プロジェクト内のバンドル/モジュール/拡張機能が互いに真に独立していることを確認し、再利用を容易にすることができます。
DeptracはCIパイプラインで使用され、プルリクエストが定義したアーキテクチャルールに違反していないことを確認することができます。オプションのGraphviz formatterを使えば、レイヤー、ルール、違反を可視化することができます。

**クラスに対するアーキテクチャの階層を自由に定義し、どのルールを適用させるかを決めることができます**（※ここが重要）

なので、自分のプロジェクトに合わせて色々なカスタマイズすることができます。
「Aのプロジェクトだとヘキサゴナルアーキテクチャ」「Bのプロジェクトだとオニオンアーキテクチャ」でやりたいみたいな事も設定上可能です。触って見た感じ結構柔軟に設定できます。

### Deptracの導入方法

公式のリポジトリでは4種類存在します。本記事では、Composerで導入します。

- [PHAR](https://github.com/qossmic/deptrac?ref=bestofphp.com#phar)
- [PHIVE](https://github.com/qossmic/deptrac?ref=bestofphp.com#phive)
- Composer
- [Optional Dependency: Graphviz](https://github.com/qossmic/deptrac?ref=bestofphp.com#optional-dependency-graphviz)

```bash
composer require qossmic/deptrac-shim
```

### Deptracの実行方法

設定したいディテクトりに`depfile.yml`を作成します。
srcディレクトリ配下にLaravel標準の諸々が入ってると仮定して作成。
```bash
touch ./src/depfile.yml
```

```bash
/src
├── README.md
├── app
├── artisan
├── bootstrap
├── composer.json
├── composer.lock
├── config
├── database
├── depfile.yaml // ここに作るよ
├── package.json
├── phpcs.xml
├── phpunit.xml
├── public
├── routes
├── storage
├── tests
└── vendor
```

公式サイトを元に、以下な中身で`depfile.yml`を作成する
```yml
# depfile.yaml
paths:
  - ./app
exclude_files:
  - '#.*test.*#'
layers:
  - name: Controller
    collectors:
      - type: className
        regex: .*Controller.*
  - name: Service
    collectors:
      - type: className
        regex: .*Service.*
  - name: Repository
    collectors:
      - type: className
        regex: .*Repository.*
ruleset:
  Controller:
    - Service
  Service:
    - Repository
  Repository: ~
```

あとは以下のように実行するだけ。めちゃ便利。
レポート出力を行い、依存関係を無視した実装があると`Violations`の数字が増えます。

```bash
php vendor/bin/deptrac analyse

477/477 [▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓] 100%

 ----------- ---------------------------------------------------------------------------------------------------------------------------------------------------------------- 
  Reason      Controller                                                                                                                                                          
 ----------- ---------------------------------------------------------------------------------------------------------------------------------------------------------------- 
  Violation   App\Http\Controlller\SampleController must not depend on \App\Infrastructre\SampleRepository (Infrasturcutre

-------------------- ----- 
Report                    
-------------------- ----- 
Violations           1  
Skipped violations   0    
Uncovered            65   
Allowed              888  
Warnings             1    
Errors               0    
-------------------- ----- 
```

#### DeptracのReport内容
:::message
- **Violations:**
    - 依存関係が守られていないもの
- **Skipped violations**
    - 依存関係を一旦無視したもの
- **Uncovered:**
    - 依存関係が定義されていないもの
- **Allowed:**
    - 依存関係チェック済みなもの
- **Warnings:**
    - 依存関係の指定がミスってるもの（2つ以上のレイヤーから依存している場合など
- **Errors:**
    - 記法ミスってんぞ
:::

### Deptracの記法ルール

基本的には以下の2種類の書き方を各々のプロジェクトに合わせる必要がある。

**`paths`**, **`exclude_files`** については特に記法も何もないので、**`layers`** と **`ruleset`** だけ記述します。

:::message

- **`paths`**
    - ルールを適用するディレクトリを指定
- **`exclude_files`**
    - ルールを除外するファイルを指定
- **`layer`**
    - レイヤー名の定義
    UseCase/Application層、Domain層、Infra層といったレイヤー名の定義を行う
- **`ruleset`**
    - 依存方向の定義
:::

### `layers`の指定

```yml
layers:
  - name: Controller            # レイヤー名
    collectors:                 # 複数指定可能
      - type: className         # ※後述します
        regex: .*Controller.*   # クラス名を正規表現で指定
```

各種レイヤーの指定を行います。レイヤーの記入する順番は依存性の矢印（上から下）に揃える必要性は記法的にはないですが、可読性のために揃えた方がいいです。
サンプルの例だと以下の3つのレイヤーで構成されており、**依存性の方向は上から下ですね。**

- Controller
- Serivce
- Repository

DDD×Laravalとかの導入であれば、以下みたいなレイヤーになるかな。

- Controller / Presenter / Handler / Adapter / Action
- UseCase / Applicatoin
- Domain
- Infrastructure

`collectors`は複数指定可能となっており例えば「Laravelを使用していてControllerとRequestをUI層としたいんだよな〜。」となった場合、以下のような指定が可能です。
`type`にいくつか種類があり、様々な指定が可能です。（指定内容は後述）

```yml
layers:
  - name: UI
    collectors:
      - type: className
        regex: .\\Http\\Controller\\.*
      - type: className
        regex: .\\Http\\Requests\\.*
```

#### `layers`の指定1: `className Collector`

>引用：className コレクターは、その完全修飾名を簡略化した正規表現にマッチさせることでクラスを収集することができます。一致するクラスは、割り当てられたレイヤーに追加されます。

上記のサンプルで使ったやつ。個人的にこれが一番使う。（というかこれだけで事足りる

```yml
layers:
  - name: Controller            
    collectors:                 
      - type: className         
        regex: .*Controller.*   
```

####  `layers`の指定2: `classNameRegex Collector`

>引用：classNameRegexコレクターは、クラスの完全修飾名を正規表現にマッチさせることでクラスを収集することができます。マッチしたクラスは、割り当てられたレイヤーに追加されます。

正規表現でクラス名を指定できる。ぶっちゃけ`type: className`との違いがわからん。（上でも正規表現使えるし）

```yml
layers:
  - name: Controller
    collectors:
      - type: classNameRegex
        regex: '#.*Controller.*#
```

#### `layers`の指定3: `directory Collector`

>引用：ディレクトリコレクターは、クラスが宣言されているファイルパスを簡略化した正規表現にマッチさせることでクラスを収集することができます。マッチしたクラスは、割り当てられたレイヤーに追加されます。

ディレクトリを指定できるただ、自分で試してて知ったけど`.\\Http\\Controller\\.*`みたいなやり方で指定できるから必要性薄いかも？

```yml
layers:
  - name: Controller
    collectors:
      - type: directory
        regex: src/Controller/.*
```

#### `layers`の指定4: `bool Collector`

>引用：boolコレクターは、他のコレクターを否定の有無にかかわらず組み合わせることができます。

確実に適用させたい部分と、まあ別にいいやって部分を指定することができる。
既存プロジェクトで導入する場合とかはいいかも。

```yml
layers:
  - name: Asset
    collectors:
      - type: bool
        must:
          - type: className
            regex: .*Foo\\.*
          - type: className
            regex: .*\\Asset.*
        must_not:
          - type: className
            regex: .*Assetic.*
```

#### `layers`の指定5: `method Collector`

>引用：メソッドコレクターは、メソッド名を正規表現にマッチングさせることでクラスを収集することができます。マッチしたクラスは、割り当てられたレイヤーに追加されます。

ADRパターンとかのシングルアクションで統一している部分に対して行うとかであれば有益かな〜くらい。

```yml
layers:
  - name: Foo services
    collectors:
      - type: method
        name: .*foo
```

### `ruleset`の指定

上記で指定したレイヤー依存のルールをセットできます。
- Controller → Service: ⭕️
- Controller → Repositpry: ❌

```yml
layers:
 # 省略
ruleset:
  Controller:
    - Service
  Service:
    - Repository
  Repository: ~
```

上記のルールとセットだと、以下のような記載をすると「ControllerがInfraに依存しちゃダメよ」というエラーになります。

```php
class TestController extends Controller
{
    # ❌パターン
    public function __construct(TestRepository $testRepository)
    {
        $this->testRepository = $testRepository;
    }

    # ⭕️パターン
    public function __construct(TestService $testService)
    {
        $this->testService = $testService;
    }
}
```

```bash
 ----------- ---------------------------------------------------------------------------------------------------------------------------------------------------------------- 
  Reason      Controller                                                                                                                                                          
 ----------- ---------------------------------------------------------------------------------------------------------------------------------------------------------------- 
  Violation   App\Http\Controlller\TestController must not depend on \App\Infrastructre\TestRepository (Infrasturcutre

-------------------- ----- 
Report                    
-------------------- ----- 
Violations           1  # エラーになるとここの数字が増えます
Skipped violations   0    
Uncovered            0   
Allowed              0
Warnings             0    
Errors               0    
-------------------- ----- 
```

### ※`Uncovered`エラーについて

DeptracのReportを見ると **`Uncovered`** の数字が増えていく場合があります。
標準の出力では表示されず、`--report-uncovered`のオプションをつけることで表示されます。
LaravelなどのFWに元々組み込まれている`Illuminate~`関連がこのLintに引っかかります。

```bash
php vendor/bin/deptrac analyse --report-uncovered

477/477 [▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓] 100%

-------------------- ----- 
Report                    
-------------------- ----- 
Violations           0  
Skipped violations   0    
Uncovered            1  // これ  
Allowed              476
Warnings             0    
Errors               0    
-------------------- ----- 

Uncovered dependencies:
App\Http\Controlller\TestController has uncovered dependency on Illuminate\Http\Request (Controller)
/src/app/Http/Contrlller/TestController.php::12
```

全てを網羅するのは結構酷なので、区切りの良いところで止めるのが吉です。
最低限守りたい部分だけ守れるようにしましょう。

## DeptracをCIに組み込む

GitHub Actionsと仮定してます。
以下で作成してLintCheckを行いましょう。これで自動チェック完了。

```yml
name: LintCheck
on: [push]

jobs:
  lint-c:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Cache vendor
        id: cache
        uses: actions/cache@v1
        with:
          path: ./vendor
          key: ${{ runner.os }}-composer-${{ hashFiles('./composer.lock') }}
          restore-keys: |
            ${{ runner.os }}-composer-
      - name: composer install
        if: steps.cache.outputs.cache-hit != 'true'
        run: composer install -qn --no-interaction --no-scripts --no-progress --prefer-dist
        working-directory: ./src
      - name: Check Layer Lint
        run: php vendor/bin/deptrac analyse
        working-directory: ./src
```

### 最後に

こんな感じでレイヤーを意識した開発の手助けになってくれる静的解析ツールの紹介でございやした。全てを網羅してチェックするのは難しいですが、品質を維持する上では割と最初のうちに導入したほうがいいかな、と個人的には思いました。

**コードレビューも割と限界がある。人の目でだけで確認するのは基本NG。**
自動化でチェックできる部分はチェックしよう。

コードの品質を守りつつ、より良いプロジェクトにする為に自称コード品質保全委員会は今日も戦います。（続く...知らんけど...）

**※Follow&❤️してくれると励みになります**

#### 参考記事

https://bestofphp.com/repo/sensiolabs-de-deptrac-php-code-analysis
https://engineering.otobank.co.jp/entry/2021/01/25/185242