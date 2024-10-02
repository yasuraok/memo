# perlメモ

## 1. packageとuseの関係

### 1.1. useの使い方
`package`は、Perlのコードを名前空間に分けるために使用されます。名前空間を使うことで、変数やサブルーチンの名前が他の名前空間と衝突するのを防ぐことができます。例えば、以下のように`MyModule`という名前空間を定義します。`package`で定義されたモジュール（名前空間）内では、そのモジュールに属するコードを記述します。モジュールの最後には通常、成功を意味する`1;`を返します。

```perl
package MyModule;

sub hello {
    return "Hello, World!";
}
1;
```

一方、`use`は別のモジュールをインポートし、そのモジュールの機能を利用可能にするために使用します。`use`はコンパイル時に指定したモジュールを読み込み、インポートも同時に行います。例えば、先ほどの`MyModule`を読み込むには以下のようにします。

```perl
use MyModule;

print MyModule::hello(); # Hello, World!
```

`use`を使うことで、指定したモジュールのサブルーチンや変数を利用できるようになります。モジュールは`::`（二重コロン）を使ってアクセスします。

### 1.2. useの詳細
Perlスクリプトでの`use`は以下のコードと等価です。

```perl
BEGIN {
    require ModuleName;
    ModuleName->import( LIST );
}
```

- `BEGIN`ブロックはそのスコープがコンパイルされるとすぐに実行される特殊なブロックです。つまり、useはコンパイル時にrequireとModuleName->importを実行します。
- `require`はモジュールを@INCから検索し読込み、コンパイルを実施します。つまり、requireをした時点でもモジュールのオリジナルの名前空間を指定してモジュール内の関数や変数を参照することができます。
- `import`については次節で解説します。

### 1.3. importの詳細
`import`関数はpackage側で独自に実装できますが、一般的にはモジュールの名前空間で定義された関数や変数を呼び出し元のパッケージの名前空間に展開する、という処理を書きます。C++でいう所のusing namespaceと似ています。また、この処理はExportモジュールのimport関数で実装されているので、これをuseすれば自作しなくて良いことになります。

```perl
package YourModule;

use Exporter 'import'; # このpackageにimport関数が配置され、useされたときに呼ばれる

# import関数内で使われる変数 @EXPORTはuse YourModuleしたときの展開対象。@EXPORT_OKはuse YourModule 'funcZ'したときの展開対象
our @EXPORT = qw/funcX funcY/;
our @EXPORT_OK = qw/funcZ/;
```

言い換えれば、**このようにimport関数そのものやuse Exporterと@EXPORT(_OK)が定義されていないpackageでは、useしても「呼び出し元のパッケージの名前空間に展開する」という処理は起きない**ことになります。「aをuseしたbをさらにuseしたcでa内の関数が (名前空間指定なしで) 呼べるのか」という疑問の答えは「そのようなimportを実装していない限りNo」ということになります。

```perl
package MyModule;

sub hello {
    print "Hello, World!\n";
}

package main;

# hello(); # Undefined subroutine &main::hello called
MyModule::hello();
```

### 1.4. 動的なモジュールロード
動的にモジュールをロードする (モジュール名をプログラム中で動的に指定してロードする) には `Module::Load` を使います。

```perl
use Module::Load;

# ロードしたいモジュール名を動的に決定
my $module_name = "Some::Module";
load $module_name;

# useと異なりimportメソッドが明示的に呼ばれないので、明示的に呼び出す
$module_name->import();

print $module_name->some_function();
```

従来はUniversal::requireというモジュールが使われていましたが、現在ではPerl 5.9.4からコアモジュールとなっている `Module::Load` の方が一般的になっています。


## 2. perlのclassの実体について

### 2.1. 原則
[クラスは単なるパッケージ、メソッドは単なるサブルーチン](https://perldoc.jp/docs/perl/5.8.8/perlobj.pod)、インスタンスは基本的にただのハッシュリファレンス、と考えることができます。通常のパッケージ・関数 (サブルーチン) 呼び出しと比べて本質的に新しいことは以下の2点のみです。

#### 2.1.1. アロー演算子によるサブルーチン呼び出し
アロー演算子を用いてサブルーチンを呼び出すと、[「矢印の左側に何があるか、リファレンスかクラス名か」が最初の引数として サブルーチンに渡されます](https://perldoc.jp/docs/perl/5.8.8/perlobj.pod#Method32Invocation)。つまり、以下のaとbは同じ処理になります。

```perl
# a. アロー演算子呼び出し
my $fred = Critter->find("Fred");
$fred->display("Height", "Weight");

# b. 第一引数にPackage名やリファレンスを明示して呼び出し
my $fred = Critter::find("Critter", "Fred");
Critter::display($fred, "Height", "Weight");
```

この構文により、`{Package名}->{メソッド名}`という呼び出しは (第一引数にPackage名が得られるので) 後発OOP言語におけるクラスメソッドに相当する処理を実装することができます。したがって、メソッド名にnewのような名前をつけ、インスタンスのようなものをreturnすれば、このPackageがクラスとなりnewがコンストラクタになる、という理屈です。

#### 2.1.2. blessによる関連付け
`bless` を使用することで、ただのリファレンスにクラス名を関連付けることができます。

```perl
package MyClass;

sub new {
    my ($class, %hash) = @_;
    my $self = \%hash;   # 属性を格納するハッシュリファレンス
    bless $self, $class; # リファレンスをオブジェクトに変換
    return $self;
}
```

`bless`を通された「元々ハッシュだったもの」には以下の変化が起こります。

1.  refの結果がクラスを示すものになる: 

    ```perl
    my $ref = {};          # ハッシュリファレンス
    print ref($ref);       # 出力: HASH

    bless $ref, 'MyClass';
    print ref($ref);       # 出力: MyClass
    ```

2.  アロー演算子でPackageのメソッドを呼べるようになる: 

    ```perl
    package MyClass;

    sub new {
        my $class = shift;
        my $self = {};
        bless $self, $class;
        return $self;
    }

    sub method {
        print "Method called\n";
    }

    package main;

    my $object = MyClass->new();
    $object->method();      # 出力: Method called
    ```

    `bless`されていない場合はどこにあるサブルーチンを呼ぶのかが決まらないので「Can't call method "method" on unblessed hash reference」のエラーとなるが、blessされていれば第一引数に$objectが渡された状態でMyClassのmethodが呼べる。

3.  `isa`と`can`メソッドが使える: 

    ```perl
    my $object = MyClass->new();

    # オブジェクトやクラスが特定のクラスに属するかを確認
    if ($object->isa('MyClass')) {
        print "Object belongs to MyClass.\n";
    }

    # メソッドが呼べるか (定義されているか) を確認
    if ($object->can('method')) {
        $object->method();
    }
    ```

4.  そのpackageにDESTROYメソッドが実装されていれば、blessされたインスタンスがスコープを抜けたときに呼ばれる

    ```perl
    package MyClass;

    sub DESTROY {
        print "Object is being destroyed\n";
    }

    package main;

    {
        my $object = MyClass->new();
    }  # スコープを外れたときに DESTROY が呼び出される
    # 出力: Object is being destroyed
    ```

より厳密な実装については[perlobj](https://perldoc.jp/docs/perl/5.8.8/perlobj.pod)を参照してください。

## 3. 現実的なオブジェクト指向プログラミングの方法

上記を原則を踏まえた上で、今風の考え方でPerl上でオブジェクト指向プログラミングをした場合に、どういう手段があるかを紹介します。

### 標準のclass
perl v5.37でexperimentalに入ったclassを使うのが最も最新かつ標準の方法になります。が、既存のソースではまず見かけないので、従来の実装を知る必要があります。詳しくは[最近Perlに追加された実験的機能
try文⁠⁠、defer文⁠⁠、class文（1）](https://gihyo.jp/dev/serial/01/perl-hackers-hub/007901)を参照してください。

### 3.2. Moo
標準が決まるまでの間、たくさんのOOP用のライブラリができましたが、特にMooというものが有名なようです (過去にはMooseとかMouseとかいろいろあったらしい)。

これを用いた場合に前章で解説した原則から変わることは主に以下です。

#### 3.2.1. has

`has`は、オブジェクトの属性（プロパティ）を定義するためのキーワードです。`has`を使うことで、オブジェクトの属性を簡単に宣言できます。以下の例では、`name`と`age`という属性を定義しています。

```perl
package Person;
use Moo;

has 'name' => (
    is => 'rw',          # 読み書き可能（read-write）
    required => 1,       # 必須属性
);

has 'age' => (
    is => 'rw',          # 読み書き可能
);
```

#### 3.2.2. newメソッドの省略

Mooを使用する場合、新しいオブジェクトを作成する際に`new`メソッドを自分で定義する必要がありません。Mooが自動的に`new`メソッドを生成してくれるため、属性の初期化やインスタンスの生成が非常に簡単になります。

```perl
package main;

my $person = Person->new(name => 'John Doe', age => 30);
print $person->name;  # John Doe
print $person->age;   # 30
```

#### 3.2.3. extends

`extends`は、クラスの継承を可能にします。親クラスを指定することで、そのクラスの機能を子クラスが引き継ぐことができます。

```perl
package Employee;
use Moo;

extends 'Person';

has 'employee_id' => (
    is => 'rw',
    required => 1,
);

package main;
my $employee = Employee->new(name => 'Jane Smith', age => 40, employee_id => 'E123');
print $employee->name;       # Jane Smith
print $employee->employee_id; # E123
```

上記の例では、`Employee`クラスが`Person`クラスを継承しており、`employee_id`という新しい属性を追加しています。

#### 3.2.4. BUILDメソッド

Mooでは、オブジェクトの初期化処理をカスタマイズするために`BUILD`メソッドを定義できます。`BUILD`メソッドは、オブジェクトが作成された**後**に自動的に呼び出されます。

```perl
package Person;
use Moo;

has 'name' => (is => 'rw', required => 1);
has 'age' => (is => 'rw');

sub BUILD {
    my $self = shift;
    print "A new Person object was created.\n";
}

package main;

my $person = Person->new(name => 'John Doe', age => 30);
# 出力: A new Person object was created.
```
