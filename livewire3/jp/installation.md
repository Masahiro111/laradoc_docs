Livewire は Laravel のパッケージであるため、Livewire をインストールして使用する前に、Laravel アプリケーションを起動して実行する必要があります。新しい Laravel アプリケーションのセットアップにヘルプが必要な場合は、[Laravel の公式ドキュメント](https://laravel.com/docs/installation) を参照してください。

Livewire をインストールするには、ターミナルを開いて Laravel アプリケーションのディレクトリに移動し、以下のコマンドを実行します。

```shell
composer require livewire/livewire:^3.0@beta
```

本当にそれだけです。さらにカスタマイズオプションが必要な場合は、以下のドキュメントを読み続けてください。それ以外の場合は、すぐに Livewire の使用を開始できます。

## 設定ファイルの公開

Livewire は「ゼロ設定」です。つまり、追加の設定を行わずに規則に従って使用できます。ただし、必要に応じて、以下の Artisan コマンドを実行して、Livewire の設定ファイルを公開およびカスタマイズできます。

```shell
php artisan livewire:publish --config
```

これにより、Laravel アプリケーションの `config` ディレクトリに新しい `livewire.php` ファイルが作成されます。

## Livewire のフロントエンドアセットを手動で含める

デフォルトでは、Livewire は必要な JavaScript および CSS アセットを Livewire コンポーネントを含む各ページに挿入します。

この動作をさらに制御したい場合は、次の Blade ディレクティブを使用してページにアセットを手動で含めることができます。

```blade
<html>
<head>
	...
	@livewireStyles
</head>
<body>
	...
	@livewireScripts
</body>
</html>
```

これらのアセットをページに手動で含めることで、Livewire はアセットが自動的に挿入されないことを認識します。

必要になることはほとんどありませんが、アプリケーションの `config/livewire.php` ファイル内の `inject_assets` [設定オプション](#publishing-config) を更新することで、Livewire のアセットの自動挿入動作を無効にすることができます。

```php
'inject_assets' => false,
```

## Livewire の更新エンドポイントの設定

Livewire コンポーネントの更新ごとに、エンドポイントである `https://example.com/livewire/update` にネットワークリクエストが送信されます。

This can be a problem for some applications that use localization or multi-tenancy.

In those cases, you can register your own endpoint however you like, and as long as you do it inside `Livewire::setUpdateRoute()`,  Livewire will know to use this endpoint for all component updates:

```php
Livewire::setUpdateRoute(function ($handle) {
	return Route::post('/custom/livewire/update', $handle);
});
```

Now, instead of using `/livewire/update`, Livewire will send component updates to `/custom/livewire/update`.

Because Livewire allows you to register your own update route, you can declare any additional middleware you want Livewire to use directly inside `setUpdateRoute()`:

```php
Livewire::setUpdateRoute(function ($handle) {
	return Route::post('/custom/livewire/update', $handle)
        ->middleware([...]); // [tl! highlight]
});
```

## Customizing the asset URL

By default, Livewire will serve its JavaScript assets from the following URL: `https://example.com/livewire/livewire.js`. Additionally, Livewire will reference this asset from a script tag like so:

```blade
<script src="/livewire/livewire.js" ...
```

If your application has global route prefixes due to localization or multi-tenancy, you can register your own endpoint that Livewire should use internally when fetching its JavaScript.

To use a custom JavaScript asset endpoint, you can register your own route inside `Livewire::setScriptRoute()`:

```php
Livewire::setScriptRoute(function ($handle) {
    return Route::get('/custom/livewire/livewire.js', $handle);
});
```

Now, Livewire will load its JavaScript like so:

```blade
<script src="/custom/livewire/livewire.js" ...
```

## Manually bundling Livewire and Alpine

By default, Alpine and Livewire are loaded using the `<script src="livewire.js">` tag, which means you have no control over the order in which these libraries are loaded. Consequently, importing and registering Alpine plugins, as shown in the example below, will no longer function:

```js
// Warning: This snippet demonstrates what NOT to do...

import Alpine from 'alpinejs'
import Clipboard from '@ryangjchandler/alpine-clipboard'

Alpine.plugin(Clipboard)
Alpine.start()
```

To address this issue, we need to inform Livewire that we want to use the ESM (ECMAScript module) version ourselves and prevent the injection of the `livewire.js` script tag. To achieve this, we must add the `@livewireScriptConfig` directive to our layout file (`resources/views/components/layouts/app.blade.php`):

```blade
<html>
<head>
    <!-- ... -->

    @vite(['resources/js/app.js'])
</head>
<body>
    {{ $slot }}

    @livewireScriptConfig <!-- [tl! highlight] -->
</body>
</html>
```

When Livewire detects the `@livewireScriptConfig` directive, it will refrain from injecting the Livewire and Alpine scripts. If you are using the `@livewireScripts` directive to manually load Livewire, be sure to remove it.

The final step is importing Alpine and Livewire in our `app.js` file, allowing us to register any custom resources, and ultimately starting Livewire and Alpine:

```js
import { Livewire, Alpine } from '../../vendor/livewire/livewire/dist/livewire.esm';
import Clipboard from '@ryangjchandler/alpine-clipboard'

Alpine.plugin(Clipboard)

Livewire.start()
```
