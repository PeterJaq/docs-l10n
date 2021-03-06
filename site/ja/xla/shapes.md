# シェイプとレイアウト

XLA `shape` proto（[xla_data.proto](https://www.tensorflow.org/code/tensorflow/compiler/xla/xla_data.proto)） は、N次元配列（略して配列）のランクとサイズ、データ型について記述します。


## 用語、表記、規約

* 配列のランクは次元の数とおなじです。*正しい配列のランク* は、次元数が1以上の配列のことを指します。

* 次元は、`N` 次元配列について、`0` から `N-1` で番号づけされます。次元番号は、便宜的につけられたラベルです。次元の順序は、シェイプのレイアウトの優先順序（minor/major order）を意味しているわけではありません。レイアウトは、`Layout` protoによって決まります。

* 規約として、次元は次元番号の昇順にリスト化されます。たとえば、サイズが `[A x B x C]` の3次元配列は、次元0のサイズが `A`、次元1のサイズが `B`、次元2のサイズが `C` となります。

XLAにあるいくつかのユーティリティは、Pythonと同様に負のインデックスをサポートするものがあります。たとえば、次元-1は、最後の次元（`N` 次元配列では `N-1` とおなじ）を指します。
上で述べた3次元配列では、次元-1はサイズ `C`、次元-2はサイズ `B` をもつといったように。

* 2、3、4次元の配列は、次元と関連付けて特定の文字で明記されることがよくあります。たとえば、2次元配列では以下のようになります。
  * 次元0: `y`
  * 次元1: `x`
* 3次元配列では、以下のようになります。
  * 次元0: `z`
  * 次元1: `y`
  * 次元2: `x`
* 4次元配列では、以下のようになります。
  * 次元0: `p`
  * 次元1: `z`
  * 次元2: `y`
  * 次元3: `x`

* XLA APIにある次元を受け取る関数は、昇順の次元番号として扱います。これは、`initializer_list` として次元を渡したときとおなじになります。

たとえば、`ShapeUtil::MakeShape(F32, {A, B, C, D})` は、`[A, B, C, D]` で構成される配列の次元サイズからなるシェイプを構築します。

## レイアウト

`Layout` protoは、配列がメモリ上でどのように表現されるかを記述します。
`Layout` protoは、次に示すフィールドを含んでいます。

```
message Layout {
  repeated int64 minor_to_major = 1;
  repeated int64 padded_dimensions = 2;
  optional PaddingValue padding_value = 3;
}
```

### 次元の優先順序

必要なフィールドは、`minor_to_major` のみです。
このフィールドは、シェイプ内の次元の優先順序を示しています。
`minor_to_major` の値は、配列の次元順序（`N` 次元配列なら `0` から `N-1`）で、もっとも優先度の低い次元を示す最初の値から、もっとも優先度の高い次元を示す最後の値になります。
もっとも優先度の低い次元は、連続なメモリに配置された配列の要素を走査したときに、もっとも頻繁に変化する次元になります。

たとえば、サイズが `[2 x 3]` である、次に示す2次元配列を考えてみましょう。

```
a b c
d e f
```

ここで次元 `0` はサイズが2で、次元 `1` はサイズが3です。
レイアウト内の `minor_to_major` フィールドが `[0, 1]` の場合、次元 `0` はもっとも優先度の低い次元となり、次元 `1` はもっとも優先度が高い次元になります。
これは、連続なメモリでは、以下に示すようなレイアウトと一致します。

```
a d b e c f
```

`0` から `N-1` までの次元優先度順のことを、（ランクが2の場合に） *列優先* と呼びます。
巨大な次元順序を想定した場合、コード上のこのレイアウトにちなんで、別名"次元0が優先度が低い"と呼びます。

一方で、レイアウトの `minor_to_major` フィールドが `[1, 0]` の場合は、連続なメモリ内のレイアウトは次のようになります。

```
a b c d e f
```

`N-1` から `0` までの次元優先度順のことを、（ランクが2の場合に） *行優先* を呼びます。
巨大な次元順序を想定した場合、コード上のこのレイアウトにちなんで、別名"次元0の優先度が高い"と呼びます。


#### デフォルトの優先度順

新しく作られたシェイプのデフォルトのレイアウトは、"次元順が高いほうから低いほう"（ランク2の場合は行優先）になります。


## パディング

パディング量は、任意のフィールド `padded_dimensions` と `padding_value` に定義されています。
フィールド `padded_dimensions` は、それぞれの次元についてパディングするサイズ（幅）を示しています。
存在する場合は、`padded_dimensions` の要素数がシェイプのランクとおなじである必要があります。

たとえば、上で `[2 x 3]` の配列を紹介しましたが、もし `padded_dimensions` が `[3, 5]` ならば、次元0は幅3でパディングされ、次元1は幅5でパディングされます。
（パディング量0、列優先のレイアウトを想定した場合、）連続なメモリのレイアウトは、次のようになります。

```
a d 0 b e 0 c f 0 0 0 0 0 0 0
```

これは、おなじ次元優先度順で次に示す配列のレイアウトと同様です。

```
a b c 0 0
d e f 0 0
0 0 0 0 0
```

## 配列のインデックス化

[index_util.h](https://www.tensorflow.org/code/tensorflow/compiler/xla/index_util.h) にある `IndexUtil` クラスは、与えられたシェイプやレイアウトに対して、複数次元のインデックスと連続なインデックスとの間で、変換するためのユーティリティを提供します。
複数次元のインデックスは、各次元に対して `int64` 型のインデックスを含んでいます。
連続なインデックスは、単一の `int64` 型の値で、配列を保持するバッファをインデックス化します。
シェイプとレイアウトを簡単に生成・扱うためのユーティリティのある、`shape_util.h` と `layout_util.h` を見てください。
