[![License](http://img.shields.io/:license-apache%202.0-brightgreen.svg)](http://www.apache.org/licenses/LICENSE-2.0.html)
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/io.debezium/debezium-parent/badge.svg)](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22io.debezium%22)
[![Build Status](https://travis-ci.org/debezium/debezium.svg?branch=master)](https://travis-ci.org/debezium/debezium)
[![User chat](https://img.shields.io/badge/chat-users-brightgreen.svg)](https://gitter.im/debezium/user)
[![Developer chat](https://img.shields.io/badge/chat-devs-brightgreen.svg)](https://gitter.im/debezium/dev)
[![Google Group](https://img.shields.io/:mailing%20list-debezium-brightgreen.svg)](https://groups.google.com/forum/#!forum/debezium)
[![Stack Overflow](http://img.shields.io/:stack%20overflow-debezium-brightgreen.svg)](http://stackoverflow.com/questions/tagged/debezium)

Copyright Debezium Authors.
Licensed under the [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0).
The Antlr grammars within the debezium-ddl-parser module are licensed under the [MIT License](https://opensource.org/licenses/MIT).

English | [Chinese](README_ZH.md)

# Debezium
Debezium は変更データキャプチャ (Change data capture; CDC) デザインパターンのための低遅延データスストリーミングプラットフォームを提供する、オープンソースプロジェクトです。
Debezium を利用して、あなたの Database を監視するように設定すれば、そしてあなたが書いたアプリケーションがデータベースに適用された低レベルの各行の変更イベントに対して処理を実行できるようになります。
このとき、コミットされた変更だけが見えるので、アプリケーションはトランザクションや変更がロールバックされることを気にする必要がなくなります。
Debezium は各種のデータベースに対して統一されたモデルを持っています、アプリケーションはデータベースの種類の差を気にする必要がありません。
更に、Debezium はデータ変更の履歴を耐久性があって複製されたログに保存するので、アプリケーションはどんなときでも停止・再起動できるようになり、起動していない間に処理できなかった変更も処理できるようになり、全てのイベントが正しく完全に処理される事を保証できます。

<!-- 
Debezium is an open source project that provides a low latency data streaming platform for change data capture (CDC). You setup and configure Debezium to monitor your databases, and then your applications consume events for each row-level change made to the database. Only committed changes are visible, so your application doesn't have to worry about transactions or changes that are rolled back. Debezium provides a single model of all change events, so your application does not have to worry about the intricacies of each kind of database management system. Additionally, since Debezium records the history of data changes in durable, replicated logs, your application can be stopped and restarted at any time, and it will be able to consume all of the events it missed while it was not running, ensuring that all events are processed correctly and completely.
-->

データベースを監視して、データ変更を通知することは、常に複雑なタスクでした。
リレーショナルデータベースのトリガーは便利ですが、それぞれのデータベースに固有のもので、多くの場合同じデータベース内の状態を更新することに限定されています（データベース外と通信することはできません）。
いくつかのデータベースは、変更を監視できる API を提供しています、しかし標準の方法は無いため、それぞれのデータベースでアプローチが異なり、利用には多くの知識と特殊なコードを必要とします。
データベースの全ての変更を確実に、同じ順番で、データベースに与える影響を最低限にしながらモニターすることは、今でも非常に難しい事です。

<!--
Monitoring databases and being notified when data changes has always been complicated. Relational database triggers can be useful, but are specific to each database and often limited to updating state within the same database (not communicating with external processes). Some databases offer APIs or frameworks for monitoring changes, but there is no standard so each database's approach is different and requires a lot of knowledged and specialized code. It still is very challenging to ensure that all changes are seen and processed in the same order while minimally impacting the database.
-->

Debezium はこの作業を代わりに実行するモジュールを提供しています。いくつかのモジュールは汎用的で、複数のデータベースシステムで利用できますが、機能と性能の面で制限があります。
他のモジュールは特定のデータベースに特化していて、遙かに高性能で、システム固有の機能を活用しています。

<!--
Debezium provides modules that do this work for you. Some modules are generic and work with multiple database management systems, but are also a bit more limited in functionality and performance. Other modules are tailored for specific database management systems, so they are often far more capable and they leverage the specific features of the system.
-->

## Basic architecture - アーキテクチャ概要

Debezium は変更データキャプチャ（Change data capture; CDC) プラットフォームであり、Kafka と Kafka Connect を利用することで、耐久性、信頼性、耐障害性を実現しています。
Kafka Connect の上に Deploy された各 connector は、分散し、スケーラブルで、耐障害性のある形でそれぞれ一つのデータベースサーバーを監視して、全ての変更を取得してそれぞ単一または複数の Kafka topic に保存します。（通常は、データベーステーブル毎に1つの topic を作ります）。
Kafka は、全ての変更データイベントが同じ順序でレプリケーションされることと、複数のクライアントがそれぞれ独立に upstream のシステムに与える影響を最小限にしながらそれを利用する事を可能にしてくれます。
更に、client はデータの利用を任意のタイミングでストップすることもできて、再開したときには正確にその場所から処理を再開することができます。
各クライアントは要望によって、全てのデータを正確に1回ずつ(exactly-once)か少なくとも1回 (at-least-once)処理するかを選択することができます。また、データベース/テーブルに対する全てのデータ変更イベントは、データベースで発生したのと正確に同じ順番に配信されます。

<!--
Debezium is a change data capture (CDC) platform that achieves its durability, reliability, and fault tolerance qualities by reusing Kafka and Kafka Connect. Each connector deployed to the Kafka Connect distributed, scalable, fault tolerant service monitors a single upstream database server, capturing all of the changes and recording them in one or more Kafka topics (typically one topic per database table). Kafka ensures that all of these data change events are replicated and totally ordered, and allows many clients to independently consume these same data change events with little impact on the upstream system. Additionally, clients can stop consuming at any time, and when they restart they resume exactly where they left off. Each client can determine whether they want exactly-once or at-least-once delivery of all data change events, and all data change events for each database/table are delivered in the same order they occurred in the upstream database.
-->

耐障害性、パフォーマンス、スケーラビリティを必要としないアプリケーションは、代わりに Debezium の *組み込みデータエンジン(embedded connecgtor engine)* を利用すれば、アプリケーションコードの中で直接コネクタを実行して変更をキャプチャすることができます。同じ変更データイベントですが、Kafka 永続化等のために Kafka を経由することなく、コネクタが直接アプリケーションにイベントを送信します。

<!--
Applications that don't need or want this level of fault tolerance, performance, scalability, and reliability can instead use Debezium's *embedded connector engine* to run a connector directly within the application space. They still want the same data change events, but prefer to have the connectors send them directly to the application rather than persist them inside Kafka.
-->

## Common use cases - よくあるユースケース

Debezium が非常に価値のあるシナリオはいくつかありますが、ここではその中でも特に一般的なものを紹介します。

<!--
There are a number of scenarios in which Debezium can be extremely valuable, but here we outline just a few of them that are more common.
-->

### Cache invalidation - キャッシュの無効化

レコードが変更や削除されたらすぐに、自動的にキャッシュの無効化ができます。キャッシュが分離したプロセス内（例：Redis, Memcache, Inifinispan 等）で動いていても、キャッシュの無効化ロジックを分離したプロセスやサービス内に配置して、メインのアプリケーションを簡素化することができます。
状況によっては、もう少し洗煉された手法を使い、更新イベントで取得された更新後のデータを利用して、キャッシュエントリを事前に upload しておくこともできます。

<!--
Automatically invalidate entries in a cache as soon as the record(s) for entries change or are removed. If the cache is running in a separate process (e.g., Redis, Memcache, Infinispan, and others), then the simple cache invalidation logic can be placed into a separate process or service, simplifying the main application. In some situations, the logic can be made a little more sophisticated and can use the updated data in the change events to update the affected cache entries.
-->

### Simplifying monolithic applications - モノリシックアプリケーションをシンプルに

いくつかのアプリケーションが同じデータベースを更新する場合において、更新後に追加の処理をしたいことがあります。検索 Index を Update するだとか、キャッシュをアップデートする、通知を送る、ビジネスロジックを実行する、等です。このケースは "デュアルライト (dual-writes)" と呼ばれることもあります、アプリケーションが単一トランザクションでカバーしきれない複数のシステムにデータを書き込むからです。
デュアルライトは、アプリケーションロジックが複雑化してメンテナンスしにくくなるだけでなく、アプリケーションが書き込みをコミットしたが、他の書き込みをする途中でクラッシュすると、データをロストしたり各種のデータ不整合を発生させる危険性があります。
変更データキャプチャを使う事で、このような処理は変更がデータベースにコミットされた後に、他のスレッド/プロセス/サービスで行えるようになります。この方法をとることで、障害への耐性があがり、変更をミスすることも無く、よりスケールしやすく、簡単にアップグレード・オペレーションができるようになります。

<!--
Many applications update a database and then do additional work after the changes are committed: update search indexes, update a cache, send notifications, run business logic, etc. This is often called "dual-writes" since the application is writing to multiple systems outside of a single transaction. Not only is the application logic complex and more difficult to maintain, dual writes also risk losing data or making the various systems inconsistent if the application were to crash after a commit but before some/all of the other updates were performed. Using change data capture, these other activities can be performed in separate threads or separate processes/services when the data is committed in the original database. This approach is more tolerant of failures, does not miss events, scales better, and more easily supports upgrading and operations.
-->

### Sharing databases - Database をシェアする

複数のアプリケーションが同じデータベースを共有するとき、片方のアプリケーションが、もう片方のアプリケーションの変更を知ることは非常に難しい問題になります。1つの方法にメッセージバスを使うことですが、データベースとのトランザクションが無いメッセージバスの利用は先ほどのデュアルライトと同じ問題に悩まされます。Debezium を使う事でこの問題はシンプルに解決できて、それぞれのアプリケーションがデータベースを監視して、変更に対応すれば問題ありません。

<!--
When multiple applications share a single database, it is often non-trivial for one application to become aware of the changes committed by another application. One approach is to use a message bus, although non-transactional message busses suffer from the "dual-writes" problems mentioned above. However, this becomes very straightforward with Debezium: each application can monitor the database and react to the changes.
-->

### Data integration - データの統合

複数の場所にまたがってデータが保存されることはよくあります、特に活用される目的が異なっている場合は、その保存方法が少しづつ異なる事も多いです。
複数のシステムを同期し続けることは非常に困難なことですが、単純な ETL 形式のソリューションは、Debezium とシンプルなイベント処理ロジックを使って非即に開発することができます。

<!--
Data is often stored in multiple places, especially when it is used for different purposes and has slightly different forms. Keeping the multiple systems synchronized can be challenging, but simple ETL-type solutions can be implemented quickly with Debezium and simple event processing logic.
-->

### CQRS

[コマンド クエリ責務分離 (Command Query Responsibility Separation; CQRS)](http://martinfowler.com/bliki/CQRS.html) アーキテクチャパターンは、更新と読み取りで別々のデータモデルを利用します。変更が更新側で記録されると、その変更が処理され、変更が色々な読み取り表現を更新するために利用されます。結果として、よくCQRSアプリケーションは複雑になります、それらが信頼性のあって順序保証された処理を必要とする場合は更に。
Debezium と変更データキャプチャ (CDC) はこれらをもっと親しみやすくします： 書き込みをいつも通り実行すれば、Debezium がそれらを耐障害性がある正しい順序のストリームとしてキャプチャして、ストリームが非同期に読み取り専用ビューをアップデートするためにコンシュームされます。
書き込み側のテーブルはドメイン指向のエンティティを持ったり、CQRS が書き込み側のテーブルを追記専用のコマンド履歴につかう [Event Sourcing](http://martinfowler.com/eaaDev/EventSourcing.html) と共に使われる事もあります。


<!--
The [Command Query Responsibility Separation (CQRS)](http://martinfowler.com/bliki/CQRS.html) architectural pattern uses a one data model for updating and one or more other data models for reading. As changes are recorded on the update-side, those changes are then processed and used to update the various read representations. As a result CQRS applications are usually more complicated, especially when they need to ensure reliable and totally-ordered processing. Debezium and CDC can make this more approachable: writes are recorded as normal, but Debezium captures those changes in durable, totally ordered streams that are consumed by the services that asynchronously update the read-only views. The write-side tables can represent domain-oriented entities, or when CQRS is paired with [Event Sourcing](http://martinfowler.com/eaaDev/EventSourcing.html) the write-side tables are the append-only event log of commands.
-->

## Building Debezium
See english READMe.md 

## Contributing

The Debezium community welcomes anyone that wants to help out in any way, whether that includes reporting problems, helping with documentation, or contributing code changes to fix bugs, add tests, or implement new features. See [this document](CONTRIBUTE.md) for details.

A big thank you to all the Debezium contributors!

<a href="https://github.com/debezium/debezium/graphs/contributors">
  <img src="https://contributors-img.web.app/image?repo=debezium/debezium" />
</a>
