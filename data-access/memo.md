# RDBへのアクセス チュートリアルまとめ

このチュートリアルでは、Goの標準ライブラリ `database/sql` パッケージと、MySQL用の**データベースドライバ**を用いて、データベース操作の基本フローを学習した。

## 1. データベース操作の基本ステップ

Goにおけるデータベース操作は、以下の3ステップに集約される。

| ステップ | 関数 | 説明 |
| --- | --- | --- |
| 接続 | `sql.Open`, `db.Ping` | 通信路（コネクションプール）を確立する。 |
| クエリ | `db.Query`, `db.QueryRow` | データの取得（SELECT文）を行う。 |
| 実行 | `db.Exec` | データの変更（INSERT, UPDATE, DELETE）を行う。 |

## 2. 各ステップの詳細

### Step 1: 接続

- `database/sql` は共通のインタフェースを提供するだけで、実際の通信はMySQLドライバ（例：`go-sql-driver/mysql`）が担う。
- `sql.Open` はDSN（Data Source Name）を渡して `sql.DB`（接続プール）を返すが、この時点では実際の接続は行われていない。
- `db.Ping` により、実際に通信が可能かを検証できる。

### Step 2: クエリ（データ取得）

- `db.Query` は複数行を取得する。取得結果を `sql.Rows` として受け取り、ループ処理しながら `rows.Scan(...)` で値を読み出す。
- `defer rows.Close()` は必須であり、これを怠ると接続リークの原因となる。
- `db.QueryRow` は1件だけ取得する場合に使用する。存在しない場合は `sql.ErrNoRows` を返すため、エラー分岐が可能である。

### Step 3: 実行（データ変更）

- `db.Exec` により、INSERT や UPDATE、DELETE など副作用を持つクエリを発行できる。
- `sql.Result` から `LastInsertId()` や `RowsAffected()` を取得することで、影響のあった行を把握できる。

## 3. main.goの解説

```go
db, err = sql.Open("mysql", cfg.FormatDSN())
if err != nil {
    log.Fatal(err)
}
```

- `"mysql"` は、`github.com/go-sql-driver/mysql` により `database/sql` に登録されたドライバ名である。
- `cfg.FormatDSN()` は、接続情報（ユーザー名、パスワード、ホスト、ポート、DB名）を文字列として組み立てるメソッドである。
- `sql.Open` は接続プールを返すが、実際の接続テストは `db.Ping()` で行う。
- Goの標準的なエラーハンドリングとして、`err != nil` によるチェックは欠かせない。

---

**ハードコード (Hard-coding)** とは、プログラムの中に、将来変更される可能性のあるデータを**直接書き込んでしまうこと。**

### 全体の流れ

1. MySQLデータベースに接続する。
2. 特定のアーティストのアルバムを全部探す。
3. 特定のIDのアルバムを1個だけ探す。
4. 新しいアルバムを1個追加する。

---

### `main`関数

```go
unfold_morego
content_copyadd
 Show full code block
func main() {
// ① .envファイルから設定を読み込む
	godotenv.Load()
    cfg := mysql.NewConfig()
    cfg.User = os.Getenv("DBUSER")
    cfg.Passwd = os.Getenv("DBPASS")
    cfg.Net = "tcp"
    cfg.Addr = "127.0.0.1:3306"
    cfg.DBName = "recordings"

// ② データベースとの窓口（ハンドル）を用意する
var err error
    db, err = sql.Open("mysql", cfg.FormatDSN())
    if err != nil {
        log.Fatal(err)
    }

// ③ 実際に通信できるか確認
    pingErr := db.Ping()
    if pingErr != nil {
        log.Fatal(pingErr)
    }
    fmt.Println("Connected!")

// ④ 作った関数を順番に呼び出して、DBを操作
	albums, err := albumsByArtist("John Coltrane")
	if err != nil {
    	log.Fatal(err)
	}
	fmt.Printf("Albums found: %v\n", albums)

// IDが2のアルバムをハードコードで取ってくる
	alb, err := albumByID(2)
	if err != nil {
    	log.Fatal(err)
	}
	fmt.Printf("Album found: %v\n", alb)

// 新しいアルバムをDBに挿入
	albID, err := addAlbum(Album{
        Title:  "The Modern Sound of Betty Carter",
        Artist: "Betty Carter",
        Price:  49.99,
	})
	if err != nil {
    	log.Fatal(err)
	}
	fmt.Printf("ID of added album: %v\n", albID)
}

```

- **① 設定の読み込み**: `godotenv.Load()`で、パスワードなどの情報を書いた`.env`ファイルを読み込む。`os.Getenv()`でその値を取り出して、`mysql.Config`っていう設定用の構造体に詰めていく。
- **② 接続準備**: `sql.Open`でデータベースとの「窓口」を作る。実際に接続してるわけじゃなくあくまで準備段階。
- **③ 疎通確認**: `db.Ping()`でデータベースに挨拶する。
- **④ DB操作**: 自分で作った関数を順番に呼び出すだけ。アーティスト名で検索したり、IDで検索したり、新しいデータを追加したりな。エラーが出たら`log.Fatal`でプログラムを止める。

---

### `albumsByArtist`関数

アーティスト名を渡すと、そのアルバムを全部取ってくる関数

```go
unfold_morego
content_copyadd
 Show full code block
func albumsByArtist(name string) ([]Album, error) {
// Album型のスライスを用意する
var albums []Album

// "SELECT * ..." でクエリを実行！ "?" を使うのがSQLインジェクション対策で重要
    rows, err := db.Query("SELECT * FROM album WHERE artist = ?", name)
    if err != nil {
        return nil, fmt.Errorf("albumsByArtist %q: %v", name, err)
    }
// deferは「この関数が終わる時に実行してくれ」っていう予約
defer rows.Close()

// for rows.Next()で結果を1行ずつループ処理する
for rows.Next() {
        var alb Album
// rows.Scanで行のデータをAlbum構造体に挿入
if err := rows.Scan(&alb.ID, &alb.Title, &alb.Artist, &alb.Price); err != nil {
            return nil, fmt.Errorf("albumsByArtist %q: %v", name, err)
        }
// スライスにどんどん追加していく
        albums = append(albums, alb)
    }
// ループが終わった後にもう一回エラーチェック。
if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("albumsByArtist %q: %v", name, err)
    }
// エラーがなければ、見つけたアルバムのリストを返す
return albums, nil
}

```

---

### `albumByID`関数：ID指定

IDを渡すと、そのアルバムを1件だけ取ってくる。

```go
unfold_lessgo
content_copyadd
func albumByID(id int64) (Album, error) {
    var alb Album

// QueryRowは1行しか返さないクエリに使う。
    row := db.QueryRow("SELECT * FROM album WHERE id = ?", id)
// Scanでデータを構造体にマッピング
if err := row.Scan(&alb.ID, &alb.Title, &alb.Artist, &alb.Price); err != nil {
// もしデータがなかったらエラーが返ってくる
if err == sql.ErrNoRows {
return alb, fmt.Errorf("albumsById %d: no such album", id)
        }
return alb, fmt.Errorf("albumsById %d: %v", id, err)
    }
    return alb, nil
}

```

---

### `addAlbum`関数：新しいデータを追加

`Album`構造体で新しいアルバムのデータを受け取って、DBに`INSERT`する。
```go
unfold_lessgo
content_copyadd
func addAlbum(alb Album) (int64, error) {
// Execはデータを返さないSQL（INSERTとかUPDATE）に使う
    result, err := db.Exec("INSERT INTO album (title, artist, price) VALUES (?, ?, ?)", alb.Title, alb.Artist, alb.Price)
    if err != nil {
        return 0, fmt.Errorf("addAlbum: %v", err)
    }
// LastInsertId()で今追加した行のIDが取れる
    id, err := result.LastInsertId()
    if err != nil {
        return 0, fmt.Errorf("addAlbum: %v", err)
    }
    return id, nil
}

```