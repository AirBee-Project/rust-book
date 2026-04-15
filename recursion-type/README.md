# 型引数を理解する

特定の型に縛られず、どんな型でも扱える「型引数」を作成することができる。この仕組みをジェネリクスと呼ぶ。

## 基本的な使い方

基本的には`Struct`でも`Enum`でも使い方は変わらない。型名の後に`<T>`を書いて、型引数にする。また`T`は任意の大文字から構成される文字列なら何でもよいが、型引数が1つしかないときは大抵`T`が使われる。(TypeのT)

### Structに実装する例

```rs
struct Storage<T> {
    value: T,
}

fn main() {
    let int_store = Storage { value: 100 };
    let str_store = Storage { value: "Rust" };
}
```

### Enumに実装する例

通知する内容がなんだろうと、`Notification`型に入れることができる。

```rs
// チャットメッセージ
#[derive(Debug)]
struct Message {
    user: String,
    text: String,
}

// カレンダーの予定
#[derive(Debug)]
struct CalendarEvent {
    title: String,
    minutes_to_start: i32,
}

// 通知の状態を定義
#[derive(Debug)]
enum Notification<T> {
    Unread(T), // 未読
    Read(T),   // 既読
    Dismissed, // 削除済み
}

fn main() {
    // メッセージの通知
    let msg = Message {
        user: "田中".to_string(),
        text: "今どこ？".to_string(),
    };

    // カレンダーの通知
    let event = CalendarEvent {
        title: "会議".to_string(),
        minutes_to_start: 10,
    };

    let n1 = Notification::Unread(msg);
    let n2 = Notification::Unread(event);
}

```

### 関数に実装する例

`fn swap`はTがどのような型であろうと、順番を入れ替えることができる。

```rs
fn swap<T>(a: T, b: T) -> (T, T) {
    (b, a)
}

fn main() {
    // 整数の入れ替え
    let (x, y) = swap(10, 20);
    println!("x: {}, y: {}", x, y);

    // 文字列の入れ替え
    let (s1, s2) = swap("Hello", "World");
    println!("s1: {}, s2: {}", s1, s2);
}

```

## 型引数にTraitに基づいた制限をかける

上記の例では`T`は何の制限もついていないので、`String`,`Vec`,`u32`,etc..どのような型でも受け入れられてしまう。しかし、型に実装したい処理内容によっては、Traitを強制したい場合があるだろう。

### 基本の例

例えば、下記の例ではコマンドプロンプト上でラベルような四角を表示する例です。`T`は`Display` Traitを実装している必要がある。

```rs
use std::fmt::Display;

fn print_label<T>(item: T)
where
    T: Display,
{
    println!("┌──────┐");
    println!("  {}  ", item);
    println!("└──────┘");
}

fn main() {
    print_label(100);
    print_label("Rust");
    print_label(3.14);
}

```

### 複数のTraitを引数の条件とする場合

`fn calculate_checkout`はどのような`T`をも受け入れているのではなく、掛け算、足し算ができる型のみを引数として取る。そのため、国によって（すなわち、数値型ごとに）ロジックを書き直す必要がない。

```rs
use std::ops::{Add, Mul};

/// (単価 * 数量) + 配送料
fn calculate_checkout<T>(price: T, quantity: T, shipping_fee: T) -> T
where
    T: Mul<Output = T> + Add<Output = T>,
{
    (price * quantity) + shipping_fee
}

fn main() {
    // 日本：円(i32)での決済
    let jp_total = calculate_checkout(500, 3, 200);
    println!("日本国内の合計: {}円", jp_total);

    // 海外：ドル(f64)での決済
    let us_total = calculate_checkout(12.50, 2.0, 5.99);
    println!("海外発送の合計: ${:.2}", us_total);
}
```

### Kasane-Logicを用いた例

```Cargo.toml
[dependencies]
kasane-logic = "0.1.3"
```

このバージョンでは`SingleId`と`RangeId`に同じ`SpatialId` Traitが実装されている。

```rs
/// 空間 ID が備えるべき基礎的な性質および移動操作を定義するトレイト。
pub trait SpatialId: Block + FlexIds {
    //そのIDの各次元の最大と最小を返す
    fn min_f(&self) -> i32;
    fn max_f(&self) -> i32;
    fn max_xy(&self) -> u32;

    //各インデックスの移動
    fn move_f(&mut self, by: i32) -> Result<(), Error>;
    fn move_x(&mut self, by: i32);
    fn move_y(&mut self, by: i32) -> Result<(), Error>;

    //各次元の長さを取得するメソット
    fn length_f_meters(&self) -> f64;
    fn length_x_meters(&self) -> f64;
    fn length_y_meters(&self) -> f64;

    //中心点の座標を求める関数
    fn spatial_center(&self) -> Coordinate;

    //頂点をの座標を求める関数
    fn spatial_vertices(&self) -> [Coordinate; 8];

    //時間が関連するもの
    fn temporal(&self) -> &TemporalId;
    fn temporal_mut(&mut self) -> &mut TemporalId;

    //SingleIdとして書き出すもの
    fn single_ids(&self) -> impl Iterator<Item = SingleId>;
    fn optimize_single_ids(&self) -> impl Iterator<Item = SingleId>;
}
```

つまり、引数に対して`SpatialId`を強制するれば、対象が`SingleId`なのか`RangeId`なのかを問わずに処理を行うことができる。

```rs
use kasane_logic::{RangeId, SingleId, SpatialId};

fn print_center<T>(id: T)
where
    T: SpatialId,
{
    let center = id.spatial_center();
    println!(
        "Center : {}/{}/{}",
        center.as_altitude(),
        center.as_latitude(),
        center.as_longitude()
    );
}

fn main() {
    let id1 = SingleId::new(5, 3, 2, 10).unwrap();
    let id2 = RangeId::new(4, [-3, 6], [8, 9], [5, 10]).unwrap();

    print_center(id1);
    print_center(id2);
}
```

## `Option`型や`Result`型はただのEnumジェネリクス

```rs
enum Option<T> {
    Some(T),
    None,
}

enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

# 再帰的な型を理解する

## 2分岐探索を作ってみよう

こういう型を書けばよいと思う。

```rs
enum Node<T>
where
    T: PartialOrd,
{
    Leaf {
        value: T,
        left: Option<Node<T>>,
        right: Option<Node<T>>,
    },
    Empty,
}
```

ところが、コンパイルしようとするとエラーが発生する。

```sh
  |
1 | enum Node<T>
  | ^^^^^^^^^^^^
...
7 |         left: Option<Node<T>>,
  |                      ------- recursive without indirection
  |
help: insert some indirection (e.g., a `Box`, `Rc`, or `&`) to break the cycle
  |
7 |         left: Option<Box<Node<T>>>,
  |                      ++++       +
```

Rustはコンパイル時に型のサイズを確定させる必要があるが、再帰的な型は無限にネストできてしまうため、エラーが出る。それを防ぐため再帰部分を`Box`型で囲むことによりエラーを防ぐことができる。

なぜ`Box`型によりサイズを確定させることができるのかはヒープとスタック、そしてポインタについて学ばなけらばわからないので、ここでは深くは触れない。

Box型を導入した新しい型は以下の通りだ。これならエラーが出ない。

```rs
enum Node<T>
where
    T: PartialOrd + PartialEq,
{
    Leaf {
        value: T,
        left: Option<Box<Node<T>>>,
        right: Option<Box<Node<T>>>,
    },
    Empty,
}
```

### 挿入する関数

```rs
impl<T> Node<T>
where
    T: PartialOrd + PartialEq,
{
    pub fn insert(&mut self, target: T) {
        match self {
            Node::Empty => {
                *self = Node::Leaf {
                    value: target,
                    left: None,
                    right: None,
                };
            }
            Node::Leaf { value, left, right } => {
                if target < *value {
                    match left {
                        Some(child_node) => child_node.insert(target),
                        None => {
                            *left = Some(Box::new(Node::Leaf {
                                value: target,
                                left: None,
                                right: None,
                            }));
                        }
                    }
                } else {
                    match right {
                        Some(child_node) => child_node.insert(target),
                        None => {
                            *right = Some(Box::new(Node::Leaf {
                                value: target,
                                left: None,
                                right: None,
                            }));
                        }
                    }
                }
            }
        }
    }
}
```

## 検索する関数

```rs
impl<T> Node<T>
where
    T: PartialOrd + PartialEq,
{
    fn contains(&self, target: &T) -> bool {
        match self {
            Node::Leaf { value, left, right } => {
                if value == target {
                    true
                } else if target < value {
                    match left {
                        Some(node) => node.contains(target),
                        None => false,
                    }
                } else {
                    match right {
                        Some(node) => node.contains(target),
                        None => false,
                    }
                }
            }
            Node::Empty => false,
        }
    }
}

```

## 木構造を表示する関数

```rs
impl<T> Node<T>
where
    T: PartialOrd + std::fmt::Display,
{
    pub fn print_structure(&self, depth: usize) {
        match self {
            Node::Leaf { value, left, right } => {
                if let Some(r) = right {
                    r.print_structure(depth + 1);
                }

                let indent = "    ".repeat(depth);
                println!("{}{}", indent, value);

                if let Some(l) = left {
                    l.print_structure(depth + 1);
                }
            }
            Node::Empty => {}
        }
    }
}

```

## 動かしてみよう

```rs
fn main() {
    let mut tree = Node::Empty;
    let values = [50, 30, 80, 20, 40, 70, 90, 10, 20, 50, 11, 0, 3, 6];
    for v in values {
        tree.insert(v);
    }
    tree.print_structure(0);
}
```
