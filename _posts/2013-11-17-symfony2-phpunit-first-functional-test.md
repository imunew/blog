---
layout: post
title: Symfony2 + PHPUnitではじめてのテスト（機能テスト編）
category : test
tagline: ""
tags : [Symfony2, PHPUnit, Doctrine, test]
---
{% include JB/setup %}


##さっそくやってみよう
<p>導入方法および単体テストについては、下記を参照ください。</p>

[Symfony2 + PHPUnitではじめてのテスト（単体テスト編）](/test/2013/11/11/symfony2-phpunit-first-unit-test/)

###機能テスト（Controller編）

####テストケースの作成
コントローラーをconsoleから作成すると、テストケースももれなく作成されます。

[Generating a New Controller - Symfony(current)](http://symfony.com/doc/current/bundles/SensioGeneratorBundle/commands/generate_controller.html)

> The generate:controller command generates a new Controller including actions, <strong>tests</strong>, templates and routing.

[ControllerGenerator.php - Github](https://github.com/sensiolabs/SensioGeneratorBundle/blob/master/Generator/ControllerGenerator.php)

{% highlight php %}
$this->renderFile('controller/ControllerTest.php.twig', $dir.'/Tests/Controller/'.$controller.'ControllerTest.php', $parameters);
{% endhighlight %}

自動生成されたテストケースは下記のようなイメージ

{% highlight php %}
<?php
namespace Sopra\WebBundle\Tests\Controller;
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

class MypageControllerTest extends WebTestCase
{
}
{% endhighlight %}

####環境変数を設定
必要に応じて、phpunit.xmlに変数設定を追加します。

{% highlight xml %}
<phpunit>
    <!-- ... -->
    <php>
        <server name="KERNEL_DIR" value="../app/" />
        <server name="SERVER_NAME" value="local.sopra.jp" />
        <server name="SERVER_PORT" value="80" />
        <server name="SCRIPT_NAME" value="" />
        <server name="REQUEST_URI" value="" />
    </php>
    <!-- ... -->
</phpunit>
{% endhighlight %}


####テストメソッドの実装
<p>大まかにいうとテストメソッドの処理フローは以下のようになります。</p>

{% highlight php %}
// 'test'で始まるpublic関数を定義
public function testExecuteSuccess() {
        
    // 必要に応じて環境変数を設定
        
    // 準備（データの初期化など）
        
    // ページにアクセス
        
    // テストしたい処理を実行（送信ボタンをクリックなど）
    
    // 結果を検証
}
{% endhighlight %}

##実践してみよう（sopraの場合）
sopraで実際に実装したときに、つまづいた点などを中心にポイントを解説していきます。
###ログイン
<p>ログインが必要なページをテストする場合は、ログイン状態をシミュレートしなければいけません。<br>
symfony.comに説明があるので、ほぼそのとおり実装しました。<br>
</p>
[How to simulate Authentication with a Token in a Functional Test (current) - Symfony](http://symfony.com/doc/current/cookbook/testing/simulating_authentication.html)

{% highlight php %}
private function logIn()
{
    $session = $this->client->getContainer()->get('session');

    $firewall = 'secured_area';
    $token = new UsernamePasswordToken('admin', null, $firewall, array('ROLE_ADMIN'));
    $session->set('_security_'.$firewall, serialize($token));
    $session->save();

    $cookie = new Cookie($session->getName(), $session->getId());
    $this->client->getCookieJar()->set($cookie);
}
{% endhighlight %}

###テスト用データの投入（Doctrin Fixtures Bundleの導入）
テストを実行するたびにデータを手動で投入していたのでは、意味が無いので、Doctrin Fixtures Bundleを導入することにしました。

[DoctrineFixturesBundle - Symfony](http://symfony.com/doc/current/bundles/DoctrineFixturesBundle/index.html)

{% highlight json %}
{
    "require": {
        "doctrine/doctrine-fixtures-bundle": "dev-master"
    }
}
{% endhighlight %}

<p>sopraでは、いったんテーブルをtruncateしてからデータを投入するようにしています。</p>

{% highlight php %}
<?php

namespace Sopra\CoreBundle\DataFixtures\Helper;

use Doctrine\ORM\EntityManager;

class EntityHelper {
    
    private $entityManager;
    
    public function __construct(EntityManager $entityManager) {
        
        $this->entityManager = $entityManager;
    }
    
    public function truncateTables($tables) {
        
        $connection = $this->entityManager->getConnection();
        $platform = $connection->getDatabasePlatform();
        
        $connection->exec('SET FOREIGN_KEY_CHECKS = 0');

        foreach ($tables as $table) {
            $connection->exec($platform->getTruncateTableSQL($table));
        }
        
        $connection->exec('SET FOREIGN_KEY_CHECKS = 1');
    }
    
}
{% endhighlight %}

'SET FOREIGN_KEY_CHECKS = 0'は外部キーを無視するために実行していますが、mysql固有の関数なので修正必要ですね。

[MySQL 5.1 リファレンスマニュアル :: 13.5.6.4 FOREIGN KEY 制約](http://dev.mysql.com/doc/refman/5.1/ja/innodb-foreign-key-constraints.html)

さらに、テストケースからFixtureをloadするために、別のHelperクラスを実装しています。

{% highlight php %}
<?php

namespace Sopra\CoreBundle\DataFixtures\Helper;

use Doctrine\Common\DataFixtures\Executor\ORMExecutor;
use Doctrine\Common\DataFixtures\FixtureInterface;
use Doctrine\Common\DataFixtures\Loader;
use Doctrine\ORM\EntityManager;
use Symfony\Component\DependencyInjection\ContainerAwareInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

class FixturesHelper {
    
    private $container;
    private $entityManager;
    
    public function __construct(ContainerInterface $container, EntityManager $entityManager) {
        
        $this->container = $container;
        $this->entityManager = $entityManager;
    }
    
    public function loadFixtures($fixtures) {
        
        $loader = new Loader();
        foreach ($fixtures as $fixture) {
            if ($fixture instanceof ContainerAwareInterface) {
                $fixture->setContainer($this->container);
            }
            if ($fixture instanceof FixtureInterface) {
                $loader->addFixture($fixture);
            }
        }
        $executor = new ORMExecutor($this->entityManager);
        $executor->execute($loader->getFixtures(), true);
    }
    
}
{% endhighlight %}

各Fixtureからサービスコンテナにアクセスするために、ContainerAwareInterfaceを実装します。
sopraでは、EntityHelperに渡すEntityManagerを取得するために、setContainerしています。

[Using the Container in the Fixtures - Symfony](http://symfony.com/doc/current/bundles/DoctrineFixturesBundle/#using-the-container-in-the-fixtures)

各クラス、インターフェイスの関係をクラス図にまとめると下記のようになります。

<a href="/images/data_fixtures.png">
<img src="/images/data_fixtures.png">
</a>


###ページにアクセス
ページへのアクセスはSymfony\Bundle\FrameworkBundle\Clientオブジェクトを通じて行いますので、setupメソッドにて生成し、メンバ変数にセットしておきます。

{% highlight php %}
namespace Sopra\WebBundle\Tests\TestCase;

use Symfony\Bundle\FrameworkBundle\Client;
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

abstract class BaseWebTestCase extends WebTestCase {

    /** @var Client */
    private $client;

    protected function setUp() {
        
        parent::setUp();
        
        $this->client = static::createClient();

{% endhighlight %}

Client::requestメソッドにてページにアクセスします。
リダイレクトされる場合はfollowRedirectし、戻り値のcrawlerオブジェクトを取得します。

{% highlight php %}
$this->getClient()->request('GET', '/path/to/page');
$crawler = $this->getClient()->followRedirect();
/* @var $crawler Symfony\Component\DomCrawler\Crawler */
{% endhighlight %}

###テストしたい処理を実行

'Submit'ボタンをクリックして、レスポンスを取得する場合は、下記のような実装になります。

{% highlight php %}
$form = $crawler->selectButton('Submit')->form();
        
$this->getClient()->submit($form);
$crawler = $this->getClient()->followRedirect();
{% endhighlight %}

###結果を検証

結果ページに'Success'という文字列が含まれているかどうかを確認するには、下記のような実装になります。

{% highlight php %}
$this->assertTrue($crawler->filter('html:contains("Success")')->count() > 0);
{% endhighlight %}

<p>
'Success'と言う文字列が一つも含まれない場合は、その時点でテスト失敗となります。
上記のassertTrue意外にも、PHPUnitにはあらかじめ様々なアサーションメソッドが用意されています。
</p>
[アサーション - phpunit](http://phpunit.de/manual/3.7/ja/writing-tests-for-phpunit.html#writing-tests-for-phpunit.assertions)


###実装サンプル
[実装サンプル - gitlab.hitomedia.jp](https://gitlab.hitomedia.jp/sopra/tree/master/src/Sopra/WebBundle/Tests/Controller/WalletControllerTest.php)


##テスト実行

{% highlight bash %}
$ cd bin
$ ./phpunit -c ../app/ ../src/Sopra/WebBundle/Tests/Controller/WalletControllerTest.php
PHPUnit 3.7.27 by Sebastian Bergmann.

Configuration read from /Users/imura/workspace/sopra/app/phpunit.xml

.....

Time: 7.95 seconds, Memory: 51.50Mb

OK (5 tests, 33 assertions)
{% endhighlight %}

