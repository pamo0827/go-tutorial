# Goジェネリクス入門：重複コードを解消し、柔軟な関数を作成する

## はじめに

Go 1.18 で導入されたジェネリクス（Generics）は、異なる型に対して同じロジックを使いたいときに、コードの重複を避け、型安全に汎用的な関数を記述できる強力な機能です。

---

## ジェネリクス導入前：コードの重複

以下の2つの関数は、mapの値の合計を計算しますが、処理内容は全く同じです。違うのは値の型だけです。

```go
func SumInts(m map[string]int64) int64 {
    var s int64
    for _, v := range m {
        s += v
    }
    return s
}

func SumFloats(m map[string]float64) float64 {
    var s float64
    for _, v := range m {
        s += v
    }
    return s
}
```

このようなコードは、型が増えるごとに同様の処理を繰り返し書く必要があり、保守性に問題があります。

---

## ジェネリクスによる解決：`SumIntsOrFloats`

```go
func SumIntsOrFloats[K comparable, V int64 | float64](m map[K]V) V {
    var s V
    for _, v := range m {
        s += v
    }
    return s
}
```

### 型パラメータ `[K comparable, V int64 | float64]` の意味：

- `K comparable`: マップのキーは比較可能な型（マップのキーとして有効な型）である必要があります。
- `V int64 | float64`: 値は `int64` または `float64` のどちらかの型でなければなりません。

---

## 使用例

```go
ints := map[string]int64{"first": 34, "second": 12}
floats := map[string]float64{"first": 35.98, "second": 26.99}

fmt.Printf("Generic Sums: %v and %v
",
    SumIntsOrFloats[string, int64](ints),
    SumIntsOrFloats[string, float64](floats))

fmt.Printf("Generic Sums, type parameters inferred: %v and %v
",
    SumIntsOrFloats(ints),
    SumIntsOrFloats(floats))
```

---

## 型制約の再利用：`Number` インターフェース

```go
type Number interface {
    int64 | float64
}

func SumNumbers[K comparable, V Number](m map[K]V) V {
    var s V
    for _, v := range m {
        s += v
    }
    return s
}
```

### `type Number interface { int64 | float64 }` の意味：

- `int64 | float64` 型に制限された型集合（ユニオン型）を表現しています。
- 複数の関数で共通の型制約を使いたい場合に便利です。

---

## 使用例（型推論あり）

```go
fmt.Printf("Generic Sums with Constraint: %v and %v
",
    SumNumbers(ints),
    SumNumbers(floats))
```

---

## まとめ：ジェネリクスのメリット

- **コードの再利用性**：重複コードを避けて汎用的な関数を定義できる。
- **型安全性**：コンパイル時に型チェックされるため、バグを防げる。
- **可読性と保守性**：意図が明確になり、修正や拡張が容易になる。
- **表現力**：意味のある型制約（例：`Number`）を使って関数の用途を明確にできる。

---