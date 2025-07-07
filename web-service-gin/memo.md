# 音楽アルバム管理用 Web API（Gin使用）

## コード全体の流れ

1. **全アルバムのリストを取得する** `GET /albums`
2. **IDを指定して特定のアルバムを取得する** `GET /albums/:id`
3. **新しいアルバムを追加する** `POST /albums`

プログラムが起動すると、`main`関数が実行され、Ginルーターを初期化し、それぞれのURLにハンドラ関数を紐付けた後、`localhost:8080` でWebサーバーが起動します。

---

## 構造体とグローバル変数

```go
type album struct {
    ID     string  `json:"id"`
    Title  string  `json:"title"`
    Artist string  `json:"artist"`
    Price  float64 `json:"price"`
}

var albums = []album{
    {ID: "1", Title: "Blue Train", Artist: "John Coltrane", Price: 56.99},
    {ID: "2", Title: "Jeru", Artist: "Gerry Mulligan", Price: 17.99},
    {ID: "3", Title: "Sarah Vaughan and Clifford Brown", Artist: "Sarah Vaughan", Price: 39.99},
}
```

### 説明

* **`type album struct`**
  アルバム1件の情報（ID、タイトル、アーティスト、価格）を定義した構造体。
  `json:"..."` は、JSON形式との対応付けを示す構造体タグ。

* **`var albums`**
  album構造体のスライスで、アプリ内で扱う仮想データベース。

---

## 各関数の役割と解説

### `main`関数

```go
func main() {
    router := gin.Default()
    router.GET("/albums", getAlbums)
    router.GET("/albums/:id", getAlbumByID)
    router.POST("/albums", postAlbums)
    router.Run("localhost:8080")
}
```

* Ginのルーターを初期化
* 各URLとハンドラ関数を関連付け
* Webサーバーを起動（8080ポートでリクエスト待機）

---

### `getAlbums`関数

```go
func getAlbums(c *gin.Context) {
    c.IndentedJSON(http.StatusOK, albums)
}
```

* 全アルバムの一覧をJSONで返す（ステータス200）
* `IndentedJSON`により整形されたJSONレスポンス

---

### `getAlbumByID`関数

```go
func getAlbumByID(c *gin.Context) {
    id := c.Param("id")

    for _, a := range albums {
        if a.ID == id {
            c.IndentedJSON(http.StatusOK, a)
            return
        }
    }

    c.IndentedJSON(http.StatusNotFound, gin.H{"message": "album not found"})
}
```

* URLパラメータからIDを取得
* 一致するアルバムがあれば返す（200 OK）
* なければエラーメッセージを返す（404 Not Found）

---

### `postAlbums`関数（改善後）

```go
func postAlbums(c *gin.Context) {
    var newAlbum album

    // JSONを構造体にマッピング（バインド）
    if err := c.BindJSON(&newAlbum); err != nil {
        // エラーがあれば400エラーを返す
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // 新しいアルバムをスライスに追加
    albums = append(albums, newAlbum)
    c.IndentedJSON(http.StatusCreated, newAlbum)
}
```

* JSONデータを受け取って構造体に変換
* 正常であれば `albums` に追加して、201 Createdを返す
* 不正なJSONなら `400 Bad Request` と具体的なエラー内容を返す


## 実行例（POSTリクエスト）

```bash
curl http://localhost:8080/albums \
  --header "Content-Type: application/json" \
  --request "POST" \
  --data '{"id": "4", "title": "The Modern Sound of Betty Carter", "artist": "Betty Carter", "price": 49.99}'
```
