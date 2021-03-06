# 9章 Railsのインフラと運用

1,2,3担当: @shoken0x  
4,5,6担当: @hiroki-tkg


## Summary

Rails開発の基本は、Skinny Controller/Fat Model  
しかし、アプリケーションが複雑化してきた時、モデル層が肥大化し過ぎて問題が発生する。  
この章では複雑なアプリケーションをRailsで開発していく際に起きうる問題を解決するためのテクニックや考え方を紹介する。


## 9-1. アーキテクチャパターンから見るRails p.332

### ドメインモデルとアクティブレコード

#### ドメインモデルとは？

- システム開発において、システムの対象となる業務に必要な知識を抽象化し、各概念の関係性を明確にすることが重要。業務知識を噛み砕いて形にするその行程を**モデリング**を言います。
- モデリングによって抽出された概念に名前をつけ、オブジェクトとして表現できるようにしたものを**ドメインモデル**と呼ぶ
- ドメインモデルは業務に必要な振る舞いについて責任を持っている。その業務に必要な振る舞いを**ドメインロジック**（またはビジネスロジック）と呼ぶ
- Railsのモデル層はドメインモデルとドメインロジックをコードとして形にするためのレイヤー

つまり、**モデル層はアプリケーションの最も中心的な振る舞いを記述する場所**ということ

ドメインモデルについての書籍

- 古典: [ドメイン駆動設計: エリック・エヴァンス](http://www.amazon.co.jp/dp/4798121967)

<img src="http://ecx.images-amazon.com/images/I/51f7WXHJYCL._SX391_BO1,204,203,200_.jpg" height="200"/>

- 新しい: [実践ドメイン駆動設計: ヴァーン・ヴァーノン](http://www.amazon.co.jp/dp/479813161X)

<img src="http://ecx.images-amazon.com/images/I/61Y5Po5i8lL._SX394_BO1,204,203,200_.jpg" height="200"/>

参考サイト

- [ドメイン駆動設計 - Wikipedia](https://ja.wikipedia.org/wiki/%E3%83%89%E3%83%A1%E3%82%A4%E3%83%B3%E9%A7%86%E5%8B%95%E8%A8%AD%E8%A8%88)
- [\[ 技術講座 \] Domain-Driven Designのエッセンス 第1回｜オブジェクトの広場](https://www.ogis-ri.co.jp/otc/hiroba/technical/DDDEssence/chap1.html)
- [DDD - MVCの流れを簡単にまとめてみる - Qiita](http://qiita.com/hirokidaichi/items/0de5ca336de862cc91bd)
- [データモデルはドメインモデルに先行する: 設計者の発言](http://watanabek.cocolog-nifty.com/blog/2015/06/post-5d2a.html)

### 復習: MVCとは？

[MVCもやもや話](http://www.slideshare.net/kaniza1/mvc-12990574)

Model-View-Controller

- オブジェクト指向のGUI向けデザインパターン
- Smalltalkにおけるウィンドウプログラム開発のための設計指針として生まれた
- (オブジェクト指向と同じく)人によって言うことが違う

それぞれの役割

- Model
  - データの永続性、ビジネスロジック（データの整合性含む）
  - GUIとは分離している
  - ViewやControllerのことは知らない
- View
  - 表示されて人が操作する部分
  - Modelのことを知っているが、Controllerのことは知らない
- Controller
  - ModelとViewをつなぐ
  - ModelのこともViewのことも知っている


[やはりお前らのMVCは間違っている](http://www.slideshare.net/MugeSo/mvc-14469802)

**Model = ActiveRecordではない！**
CUIでもGUIでもWebアプリケーションでも変わらない処理があるなら、それはモデルの仕事


元来のMVC

![Screen Shot 2015-10-05 at 4.26.14 PM.png](https://qiita-image-store.s3.amazonaws.com/0/32750/646d7a59-079a-834e-ec75-f061abb7a992.png "Screen Shot 2015-10-05 at 4.26.14 PM.png")


WebでのMVC

![Screen Shot 2015-10-05 at 4.25.57 PM.png](https://qiita-image-store.s3.amazonaws.com/0/32750/2150b517-3aa5-63bd-b804-20d1a4c5f83d.png "Screen Shot 2015-10-05 at 4.25.57 PM.png")

### MVCについて考える

電卓アプリをGUIで開発することを考えてみる。

- MVCにわけずに、1つのファイルで作ることも可能。何が問題になる？  
- GUI電卓アプリを、CUIやWebサービスへの移植を考えると...


**クラスごとに責務が別れていて疎結合だと、再利用やテストがしやすい**


#### アクティブレコード

ドメインモデルは純粋に業務に必要な概念のみを抽出した抽象的なもの。現実世界でアプリケーションとして実装するにはDBとの連携で永続化することが必要不可欠。

RailsはオブジェクトとRDBのデータをマッピングするため**アクティブレコードパターン**を採用している。そのためRailsがモデル層で利用するライブラリの名前がActiveRecord。


### アクティブレコードパターンとは

- ドメインモデルに関わるロジックと、データの永続化処理を1つのクラスにまとめてカプセル化する手法
- RDBでのテーブルがクラス、レコード1つがインスタンスに対応する
- メリット: RDBのテーブル構造とクラスの関係がシンプルに表現されており直感的に理解しやすい
- デメリット: RDBのテーブルとクラスを直接対応付けなければならない制約のため、複雑なドメインモデルを表現するには限界がある


### ActiveRecordの落とし穴とそれを回避する方法

課題: 1つのクラスがロジックとデータの永続化の両方について責任を持っているアクティブレコードのアーキテクチャでは、モデルが一気に肥大化する

肥大化する主な原因:

- RDBのテーブル構造と一体になっていること。構造に縛られたクラス設計になっているとどうしても様々な機能が1つのクラスの中に押し込まれてしまう。
- 1つのクラスの中で広範囲の処理ができること。バリデーションやコールバックを宣言的に記述する仕組みのため、データの整合性に対する責任も含むことになり多機能化していく。


フォース:

- RDBに依存しないモデルクラスを作成する
- 1つの物事についてのみ責任を持つ小さなクラスを作成する

具体的な実装:

- バリデーションとコールバック機能を小さなクラスに分離する
- ActiveModelを直接利用してRDBに依存しないモデルクラスを定義する
- システム機能の一部をActiveRecordモデルの外に抽出するためのテクニック　

## 9-2. 複雑なバリデーションとコールバックを整理する p.334

バリデーションやコールバックの中には、複数のモデルに関係するような複雑なルールが必要になる場合がある。  
そうなった時、単一のクラスに分離し責任を明確にするという選択が有効な場合がある。

### コールバックをクラスに分離する

コールバックは、

- ブロックで記述する
- シンボルで記述する
- オブジェクトを渡す

という3つの方法がある。通常は、シンボルで記述する。ブロックで記述するとコールバックの意味を表現する名前が付けられないため避けたるべき。

オブジェクトで渡す具体的な実装は p.335


#### メリット

- 複数のフックポイントに股がって一つの機能を実現する場合のコールバックの関係性を明確に表現できる
- 機能（責任）の境界がはっきりするすることで、テストを行いやすくなる
  - テストしたいのはコールバックで呼ばれている暗号化のロジックなのに、ActiveRecordインスタンスを準備するのは大変
  - ActiveRecordに依存しない形で複雑な処理のテストを記述できる

テストがしやすい(テストのためにインスタンスを準備しなくてもよい) = 疎結合  
テストのしやすさと結合度は密接に関係している。疎結合だとテストしやすい。また再利用もしやすい。


#### バリデーションをクラスに分離する

バリデーションの記述方法

- `validates`で組み込みのバリデーターを呼び出す
- `validate`メソッドに対してシンボル、またはブロックを渡す（シンボル推奨）

#### EachValidatorクラス

1つの属性値を検証するのに特化したクラス

ActiveModel::EachValidatorを継承したクラスを定義すると、`validates`メソッドがそのクラス名に基づいたオプション値を受け取れるようになる。

具体的な実装は p.338


#### Validatorクラス

1つの属性値だけに留まらない複雑な検証ルールが必要になった場合は`ActiveModel::Validator`クラスを継承したクラスを定義する。`validates_with`メソッドの引数にバリデータークラスを指定して利用する。

複雑な検証ルールの例: Scheduleの持つstarted_atとfinished_atが前後のScheduleと重なっていないか

具体的な実装は p.340


#### メリット

- 責任の境界がはっきりする
- テストがしやすくなる


### どういった場合にクラスを分離するか

別クラスに分離する場合のデメリット:

- コード量が増加する
- 実装が分散することで、モデルクラスの動作を把握する手間が増大する

メリットが上回る使いどころ:

- 処理対象のモデルクラスの成約が厳しく、テストのための事前条件を満たすための負荷が大きい場合
  - 本質的ではない事前条件のセットアップは手間となる。また複雑なセットアップはテストの信頼性を低下させる。
- 処理が業務知識の中で重要な位置を占める場合
  - コールバックやバリデーションが単体で意味を持つ場合、独立した概念であることを実装上でも示しておくため業務上のルールに相当する名前を付けてクラスを分離する
  - 業務上の概念と設計上の概念ができるだけ一致していることが望ましい
  - バリデーションやコールバックの実行に条件を持たせるのはバッドプラクティス。サービスクラスに記述した方が良い場合もある。対象となる制約条件がどの範囲で利用されるのかを考えた上で設計する

参考: バリデーションを含むビジネスロジックをフォーム毎に分離したTrailblazer  
[Trailblazer勉強会に参加してきた #trbtky - Shoken Startup Blog](http://shoken.hatenablog.com/entry/2015/09/21/091353)


### 別のクラスに分離する以外の方法

別のクラスにする他、Moduleにまとめて切り出して表現することも可能。この考え方をConcernと呼ぶ。


## 9-3. DBに依存しないモデルを作るActiveModel p.343

- Railsのモデル層の基本はActiveRecord
- ActiveRecordはRDBの構造と直接対応したものとなっている

上記の制約のもとでは表現できない概念が登場することがある。そういった場合、ActiveRecordを利用せずにRDBと直接対応**しない**クラスを自分で定義したい。

一方で、AcriveRecordの豊富な機能は使いたい。  
ActiveRecordの主な機能

- バリデーション
- コールバック
- 属性名を元にした動的なメソッドを定義
- 属性値に対する変更を保持する
- オブジェクトのシリアライズ

そこでActiveModelモジュールを使う。上記の機能もActiveModelによって定義されている。

ActiveModelが提供する機能

- ActiveModel::AttributeMethods
- ActiveModel::Callbacks
- ActiveModel::Dirty
- ActiveModel::Naming
- ActiveModel::Serialization
- ActiveModel::Validations

Rails4からActiveModel::Modelをincludeできるようになった。
ActiveModel::Modelを利用して定義したクラスは、フォームヘルパーに渡せる。なので、

- DBに依存（保存）しない入力フォームの構築
- 複数のActiveRecordオブジェクトに股がる情報をまとめて扱いたい時

に非常に便利。


## 9-4 値オブジェクト p.353

オブジェクトは「同一性」をどう捉えるかによって2種類に分けられる。

#### エンティティ

オブジェクトの同一性が重要な意味を持つもの。
例 : idなど

#### 値オブジェクト

何であるか。値が同じであればアプリケーション上は同一であると見なして良いオブジェクト。
例 : メールアドレス、住所など


### その文字列は本当に単なる文字列か

ActiveRecordオブジェクトは比較的単純なオブジェクトしか扱えない

String, Fixnum, Time, Date
しかし、それだけでは十分ではない

#### 単純な文字列による実装の例

```
class User < ActiveRecord::Base
 def same_prefecture?(other)
  prefecture == other.prefecture
 end

 def same_city?(other)
  city == other.city
 end
end
```

#### 値オブジェクトによる実装の例

```
class User < ActiveRecord::Base
 def address
   @address ||= Address.new(prefecture, city, house_number)
 end

 def address=(address)
  self.prefecture   = address.prefecture
  self.city         = address.city
  self.house_number = address.house_number
  @address = address
 end
end
```

値オブジェクトであるAddressクラス

```
class Address
 attr_accessor :prefecture, :city, :house_number

 def initialize(prefecture = nil, city = nil, house_number = nil)
  @prefecture   = prefecture
  @city         = city
  @house_number = house_number
 end

 def hash
  prefecture.hash + city.hash + house_number.hash
 end

 def ==(other)
  return false unless other.is_a?(Address)

  same_prefecture?(other) && same_city?(other) && same_house_number?(other)
 end

 def same_prefecture?(other)
   prefecture == other.prefecture
 end

 def same_city?(other)
  city == other.city
 end

 def same_house_number?(other)
  house_number == other.house_number
 end
end
```

- 値オブジェクトを使うことで、責任範囲を限定できる。
- また実装を再利用できる

### composed of

オブジェクトの属性値を値オブジェクトにマッピングする機能

```
class User < ActiveRecord::base
  composed_of :address, mapping: [%w(prefecture prefecture), %w(city city), %w(house_number house_number)]
end
```


## 9-5 Concern p.358

ある機能を実現するために必要な処理をConcernモジュールとして分離することで、  
モデルやコントローラから特定の機能のためにのみ利用されるメソッドやバリデーションの実装を分離できる。


#### 横断的関心事  (複数のモデルや機能にまたがって影響するようなもの)

- アプリケーション全体で利用されるロギング機能
- 変更に対する証跡の記録
- タグによる分類機能
- データの論理削除や、有効化・無効化処理
- 他のサービスやバックエンドプロセスと連携するためのシリアライズ

このような横断的関心事を実装する場合に、Concernモジュールとして実装する


#### ActiveSupport::Concern

記述を補助してくれる機能

モジュールに対して

- includedメソッドにブロックを渡す機能を追加。includeされた時にフックして実行する処理を直感的に記述できる
- クラスメソッドを追加するための記述を補助する
- モジュールの依存関係を管理

includeされたときの処理を分かりやすく記述  
簡単にクラスメソッドを追加する  
モジュール同士の依存関係を解決  


## 9-6. サービスクラス p.370

サービスクラスに抽出する処理とは

- 非常に複雑な処理
- 特定のエンティティや値オブジェクトに所属させると不自然になってしまう処理

目的: **モデルが過剰に複雑になることを防ぐこと**

### サービスクラスの特徴

- コントローラとモデルの中間のイメージ
- モデルが行う処理を取りまとめて実行するためのインターフェース
  - 例: 認証サービス、価格計算サービス
- 適切に設計されたサービスクラスは**状態をもたない**
- 業務知識と実装のメンタルモデルを一致させることも重要な役割の一つ
  - メンタルモデルとは、「業務に関わる複数の人がどう理解してるか」


サービスクラスはシステムが対象としている業務領域の知識と照らし合わせて、より自然な形でアプリケーションを設計するための手段。そのため、サービスクラスは**システムが対象としている業務に結びついた名前付け**をしておくのが良いとされている。


### サービスクラスの実装

銀行口座の振替処理の例。`BankAccountTransferService`というクラスを実装する。詳細は書籍を参照。


**TODO** p.374の上部にある記述で何が不自然なのかわからない。

```
このように複数の役割を持つエンティティが登場し、それを1つのトランザクションとしてまとめて表現するのが
自然である時などはサービスクラスが有効です。BankAccountに直接実装しようとした場合、トランザクションの起点
をインスタンスメソッドとして配置するのは不自然です。
```

=> わかった！その**インスタンス**（あるユーザーのBankAccount）のメソッドとしてfromとtoを取って振替処理をするのは不自然っていう意味だ。なんで特定のインスタンスが振替処理ができるの？っていうこと。

一方でクラスメソッドとして実装すると、特定の機能にのみ関わる振る舞いがクラス全体で必要な振る舞いであるかのように見えてしまう。そして機能が増えていけばどんどんBankAccountモデルが肥大化する。

### サービスクラスを利用する際の注意点

- 本来モデルにあるべき振る舞いまでサービスクラスに抽出するのはNG
- 手続き的な記述で再利用性を低下させない。サービスは処理の大枠を表現し、より詳細なレベルの振る舞いは各モデルが責任を持つべき

例えば、振替処理に口座間の通貨単位が異なる場合の処理を追加することを考える。

- サービスクラスに為替調整の機能を追加するのはNG
- Moneyオブジェクトや為替レートを扱うモデルにその処理を実装して、サービスからは透過的に扱えるようにする方が望ましい




## 9-7. 終わりに p.375

アプリケーションを設計し開発する上で大事なことは、自分で何が適切なのかを考えて選択し続けていくことだと思います。そして、自分が置かれている状況によって適切な手段は変わってきます。より良い選択をするためには知識が必要です。  

この章で紹介してきたことは、あらゆる状況に対応できる銀の弾丸のような解決策ではなく数多く存在する選択肢の1つに過ぎません。しかし、この章で紹介して来たテクニックや考え方を知っておくことは、自分が良い選択をするための力になってくれるでしょう。


## COLUMN: 単一テーブル継承について - typeカラムが使えないわけ p.375

- ActiveRecordで継承のためSTI(Single Table Inheritance)をサポートしている
- STIを利用するためにtypeカラムを使う
- typeカラムがあるとActiveRecordは自動的にSTIであると判断してしまう

備考: STIを使ったモデルが3つ(親1つ、子2つ)あった場合、テーブルは1つになりtypeカラムで区別する

参考:
[【Rails】ActiveRecord：単一テーブル継承(sti)とポリモーフィック関連を未だにぱっと思い出せないのでまとめ。 - 記すに足らず。](http://shirusu-ni-tarazu.hatenablog.jp/entry/2012/11/04/173742)
