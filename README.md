## Go言語でGoodle Cloud Platform上のVMからGoogleドライブ上のファイルを読み込む

### 前提

Google Drive APIs RESTの中の[Go Quickstart](https://developers.google.com/drive/v3/web/quickstart/go)にあるサンプル`quickstart.go`がVM上で正常に動作すること。そのためには:

- OAuth2.0クライアントIDを作成済みで、`client_secret.json`というファイルが`quickstart.go`と同じフォルダー上に存在している([Go Quickstart](https://developers.google.com/drive/v3/web/quickstart/go)のStep 1)。
- `quickstart.go`が起動するため、あらかじめ以下のツールのインストールが済んでいる([Go Quickstart](https://developers.google.com/drive/v3/web/quickstart/go)のStep 2)。
	* Drive API Go client library
	* OAuth2 package

### `quickstart.go`について補足

`quickstart.go`を実行すると、Googleドライブ上にあるファイル10個のファイル名とIDが出力される。
OAuth2の難しいことはさておき、Googleドライブのファイルにアクセスするために知っておきたい最低限のやりかただけ記す。

### Googleドライブからファイル名を探す(`q1.go`)

例えば`gdsample`という文字列をファイル名に含むファイルを探したい場合、`quickstart.go`の`srv.Files.List().`～の部分を書き換えるだけ。本質的な修正箇所は以下の通り。

`quickstart.go`:
```go
        r, err := srv.Files.List().PageSize(10).
                Fields("nextPageToken, files(id, name)").Do()
```


`q1.go`:
```go
        r, err := srv.Files.List().
                Q("name contains 'gdsample' and trashed = false").
                Fields("nextPageToken, files(id, name)").Do()
```

`trashed = false`は「ゴミ箱に入ったファイルは除く」ことを意味する。GoogleドライブのファイルはIDで一意に決まるせいか **「フォルダー」のどこにあるかは関係ない** らしく、この検索で`gdsample`の文字列を含むファイルはたとえゴミ箱にあっても引っかかってしまうためにわざわざこんなことをする。ゴミ箱の中を常にカラにしているならば、もちろんこんな指定をする必要はない。

### Googleドライブのファイルの内容を読み込む(`q2.go`)

ファイルのIDを指定してそのファイルの中身を読む場合、今度は`srv.Files.List().`～の部分を書き換えるだけでは **中身を読めない**。本質的な変更箇所は次の2か所。

#### OAuth2のスコープを変える

`q1.go`ではファイルのメタ情報だけを読む指定(`drive.DriveMetadataReadonlyScope`)をしていた。ファイルの内容を読むためにはここを`drive.DriveReadonlyScope`に変える必要がある。さもないとファイルの読み込み権限がないのでhttpエラー403が返ってくる。

`q1.go`:
```go
        config, err := google.ConfigFromJSON(b, drive.DriveMetadataReadonlyScope)
```


`q2.go`:
```go
        config, err := google.ConfigFromJSON(b, drive.DriveReadonlyScope)
```

参考までに`drive.DriveReadonlyScope`とは文字列`"https://www.googleapis.com/auth/drive.readonly"`である。

#### ダウンロードを指定する

`srv.Files.List().`～の部分を書き換える。

`q1.go`:
```go
        r, err := srv.Files.List().
                Q("name contains 'gdsample' and trashed = false").
                Fields("nextPageToken, files(id, name)").Do()
```


`q2.go`:
```go
        TXTID := "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"  // File ID
        res, err := srv.Files.Get(TXTID).Download()
```

TXTIDには自分のIDに変えてほしい。

あとは`q2.go`を参照されたい。実は個人的にShift-JISからUTF-8への変換が必要だったためそのプログラムも記載してある(`go get golang.org/x/text/encoding/japanese`が必要)。

