// LETで中間変数を定義しながら、最後に1つの結果を返す
=LET(

  // A1:A100 から空セルを除いた入力セル一覧を src に入れる
  src,FILTER(A1:A100,A1:A100<>""),

  // 区分として認識する文字列の固定一覧を kinds に入れる
  kinds,{"新規";"削除";"変更"},

  // rec には最終的に「区分 / 会社 / 詳細」の中間表を作る
  rec,

    // REDUCE の先頭ダミー行を最後に消すため DROP(…,1) する
    DROP(

      // src の各セルを順番に処理して、1つの縦表に積み上げる
      REDUCE(

        // accumulator の初期値として空文字を入れておく
        "",

        // 処理対象は空セルを除いた各入力セル
        src,

        // acc は今まで積み上げた表、cell は今見ている1セル
        LAMBDA(acc,cell,

          // 1セルの中を分解するためのローカル変数を作る
          LET(

            // cell 内の複数行テキストを改行で縦分割する
            x,TEXTSPLIT(cell,,CHAR(10),TRUE),

            // 先頭行を会社名として取り出す
            co,INDEX(x,1),

            // 2行目以降を body として取り出す
            body,DROP(x,1),

            // body の各行に対して「その時点の現在区分」を付与する
            st,SCAN("",body,LAMBDA(a,v,IF(ISNUMBER(MATCH(v,kinds,0)),v,a))),

            // rows には「区分行そのもの」を除いた詳細行だけを残す
            rows,

              // 条件に合う行だけ残す
              FILTER(

                // 3列の表「現在区分 / 会社名 / 元の行」を横結合で作る
                HSTACK(st,IF(SEQUENCE(ROWS(body)),co),body),

                // body が kinds に一致しない行、つまり詳細行だけを残す
                NOT(ISNUMBER(MATCH(body,kinds,0)))
              ),

            // 今までの表 acc の下に、このセルから作った rows を縦に追加する
            VSTACK(acc,rows)
          )
        )
      ),

      // REDUCE の初期値として入れたダミー先頭行を削除する
      1
    ),

  // 最終出力側も REDUCE 先頭にダミーが入るので最後に DROP する
  DROP(

    // 「新規」「削除」「変更」を順番に処理して最終出力を積み上げる
    REDUCE(

      // 最終出力用 accumulator の初期値
      "",

      // 順番に処理するカテゴリ一覧
      kinds,

      // acc は今までの最終出力、cat は今処理中のカテゴリ
      LAMBDA(acc,cat,

        // 1カテゴリ分の表示ブロックを作るためのローカル変数を定義する
        LET(

          // rec から 1列目が cat の行だけを抽出する
          fc,FILTER(rec,INDEX(rec,,1)=cat,""),

          // このカテゴリが空かどうかを判定する
          emptyCat,AND(ROWS(fc)=1,INDEX(fc,1,1)=""),

          // このカテゴリに属する会社名一覧を重複除去して作る
          comps,IF(emptyCat,"",UNIQUE(INDEX(fc,,2))),

          // このカテゴリに属する詳細行数を数える
          detailCnt,IF(emptyCat,0,ROWS(fc)),

          // 詳細が1件もなければ何も追加せず、そのまま acc を返す
          IF(

            // 詳細0件かどうかの判定
            detailCnt=0,

            // 0件ならこのカテゴリのブロックは出力しない
            acc,

            // 0件でなければカテゴリブロックを組み立てる
            LET(

              // block には「会社名 → その会社の詳細行群」を作る
              block,

                // 会社一覧を REDUCE で縦に積み上げ、先頭ダミーを DROP する
                DROP(

                  // comps の各会社について会社名と詳細群を積み上げる
                  REDUCE(

                    // block 用 accumulator の初期値
                    "",

                    // 処理対象はこのカテゴリ内の会社一覧
                    comps,

                    // a は今までの block、co は今処理中の会社名
                    LAMBDA(a,co,

                      // 会社名1行と、その会社の詳細一覧を下に追加する
                      VSTACK(a,co,FILTER(INDEX(fc,,3),INDEX(fc,,2)=co))
                    )
                  ),

                  // block 用 REDUCE の先頭ダミーを削除する
                  1
                ),

              // 今までの出力 acc の下に、このカテゴリの見出し・本文・件数を追加する
              VSTACK(acc,"■"&cat,block,"","計"&detailCnt&"件","")
            )
          )
        )
      )
    ),

    // 最終出力用 REDUCE の先頭ダミーを削除する
    1
  )
)
