---
layout: post
title: Symfony2 + PHPUnitではじめてのテスト（単体テスト編）
category : test
tagline: ""
tags : [Symfony2, PHPUnit, test]
---
{% include JB/setup %}

##まずは導入してみよう
<p>なぜ、テストコードを書くのか、テストコードはどう書くべきか、といった話はおいといて、今回はとにかくテストコードを書いて実行してみるということを目的とします。
</p>

###PHPUnitのインストール(composer)
<p>さっそくインストールからはじめましょう。<br>
今回はタイトルにもあるようにSymfony2(2.3)環境にPHPUnit(3.7)を導入します。<br>
composer.jsonに下記記述を追加し、updateすれば、インストールは完了です。</p>
[第3章 PHPUnit のインストール - phpunit](http://phpunit.de/manual/3.7/ja/installation.html#installation.composer)

{% highlight json %}
{
    "require-dev": {
        "phpunit/phpunit": "3.7.*"
    }
}
{% endhighlight %}

##さっそくやってみよう
<p>はじめにこれを読みましょう。</p>
* [Testing - Symfony(current)](http://symfony.com/doc/current/book/testing.html)
* [テスト - Symfony2日本語ドキュメント](http://docs.symfony.gr.jp/symfony2/book/testing.html)

###ユニットテスト
<p>下記のようなシンプルなクラスがあるとします。<br>
年月を指定して、月初もしくは月末を取得するだけのクラスですが、実際に使う前に単体である何パターンかテストしておきたいところです。
</p>
{% highlight php %}
<?php
namespace Sopra\CoreBundle\Helper;
use DateTime;

class DateTimeHelper {
    
    private function __construct() {  }
    
    public static function getFirstDayInMonth($year, $month) {
        
        return new DateTime("$year-$month-01");
    }
    
    public static function getLastDayInMonth($year, $month) {
        
        return new DateTime(date('Y-m-d', mktime(0,0,0, $month + 1, 0, $year)));
    }
}
{% endhighlight %}

<p>そこで、下記のようなテストケースを作成します。
</p>

{% highlight php %}
<?php
namespace Sopra\CoreBundle\Tests\Helper;

use Sopra\CoreBundle\Helper\DateTimeHelper;
use Sopra\CoreBundle\Tests\TestCase\BaseTestCase;

class DateTimeHelperTest extends BaseTestCase {
    
    public function testGetFirstDayInMonth() {
        
        $firstDay = DateTimeHelper::getFirstDayInMonth(2013, 1);
        $this->assertEquals('2013-01-01', $firstDay->format('Y-m-d'));
        
        $firstDay = DateTimeHelper::getFirstDayInMonth(2013, 12);
        $this->assertEquals('2013-12-01', $firstDay->format('Y-m-d'));
        
        $firstDay = DateTimeHelper::getFirstDayInMonth(2014, 3);
        $this->assertEquals('2014-03-01', $firstDay->format('Y-m-d'));
    }
    
    public function testGetLastDayInMonth() {
        
        $lastDay = DateTimeHelper::getLastDayInMonth(2012, 2);
        $this->assertEquals('2012-02-29', $lastDay->format('Y-m-d'));
        
        $lastDay = DateTimeHelper::getLastDayInMonth(2013, 2);
        $this->assertEquals('2013-02-28', $lastDay->format('Y-m-d'));
        
        $lastDay = DateTimeHelper::getLastDayInMonth(2013, 12);
        $this->assertEquals('2013-12-31', $lastDay->format('Y-m-d'));
    }
}
{% endhighlight %}

####テストの実行
<p>作成したテストを実行するために、phpunitコマンドを実行します。<br>
phpunitコマンドについては、本家サイトに詳しい説明がありますので参照ください。<br>
[コマンドラインのテストランナー - PHPUnit](http://phpunit.de/manual/3.7/ja/textui.html)
実際に上述のDateTimeHelperTestを実行してみると、下記のように出力されます。
</p>

{% highlight bash %}
$ cd bin
$ ./phpunit -c ../app/ ../src/Sopra/CoreBundle/Tests/Helper/DateTimeHelperTest.php
PHPUnit 3.7.27 by Sebastian Bergmann.

Configuration read from /path/to/Application Root/app/phpunit.xml

..

Time: 353 ms, Memory: 16.25Mb

OK (2 tests, 6 assertions)
{% endhighlight %}

