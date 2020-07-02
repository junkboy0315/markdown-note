# ドメイン駆動設計入門

[[toc]]

## ドメイン駆動設計とは

- ドメイン --- プログラムを適用する対象となる領域
- ドメインモデル --- 現実の事象あるいは概念を抽象化した概念
- ドメインオブジェクト --- モデルをソフトウェア上で動作するモジュールとして表現したもの

これらは、

- ドメインエキスパートと開発者が協力して作っていく
- 必要に応じて適宜取捨選択を行いながら作る
- これらは互いに影響しあい常に反復的に変化していくべき

## 値オブジェクト

- システムならではの値の表現
- プリミティブな値が持つ性質がそのまま適用される

### 特徴

- 不変
  - 一度作成したら値自身を変更することはできない
- 交換が可能
  - 変数への代入により値を交換することはできる
- 等価性によって比較する
  - 属性によって識別される
  - `equals`メソッド等により値同士の透過性を比較できる

### 値オブジェクトにする基準

- ルールが存在しているか
  - ルールがあるなら値オブジェクトにそのルールを語らせるべき
- 単体で取り扱いたいか

### ふるまい

- 値オブジェクトは（単に値を保持するだけでなく）ふるまいも持つことができる。
- ふるまいにより自身に関するルールを語る
- 結果は新たな値として返却される

### メリット

- 表現力が増す
  - ただのプリミティブな値では何も分からない
- 値が正しいことを保証できる
  - コンストラクタでバリデーションして不正な値を防げる
- 誤った代入を防げる
  - 値オブジェクトは型を持つので不正な代入はできない。プリミティブな値ならできてしまう。
- ロジックが一箇所にまとまる

## エンティティ

### 特徴

- 可変である
  - ふるまい(メソッド)により値を変化させる。メソッドにルールを語らせる。
  - 全ての属性を可変にする必要はないので注意
- 同一性により区別される
  - 識別子(id)により区別する
  - 仮に全ての属性が全く同じであっても区別される

### エンティティにする基準

- ライフサイクル・連続性があるか
- 同じ概念を表していても、システムによっては値オブジェクトがいい場合もあれば、エンティティが良い場合もある。

## ドメインオブジェクトのメリット

「値オブジェクト」や「エンティティ」といったドメインオブジェクトを定義するメリット

- ドキュメント性が高まる
  - 開発者はコードを手がかりにしてルールを知ることができる
- ドメインにおける変更をコードに伝えやすくする
  - ただのデータ構造体ならそうはいかない

## ドメインサービス

オブジェクト自身に実装すると不自然なふるまいは、ドメインサービスに記載する。

- クラスメソッドのようなもの
- ステートレスである
- 何でもかんでもサービスに記載するな。「不自然」なものだけ記載せよ。
- 命名規則は`UserDomain.Services.XxxxService`などがおすすめ

## リポジトリ

- 主に DB レイヤーを抽象化するもの
- 「保存(永続化)」と「復元(再構築)」のみを扱う。それ以外は扱わない。
- **インターフェース**として定義される
  - そうすることで DB 操作の詳細を隠蔽するとともに複数の DB 等を切り替えられるようにする。

### ふるまい

- 永続化(ドメインオブジェクトを引数に取る)
- 破棄(ドメインオブジェクトを引数に取る)
- 再構築(id 等を引数にとる。ドメインオブジェクトを返す)

## アプリケーションサービス

- ユースケースを実現するためのサービス
  - 例えば「ユーザ」というドメインのアプリケーションサービスであれば、ユーザを登録する、取得する、変更するなどの処理をまとめて記載する
- 以下を組み合わせて作る
  - 値オブジェクト
  - エンティティ
  - ドメインサービス
  - リポジトリ
- **インターフェース**を用意することで、モック等が可能になり開発を進めやすくなるので、必要に応じて検討する

### 取得処理での注意点

- ドメインオブジェクト自体を返却すると意図せずロジックが流出する場合があるので、**必要なデータだけ返却する**のがおすすめ（データ転送用オブジェクト(DTO)を使うなど)

### 更新処理での注意点

- 更新したい項目が増減する場合は、引数が頻繁に増減するのを防ぐため、**コマンドオブジェクト**に集約すると良い
- コマンドオブジェクトは「処理のファサード」と呼ばれる（複雑な処理を単純にまとめる）
- ドメインのルールをここに記載しない。ドメインサービス等に記載すること。

### 凝集度

- 責任範囲がどれだけ集中しているか
- LCOM(Lack of Cohesion in Methods)という測定方法がある。すべてのインスタンス変数はすべてのメソッドで使われるべき、という考え方。
- 凝集度が低いときはユースケース（登録、変更、削除）ごとにサービスを分割することも検討する。この際、ドメインとしてのまとまりがわかりにくくなるので、パッケージを利用することで関連サービスを一つのフォルダにまとめておくと良い。

## サービスとは

- 活動や行動である
- 状態を持たない。また、状態を変えるためのふるまいを持たない。
- ドメインサービス
  - ドメインにおける活動
  - ドメイン知識を持つ
- アプリケーションサービス
  - 利用者の目的（ユースケース）を解決するためにある
  - アプリケーションとして成り立たせるためのサービス
- ドメインサービスとアプリケーションサービスは対象となる領域が異なるものの、本質的には同じもの

## 依存関係のコントロール

### 依存関係逆転の法則

- 上位レベルのモジュールは下位レベルのモジュールに依存してはならない、どちらも抽象(Interface)に依存すべし
- 抽象は実装の詳細に依存してはならない。実装の詳細が抽象に依存すべし。

一般的に抽象型はそれを利用するクライアント（上位モジュール）が要求する定義である。つまり、**方針の主導権を上位モジュールに握らせる**ということ。

重要なドメインのルールは、上位モジュールに含まれる。低レベルの変更（DB 変更）がドメインルールに影響を与えるようなことがあってはならない。

### 依存関係の管理

- Service Locator パターン
  - アンチパターン
  - 依存対象を取得する util のようなもの
  - サービスのコンストラクタ等において呼び出して使う
- IoC Container (DI Container) パターン
  - (依存関係解消済みの)サービス自体を取得するための util のようなもの
  - 言語によってはデフォルトで用意されている