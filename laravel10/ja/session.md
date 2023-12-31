# HTTP セッション

- [はじめに](#introduction)
    - [設定](#configuration)
    - [ドライバの動作条件](#driver-prerequisites)
- [セッションの操作](#interacting-with-the-session)
    - [データの取得](#retrieving-data)
    - [データの保存](#storing-data)
    - [データの一時保存](#flash-data)
    - [データの削除](#deleting-data)
    - [セッション ID の再生成](#regenerating-the-session-id)
- [セッションブロッキング](#session-blocking)
- [カスタムセッション ドライバの追加](#adding-custom-session-drivers)
    - [ドライバの実装](#implementing-the-driver)
    - [ドライバの登録](#registering-the-driver)

<a name="introduction"></a>
## はじめに

HTTP 駆動のアプリケーションはステートレスであるため、セッションは複数のリクエストにわたってユーザーに関する情報を保存する方法を提供します。そのユーザー情報は通常、後続のリクエストからアクセスできる永続的な保存 / バックエンドに配置されます。

Laravel には、表現力豊かな統合 API を通じてアクセスされるさまざまなセッションバックエンドが用意されています。[Memcached](https://memcached.org)、[Redis](https://redis.io)、データベースなどの一般的なバックエンドをサポートしています。

<a name="configuration"></a>
### 設定

アプリケーションのセッション設定ファイルは `config/session.php` に保存されています。このファイルで使用できるオプションを必ず確認してください。デフォルトでは、Laravel は `file` セッションドライバを使用するように設定されており、これは多くのアプリケーションで適切に機能します。アプリケーションが複数の Web サーバー間で負荷分散される場合は、Redis やデータベースなど、すべてのサーバーがアクセスできる集中型保存領域を選択する必要があります。

セッションの `driver` 設定オプションは、各リクエストのセッションデータが保存される場所を定義します。Laravel には、すぐに使用できるいくつかの優れたドライバが同梱されています。

<div class="content-list" markdown="1">

- `file` - セッションを `storage/framework/sessions` に保存します。
- `cookie` - セッションを安全な暗号化された cookie に保存します。
- `database` - セッションをリレーショナルデータベースに保存します。
- `memcached` / `redis` - セッションを、これらの高速なキャッシュベースの保存領域のいずれかに保存します。
- `dynamodb` - セッションを AWS DynamoDB に保存します。
- `array` - セッションを PHP 配列に保存され、永続化されません。

</div>

> **Note**  
> array ドライバは主に [testing](/docs/{{version}}/testing) 中に使用され、セッションに保存されたデータが永続化されるのを防ぎます。

<a name="driver-prerequisites"></a>
### ドライバの動作条件

<a name="database"></a>
#### データベース

`database` セッションドライバを使用する場合は、セッションレコードを含むテーブルを作成する必要があります。テーブルの `Schema` 宣言の例を以下に示します。

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    Schema::create('sessions', function (Blueprint $table) {
        $table->string('id')->primary();
        $table->foreignId('user_id')->nullable()->index();
        $table->string('ip_address', 45)->nullable();
        $table->text('user_agent')->nullable();
        $table->text('payload');
        $table->integer('last_activity')->index();
    });

このマイグレーションを生成するには、`session:table` Artisan コマンドを使用します。データベースのマイグレーションの詳細については、完全な [マイグレーションドキュメント](/docs/{{version}}/migrations) を参照してください。

```shell
php artisan session:table

php artisan migrate
```

<a name="redis"></a>
#### Redis

Laravel で Redis セッションを使用する前に、PECL 経由で PhpRedis PHP 拡張機能をインストールするか、Composer 経由で `predis/predis` パッケージ (~1.0) をインストールする必要があります。Redis の設定の詳細については、Laravel の [Redis ドキュメント](/docs/{{version}}/redis#configuration) を参照してください。

> **Note**
> `session` 設定ファイルでは、`connection` オプションを使用して、セッションで使用される Redis 接続を指定できます。

<a name="interacting-with-the-session"></a>
## セッションの操作

<a name="retrieving-data"></a>
### データの取得

Laravel でセッションデータを操作するには、主に２つの方法があります。グローバル `session` ヘルパと `Request` インスタンス経由です。まず、`Request` インスタンスを介してセッションにアクセスする方法を見てみましょう。これは、ルートクロージャまたはコントローラメソッドでタイプヒントを指定します。 コントローラメソッドの依存関係は、Laravel [サービスコンテナ](/docs/{{version}}/container) 経由で自動的に依存性注入されます。

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use Illuminate\Http\Request;
    use Illuminate\View\View;

    class UserController extends Controller
    {
        /**
         * Show the profile for the given user.
         */
        public function show(Request $request, string $id): View
        {
            $value = $request->session()->get('key');

            // ...

            $user = $this->users->find($id);

            return view('user.profile', ['user' => $user]);
        }
    }

セッションからアイテムを取得するとき、デフォルト値を第２引数として `get` メソッドに渡すことができます。指定されたキーがセッションに存在しない場合、このデフォルト値が返されます。クロージャをデフォルト値として `get` メソッドに渡し、要求されたキーが存在しない場合、クロージャが実行され、その結果が返されます。

    $value = $request->session()->get('key', 'default');

    $value = $request->session()->get('key', function () {
        return 'default';
    });

<a name="the-global-session-helper"></a>
#### グローバルセッションヘルパ

グローバル `session` PHP 関数を使用して、セッション内のデータを取得および保存することもできます。`session` ヘルパが単一の文字列引数を指定して呼び出されると、そのセッションキーの値が返されます。キーと値のペアの配列を使用してヘルパを呼び出すと、それらの値はセッションに保存されます。

    Route::get('/home', function () {
        // セッションからデータを取得します...
        $value = session('key');

        // デフォルト値を指定しています...
        $value = session('key', 'default');

        // データをセッションに保存します...
        session(['key' => 'value']);
    });

> **Note**
> HTTP リクエストインスタンス経由でセッションを使用する場合と、グローバル `session` ヘルパを使用する場合には、実質的な違いはほとんどありません。どちらのメソッドも、すべてのテストケースで使用できる `assertSessionHas` メソッド経由で [テスト可能](/docs/{{version}}/testing) になります。

<a name="retrieving-all-session-data"></a>
#### 全セッションデータの取得

セッション内のすべてのデータを取得したい場合は、`all` メソッドを使用します。

    $data = $request->session()->all();

<a name="determining-if-an-item-exists-in-the-session"></a>
#### セッション内アイテムの存在判定

アイテムがセッションに存在するかを確認するには、`has` メソッドを使用します。アイテムが存在し、`null` でない場合、`has` メソッドは `true` を返します。

    if ($request->session()->has('users')) {
        // ...
    }

`exists` メソッドを使用することで、指定したセッション内のアイテムが `null` だとしても、存在するかどうか確認できます。

    if ($request->session()->exists('users')) {
        // ...
    }

アイテムがセッションに存在しないかどうかを判断するには、`missing` メソッドを使用します。アイテムが存在しない場合、`missing` メソッドは `true` を返します。

    if ($request->session()->missing('users')) {
        // ...
    }

<a name="storing-data"></a>
### データの保存

セッションにデータを保存するには、通常、リクエストインスタンスの `put` メソッドまたはグローバル `session` ヘルバを使用します。

    // リクエストインスタンス経由
    $request->session()->put('key', 'value');

    // グローバル session ヘルパ経由
    session(['key' => 'value']);

<a name="pushing-to-array-session-values"></a>
#### 配列セッション値への追加

`push` メソッドは、配列のセッション値に新しい値を追加できます。たとえば、`user.teams` キーにチーム名の配列が含まれている場合、次のように配列に新しい値を追加できます。

    $request->session()->push('user.teams', 'developers');

<a name="retrieving-deleting-an-item"></a>

#### アイテムの取得と削除

`pull` メソッドは、単一のステートメントでセッションからアイテムを取得して削除します。

    $value = $request->session()->pull('key', 'default');

<a name="#incrementing-and-decrementing-session-values"></a>
#### セッション値の増減

セッション データにインクリメントまたはデクリメントしたい整数が含まれている場合は、`increment` および `decrement` メソッドを使用できます。

    $request->session()->increment('count');

    $request->session()->increment('count', $incrementBy = 2);

    $request->session()->decrement('count');

    $request->session()->decrement('count', $decrementBy = 2);

<a name="flash-data"></a>
### データの一時保存

場合によっては、後継リクエストに備えてセッションにアイテムを保存したい際は、`flash` メソッドを使用します。このメソッドを使用してセッションに保存されたデータは、即時もしくは後続の HTTP リクエスト中で使用できるようになります。後続の HTTP リクエストの後、一時保存されたデータは削除されます。一時保存データは主に、短期間のステータスメッセージに役立ちます。

    $request->session()->flash('status', 'Task was successful!');

複数のリクエストの間、一時保存データを保持する必要がある場合は、追加のリクエストに備えてすべての一時保存データを保持する `reflash` メソッドを使用します。特定のフラッシュデータのみを保持する必要がある場合は、`keep` メソッドを使用します。

    $request->session()->reflash();

    $request->session()->keep(['username', 'email']);

現在のリクエストに対してのみ一時保存データを保持するには、`now` メソッドを使用します。

    $request->session()->now('status', 'Task was successful!');

<a name="deleting-data"></a>
### データの削除

`forget` メソッドはセッションからデータの一部を削除します。セッションからすべてのデータを削除したい場合は、`flush` メソッドを使用することができます。

    // １つのキーを削除 ...
    $request->session()->forget('name');

    // 複数のキーを削除 ...
    $request->session()->forget(['name', 'status']);

    $request->session()->flush();

<a name="regenerating-the-session-id"></a>
### セッション ID の再生成

セッション ID の再生成は、悪意のあるユーザーがアプリケーションに対して [セッション固定](https://owasp.org/www-community/attacks/Session_fixation) 攻撃をを防ぐために行われることがよくあります。

Laravel [アプリケーションスターターキット](/docs/{{version}}/starter-kits) または [Laravel Fortify](/docs/{{version}}/fortify) のいずれかを使用している場合、Laravel は認証中にセッション ID を自動的に再生成します。ただし、セッション ID を手動で再生成する必要がある場合は、`regenerate` メソッドを使用してください。

    $request->session()->regenerate();

セッション ID を再生成し、セッションからすべてのデータを削除したい場合、`invalidate` メソッドを使用すると一文で記述可能です。

    $request->session()->invalidate();

<a name="session-blocking"></a>
## セッションブロッキング

> **Warning**
> セッションブロッキングを利用するには、アプリケーションで [アトミックロック](/docs/{{version}}/cache#atomic-locks) をサポートするキャッシュドライバを使用する必要があります。現在、これらのキャッシュドライバには、`memcached`、`dynamodb`、`redis`、および `database` ドライバが含まれます。`cookie` セッションドライバを使用することはできません。

デフォルトでは、Laravel は同じセッションを使用するリクエストの同時実行を許可します。たとえば、JavaScript HTTP ライブラリを使用してアプリケーションに対して２つの HTTP リクエストを作成すると、両方が同時に実行されます。多くのアプリケーションでは、これは問題になりません。 ただし、セッションデータの損失は、両方ともセッションにデータを書き込む２つの異なるアプリケーションエンドポイントに同時にリクエストを行うアプリケーションの小さなサブセットで発生する可能性があります。

これを軽減するために、Laravel は特定のセッションの同時リクエストを制限できる機能を提供します。まず、ルート定義に `block` メソッドをチェーンさせるだけです。この例では、`/profile` エンドポイントへの受信リクエストはセッションロックを取得します。このロックが保持されている間、同じセッション ID を共有する `/profile` または `/order` エンドポイントへの受信リクエストは、最初のリクエストの実行が完了するまで待機してから、実行を続行します。

    Route::post('/profile', function () {
        // ...
    })->block($lockSeconds = 10, $waitSeconds = 10)

    Route::post('/order', function () {
        // ...
    })->block($lockSeconds = 10, $waitSeconds = 10)

`block` メソッドは２つのオプションの引数を受け取ります。`block` メソッドの第１引数は、セッションロックが解放されるまで保持される最大秒数です。もちろん、この時間より前にリクエストの実行が終了した場合、ロックはより早く解放されます。

`block` メソッドでの第２引数は、セッションロックの取得を試行する際にリクエストが待機する秒数です。リクエストが指定された秒数以内にセッションロックを取得できない場合、`Illuminate\Contracts\Cache\LockTimeoutException` がスローされます。

これらの引数のどちらも渡されない場合、ロックは最大１０秒間取得され、リクエストはロックの取得を試行する間最大１０秒間待機します。

    Route::post('/profile', function () {
        // ...
    })->block()

<a name="adding-custom-session-drivers"></a>
## カスタムセッションドライバの追加

<a name="implementing-the-driver"></a>
#### ドライバの実装

既存のセッションドライバがアプリケーションのニーズに適合しない場合は、Laravel を使用して独自のセッションハンドラを作成できます。カスタム セッションドライバは、PHP の組み込みの `SessionHandlerInterface` を実装する必要があります。このインターフェイスには、いくつかの簡単なメソッドが含まれています。スタブ化された MongoDB 実装は以下のようになります。

    <?php

    namespace App\Extensions;

    class MongoSessionHandler implements \SessionHandlerInterface
    {
        public function open($savePath, $sessionName) {}
        public function close() {}
        public function read($sessionId) {}
        public function write($sessionId, $data) {}
        public function destroy($sessionId) {}
        public function gc($lifetime) {}
    }

> **Note**
> Laravel には、拡張機能を格納するディレクトリはありません。好きな場所に自由に配置できます。この例では、`MongoSessionHandler` を格納する `Extensions` ディレクトリを作成しました。

これらのメソッドの目的はすぐには理解しずらいため、各メソッドの機能を簡単に説明します。

<div class="content-list" markdown="1">

- `open` メソッドは通常、ファイルベースのセッションストアシステムで使用されます。Laravel には `file` セッションドライバが付属しているため、このメソッドに何も入れる必要はほとんどありません。このメソッドは空のままにすることができます。
- `close` メソッドも、`open` メソッドと同様に、通常は無視できます。ほとんどのドライバでは必要ありません。
- `read` メソッドは、指定された `$sessionId` に関連付けられたセッションデータの文字列バージョンを返す必要があります。Laravel がシリアル化を実行するため、ドライバでセッションデータを取得または保存するときにシリアル化やその他のエンコードを行う必要はありません。
- `write` メソッドは、`$sessionId` に関連付けられた指定の `$data` 文字列を、MongoDB や選択した別のストレージシステムなどの永続ストレージシステムに書き込む必要があります。繰り返しますが、シリアル化を実行しないでください。Laravel がすでにシリアル化を処理しています。
- `destroy` メソッドは、`$sessionId` に関連付けられたデータを永続ストレージから削除する必要があります。
- `gc` メソッドは、指定の `$lifetime` (UNIX タイムスタンプ) よりも古いすべてのセッションデータを破棄する必要があります。Memcached や Redis などの自己期限切れシステムの場合、このメソッドは空のままにすることができます。

</div>

<a name="registering-the-driver"></a>
#### ドライバの登録

ドライバを実装したら、Laravel に登録する準備が整いました。Laravel のセッションバックエンドにドライバを追加するには、`Session` [ファサード](/docs/{{version}}/facades) によって提供される `extend` メソッドを使用します。[サービスプロバイダ](/docs/{{version}}/providers) の `boot` メソッドから `extend` メソッドを呼び出す必要があります。既存の `App\Providers\AppServiceProvider` からこれを行うことも、また、まったく新しいプロバイダを作成することもできます。

    <?php

    namespace App\Providers;

    use App\Extensions\MongoSessionHandler;
    use Illuminate\Contracts\Foundation\Application;
    use Illuminate\Support\Facades\Session;
    use Illuminate\Support\ServiceProvider;

    class SessionServiceProvider extends ServiceProvider
    {
        /**
         * 各種アプリケーションサービスを登録します。
         */
        public function register(): void
        {
            // ...
        }

        /**
         * あらゆるアプリケーションサービスを初期起動します。
         */
        public function boot(): void
        {
            Session::extend('mongo', function (Application $app) {
                // SessionHandlerInterface の実装を返します...
                return new MongoSessionHandler;
            });
        }
    }

セッションドライバを登録すると、`config/session.php` 設定ファイルで `mongo` ドライバを使用できます。
