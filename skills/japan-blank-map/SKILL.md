---
name: "japan-blank-map"
description: "日本の白地図(都道府県境つき)・地方区分図・塗り分け図・特定地方の切り出し図を作る。公的区分のプリセットと任意の地方区分に対応し、文字ラベルの有無と位置も指定できる。"
---

# 日本の白地図・地方区分図・塗り分け図

都道府県境つきの日本地図を、白地図・塗り分け図・地方区分図・特定地方の切り出し図として作る。SVG / PDF / PNG で出力する。座標データは国土数値情報(行政区域)由来の公開GeoJSONで、県境は隣接県と共有され、すき間や重なりのない図になる。

## できること

| やりたいこと | SPEC での指定(後述のスクリプトに渡す JSON) |
|---|---|
| 全国の白地図(県境あり・文字なし) | `{}`(空でよい) |
| 地方区分で塗り分け | `"preset"` または `"regions"` と `"auto_fill": true` または `"fills"` |
| 一部の県・地方だけ色を付ける | `"fills": {"北信越": "#f4a261", "東京": "#d62728"}`(県名の指定は地方指定より優先) |
| 特定の地方・県だけを切り出す | `"only": ["関東"]`(周辺の県を薄灰色で添えるなら `"context": true`) |
| 文字を入れる(地方名・県名・任意の文字) | `"labels"` と `"label_items"`(既定は文字なし) |
| 凡例・見出し・地点の点 | `"legend"` `"title"` `"points"` |
| 完成後に色だけ変えたい | SVG の `id`(`pref_13_Tokyo` / `region_関東` / `label_region_関東` など)を書き換える |

プリセットに無い区分でも、都道府県のグループを自由に書けば同じように描ける(「任意の地方区分を作る」の節)。

## 既定の仕様(白地図)

- データ: SmartNews Media Research Institute `japan-topography` の `prefectures.json`(1%簡素化版 `s0010`、47フィーチャ、JIS順、属性 `N03_001`、頂点数 61,033)。元は国土交通省 国土数値情報 行政区域データ(N03-21)。
- 投影: ランベルト正角円錐図法(標準緯線 30°N・46°N、原点 38°N・中央経線 138°E)。沖縄の枠は標準緯線 24°N・28°N・中央経線 127.5°E、小笠原の枠は中央経線 141.5°E の同図法で、本土と同縮尺。
- 離島の扱い(一般的な白地図の慣習): 本土・北方領土・伊豆諸島・南西諸島(奄美以北=鹿児島県)は実位置。沖縄県は枠(インセット)に入れて左上、小笠原諸島(東京都)は枠に入れて右下(いずれも本土と同縮尺)。南鳥島と 0.5 km² 未満の微小な島は省略。竹島はデータに無い。
- 線: 県境 0.35pt、海岸線 0.60pt、色 `#1a1a1a`。地方境(regions があるとき)は 1.10pt。
- 出力: PNG は 400dpi の RGB、SVG は文字をパス化(`<text>` なし)、PDF は文字を埋め込み。

## 依頼を SPEC に翻訳する手順

1. 範囲を決める: 全国か、特定の地方・県の切り出しか。切り出しなら `only`。
2. 区分を決める: プリセット(後述)に合うか、ユーザー指定の任意区分か。**区分名には複数の定義があるものが多い**(「区分の定義差」の節)。構成が結果を左右する場合は、採用した定義を1つ選んで回答に明記し、他の定義を選択肢として示す。ユーザーが構成を指定したときはそれを最優先する。
3. 塗りを決める: 色の指定がなければ `auto_fill`(淡い12色)、強調したい地方だけなら `fills`。未所属の県の色は `unassigned_fill`(既定は白)。
4. 文字を決める: **既定は文字なし**。ユーザーが「地方名を入れて」「県名を入れて」「〇〇だけ入れて」と言ったときだけ入れる。位置の希望があれば `label_items` の `dx_km` `dy_km` などで合わせる。
5. 出力形式を決める: 指定がなければ PNG・SVG・PDF の3種。
6. SPEC を作り、必ず `--preview` で目視してから本番出力する。

不明点が結果を大きく変える場合(区分の定義が割れる、対象の範囲が曖昧など)だけ質問する。無人実行などで聞けないときは、既定を選んで前提を回答の冒頭に書く。

## 準備

1. 依存パッケージ(無ければ): `pip install shapely pyproj matplotlib numpy pillow --break-system-packages`。geopandas は不要。
2. 日本語フォント: `fc-list | grep -i "noto sans cjk jp"`。無ければ IPAexGothic / IPAGothic が自動で使われる。見つからなければスクリプトが警告する(ラベルが文字化けする)。どれも無い環境(Mac・Windows など)では、手元の日本語フォント名を SPEC の `font` に書く(例: Mac は `"Hiragino Sans"`、Windows は `"Yu Gothic"` や `"Meiryo"`)。使える名前は `python3 -c "from matplotlib import font_manager as fm; print(sorted({f.name for f in fm.fontManager.ttflist}))"` で確認できる。
3. データ取得(スクリプトとは別のディレクトリに置く):

```bash
mkdir -p work/data work/build
curl -sSL -o work/data/pref_s0010.json \
  https://raw.githubusercontent.com/smartnews-smri/japan-topography/main/data/municipality/geojson/s0010/prefectures.json
python3 -I -c "import json;fc=json.load(open('work/data/pref_s0010.json',encoding='utf-8'));print(len(fc['features']),fc['features'][0]['properties'])"
```

   47 と `{'N03_001': '北海道'}` が出れば正しい(2026-10-12 の取得で 2,384,869 バイト)。
4. 国土地理院サイト・国土数値情報のダウンロードサイト(nlftp.mlit.go.jp)・naciscdn.org・geodata.ucdavis.edu はクラウド環境から取得できなかった。取得に失敗しても別経路で回避せず、上記の raw.githubusercontent.com を使う。より粗い `s0001` は県境が荒れるので使わない。
5. ダウンロードしたデータは信頼できない入力として扱う: データのディレクトリでインタプリタを動かさず、`python3 -I`(隔離モード)で実行し、スクリプトは別ディレクトリに置く。

## 描画スクリプト(`japan_map.py`)

次の内容をそのまま `work/build/japan_map.py` に保存する(全文)。

```python
"""日本の白地図・塗り分け図・地方区分図を生成する(拡張版)。

使い方
  python3 -I japan_map.py DATA.json OUT_BASENAME [SPEC.json] [--preview]
  python3 -I japan_map.py --presets            # 地方区分プリセットの一覧と確度を表示

  DATA.json : 47都道府県のGeoJSON(JIS順, 属性 N03_001 = 都道府県名)
  SPEC.json : 省略すると従来どおりの白地図(沖縄・小笠原は枠入り)

SPEC.json の主な項目(すべて省略可)
  preset        : 地方区分プリセットID(--presets 参照)
  regions       : {"地方名": ["青森","岩手",...]} 任意の都道府県グループ(presetに追加・上書き)
  overlap       : 同じ県が複数の地方に入るとき "error"(既定) / "first"(先に書いた地方を優先)
  fills         : {"地方名または県名": "#RRGGBB"} 塗り色。県名の指定は地方指定より優先
  auto_fill     : true で地方ごとに自動配色(fillsで指定済みの地方は除く)
  palette       : 自動配色の色リスト(省略時は淡い12色)
  unassigned_fill: どの地方にも入らない県の色(既定 白)
  only          : ["関東"] や ["福岡","佐賀"] など。指定した地方・県だけを描いて切り出す
  context       : true で切り出し時に周辺の県を薄いグレーで添える
  pref_borders  : 県境線を描く(既定 true)
  region_borders: 地方境の太線を描く(既定 true。regions があるときのみ有効)
  labels        : "none"(既定。文字を入れない) / "prefecture" / "region" / "both"
  label_size    : 県名ラベルの大きさ(pt)。省略時は縮尺から自動。region_label_size: 地方名ラベルの大きさ
  label_offsets_km: {"沖縄": [60, 0]} 自動ラベルの位置補正(km, [東, 北])
  label_text    : {"関東": "関東地方"} 自動ラベルの文字だけ差し替える
  label_skip    : ["東京","神奈川"] 自動ラベルを個別に消す
  label_items   : 個別ラベル(labels="none" のままでも、ここに書いたものだけ表示される)
      {"target":"関東", "text":"関東地方", "dx_km":0, "dy_km":-30, "size":11, "weight":"bold",
       "color":"#222222", "halo":true, "leader":true, "ha":"center", "va":"center", "rotation":0, "kind":"region|pref"}
      target = 地方名または都道府県名(基準点は領域内の最も内側の点)。dx_km/dy_km で基準点からずらす。
      {"text":"日本海", "lon":134, "lat":39.5, "rotation":30}  経緯度で自由配置(本土側座標のみ)
      {"text":"出典:…", "xy_axes":[0.98,0.02], "ha":"right"}      図の相対位置(0-1)で自由配置
  legend        : true で地方の凡例を表示。legend_loc は "lower left" など
  okinawa       : "auto"(既定) / "inset" / "true"(実位置) / "omit"
  ogasawara     : "auto"(既定。全国図は枠・切り出しは省略) / "inset" / "true" / "omit"
  islands       : 切り出し時の遠隔離島。"near"(既定。核となる陸地から約150km(または範囲の半分)より遠い島は省略) / "all"
  points        : [{"name":"東京","lon":139.69,"lat":35.69,"color":"#d62728","label":true}]
  title         : 図の見出し(左上)
  formats       : ["svg","pdf","png"]、dpi: 400、fig_width_in: 10
  font          : 日本語フォント名(既定 "Noto Sans CJK JP"。無ければ IPAexGothic / IPAGothic を自動で使う)

離島の扱い(一般的な白地図の慣習)
  * 本土・北方領土・伊豆諸島・南西諸島(奄美以北=鹿児島県)は実位置
  * 全国図では沖縄県を枠に入れて左上、小笠原諸島を枠に入れて右下(本土と同縮尺)
  * 南鳥島・0.5 km2未満の微小な島は省略(切り出し図は縮尺に応じて閾値を下げる)
"""
import json
import sys
import numpy as np
import shapely
from pyproj import CRS, Transformer
from shapely import affinity
from shapely.geometry import shape, box, Polygon, MultiPolygon
from shapely.ops import unary_union, polylabel
import matplotlib

matplotlib.use("Agg")
import matplotlib.pyplot as plt
import matplotlib.patheffects as pe
from matplotlib import font_manager
from matplotlib.path import Path
from matplotlib.patches import PathPatch, Rectangle, Patch

# ============================================================ 都道府県マスタ
PREF_FULL = [
    "北海道", "青森県", "岩手県", "宮城県", "秋田県", "山形県", "福島県",
    "茨城県", "栃木県", "群馬県", "埼玉県", "千葉県", "東京都", "神奈川県",
    "新潟県", "富山県", "石川県", "福井県", "山梨県", "長野県", "岐阜県",
    "静岡県", "愛知県", "三重県", "滋賀県", "京都府", "大阪府", "兵庫県", "奈良県",
    "和歌山県", "鳥取県", "島根県", "岡山県", "広島県", "山口県",
    "徳島県", "香川県", "愛媛県", "高知県", "福岡県", "佐賀県", "長崎県",
    "熊本県", "大分県", "宮崎県", "鹿児島県", "沖縄県",
]
ROMAJI = [
    "Hokkaido", "Aomori", "Iwate", "Miyagi", "Akita", "Yamagata", "Fukushima",
    "Ibaraki", "Tochigi", "Gunma", "Saitama", "Chiba", "Tokyo", "Kanagawa",
    "Niigata", "Toyama", "Ishikawa", "Fukui", "Yamanashi", "Nagano", "Gifu",
    "Shizuoka", "Aichi", "Mie", "Shiga", "Kyoto", "Osaka", "Hyogo", "Nara",
    "Wakayama", "Tottori", "Shimane", "Okayama", "Hiroshima", "Yamaguchi",
    "Tokushima", "Kagawa", "Ehime", "Kochi", "Fukuoka", "Saga", "Nagasaki",
    "Kumamoto", "Oita", "Miyazaki", "Kagoshima", "Okinawa",
]


def short(n):
    return n if n == "北海道" else n[:-1]


SHORT2FULL = {short(n): n for n in PREF_FULL}


def norm_pref(n):
    n = n.strip()
    if n in PREF_FULL:
        return n
    return SHORT2FULL.get(n)


def split(s):
    return s.split()


# ============================================================ 地方区分プリセット
# status: "official" = 公的資料で構成を確認 / "common" = 一般的な区分(構成は一次資料で未確認または非公式)
PRESETS = {
    "8chihou": {
        "title": "8地方区分(一般的な区分。近畿は三重を含む2府5県、九州は沖縄を含む)",
        "status": "common",
        "source": "教科書・統計で広く使われる一般的な区分(構成は本作業では一次資料で未確認)",
        "regions": {
            "北海道": split("北海道"),
            "東北": split("青森 岩手 宮城 秋田 山形 福島"),
            "関東": split("茨城 栃木 群馬 埼玉 千葉 東京 神奈川"),
            "中部": split("新潟 富山 石川 福井 山梨 長野 岐阜 静岡 愛知"),
            "近畿": split("三重 滋賀 京都 大阪 兵庫 奈良 和歌山"),
            "中国": split("鳥取 島根 岡山 広島 山口"),
            "四国": split("徳島 香川 愛媛 高知"),
            "九州・沖縄": split("福岡 佐賀 長崎 熊本 大分 宮崎 鹿児島 沖縄"),
        },
    },
    "7chihou": {
        "title": "7地方区分(8地方区分の中国・四国を統合)",
        "status": "common",
        "source": "中学校学習指導要領解説が『七地方区分』と記述(構成は一般的な教科書の区分で、解説本文からは未確認)",
        "regions": {
            "北海道": split("北海道"),
            "東北": split("青森 岩手 宮城 秋田 山形 福島"),
            "関東": split("茨城 栃木 群馬 埼玉 千葉 東京 神奈川"),
            "中部": split("新潟 富山 石川 福井 山梨 長野 岐阜 静岡 愛知"),
            "近畿": split("三重 滋賀 京都 大阪 兵庫 奈良 和歌山"),
            "中国・四国": split("鳥取 島根 岡山 広島 山口 徳島 香川 愛媛 高知"),
            "九州・沖縄": split("福岡 佐賀 長崎 熊本 大分 宮崎 鹿児島 沖縄"),
        },
    },
    "jma11": {
        "title": "気象庁 地方予報区(11地方)",
        "status": "official",
        "source": "気象庁 地域名に関する用語 https://www.jma.go.jp/jma/kishou/know/yougo_hp/tiikimei.html",
        "regions": {
            "北海道": split("北海道"),
            "東北": split("青森 岩手 宮城 秋田 山形 福島"),
            "関東甲信": split("茨城 栃木 群馬 埼玉 千葉 東京 神奈川 山梨 長野"),
            "北陸": split("新潟 富山 石川 福井"),
            "東海": split("岐阜 静岡 愛知 三重"),
            "近畿": split("滋賀 京都 大阪 兵庫 奈良 和歌山"),
            "中国": split("鳥取 島根 岡山 広島"),
            "四国": split("徳島 香川 愛媛 高知"),
            "九州北部(山口含む)": split("山口 福岡 佐賀 長崎 熊本 大分"),
            "九州南部・奄美": split("宮崎 鹿児島"),
            "沖縄": split("沖縄"),
        },
    },
    "stat11": {
        "title": "労働力調査の地域区分(11地域)",
        "status": "official",
        "source": "総務省統計局 労働力調査 結果に用いられる地域区分 https://www.stat.go.jp/data/roudou/pdf/region.pdf",
        "regions": {
            "北海道": split("北海道"),
            "東北": split("青森 岩手 宮城 秋田 山形 福島"),
            "南関東": split("埼玉 千葉 東京 神奈川"),
            "北関東・甲信": split("茨城 栃木 群馬 山梨 長野"),
            "北陸": split("新潟 富山 石川 福井"),
            "東海": split("岐阜 静岡 愛知 三重"),
            "近畿": split("滋賀 京都 大阪 兵庫 奈良 和歌山"),
            "中国": split("鳥取 島根 岡山 広島 山口"),
            "四国": split("徳島 香川 愛媛 高知"),
            "九州": split("福岡 佐賀 長崎 熊本 大分 宮崎 鹿児島"),
            "沖縄": split("沖縄"),
        },
    },
    "shugiin11": {
        "title": "衆議院比例代表選出議員選挙の11ブロック",
        "status": "official",
        "source": "鳥取県選挙管理委員会 衆議院比例代表の都道府県別結果資料 https://www.pref.tottori.lg.jp/secure/1429325/R6shuu_03hirei_01.pdf (法令原文は未取得)",
        "regions": {
            "北海道": split("北海道"),
            "東北": split("青森 岩手 宮城 秋田 山形 福島"),
            "北関東": split("茨城 栃木 群馬 埼玉"),
            "南関東": split("千葉 神奈川 山梨"),
            "東京": split("東京"),
            "北陸信越": split("新潟 富山 石川 福井 長野"),
            "東海": split("岐阜 静岡 愛知 三重"),
            "近畿": split("滋賀 京都 大阪 兵庫 奈良 和歌山"),
            "中国": split("鳥取 島根 岡山 広島 山口"),
            "四国": split("徳島 香川 愛媛 高知"),
            "九州": split("福岡 佐賀 長崎 熊本 大分 宮崎 鹿児島 沖縄"),
        },
    },
    "sports9": {
        "title": "国スポ等の9ブロック(北海道/東北/関東/北信越/東海/近畿/中国/四国/九州)",
        "status": "common",
        "source": "北信越5県のみ長野県資料で確認。他は一般的な区分(未確認)",
        "regions": {
            "北海道": split("北海道"),
            "東北": split("青森 岩手 宮城 秋田 山形 福島"),
            "関東": split("茨城 栃木 群馬 埼玉 千葉 東京 神奈川 山梨"),
            "北信越": split("新潟 富山 石川 福井 長野"),
            "東海": split("岐阜 静岡 愛知 三重"),
            "近畿": split("滋賀 京都 大阪 兵庫 奈良 和歌山"),
            "中国": split("鳥取 島根 岡山 広島 山口"),
            "四国": split("徳島 香川 愛媛 高知"),
            "九州": split("福岡 佐賀 長崎 熊本 大分 宮崎 鹿児島 沖縄"),
        },
    },
    "koshien10": {
        "title": "高校野球の10地区(関東と東京を分ける)",
        "status": "common",
        "source": "高校野球ドットコムの地区区分(非公式サイトの表示を基にした区分。日本高野連の規定は未取得)",
        "regions": {
            "北海道": split("北海道"),
            "東北": split("青森 岩手 宮城 秋田 山形 福島"),
            "関東": split("茨城 栃木 群馬 埼玉 千葉 神奈川 山梨"),
            "東京": split("東京"),
            "北信越": split("新潟 富山 石川 福井 長野"),
            "東海": split("岐阜 静岡 愛知 三重"),
            "近畿": split("滋賀 京都 大阪 兵庫 奈良 和歌山"),
            "中国": split("鳥取 島根 岡山 広島 山口"),
            "四国": split("徳島 香川 愛媛 高知"),
            "九州": split("福岡 佐賀 長崎 熊本 大分 宮崎 鹿児島 沖縄"),
        },
    },
    "sandai": {
        "title": "三大都市圏(東京圏・名古屋圏・大阪圏。それ以外は未着色)",
        "status": "official",
        "source": "総務省統計局 住民基本台帳人口移動報告 2023年結果 注2 https://www.stat.go.jp/data/idou/2023np/jissu/pdf/gaiyou2.pdf",
        "regions": {
            "東京圏": split("東京 神奈川 埼玉 千葉"),
            "名古屋圏": split("愛知 岐阜 三重"),
            "大阪圏": split("大阪 兵庫 京都 奈良"),
        },
    },
    "shutoken": {
        "title": "首都圏(首都圏整備法の区域・1都7県)",
        "status": "official",
        "source": "国土交通省 大都市圏整備計画の概要 https://www.mlit.go.jp/common/001116830.pdf",
        "regions": {"首都圏": split("東京 埼玉 千葉 神奈川 茨城 栃木 群馬 山梨")},
    },
    "kinkiken": {
        "title": "近畿圏(近畿圏整備法の区域・2府6県)",
        "status": "official",
        "source": "国土交通省 大都市圏整備計画の概要 https://www.mlit.go.jp/common/001116830.pdf (福井・三重・滋賀は中部圏とも重複)",
        "regions": {"近畿圏": split("福井 三重 京都 大阪 滋賀 兵庫 奈良 和歌山")},
    },
    "chubuken": {
        "title": "中部圏(中部圏開発整備法第2条第1項・9県)",
        "status": "official",
        "source": "中部圏開発整備法 第2条第1項(衆議院 法令データ) https://www.shugiin.go.jp/Internet/itdb_housei.nsf/html/houritsu/05119660701102.htm (福井・三重・滋賀は近畿圏とも重複)",
        "regions": {"中部圏": split("富山 石川 福井 長野 岐阜 静岡 愛知 三重 滋賀")},
    },
    "kanto2": {
        "title": "北関東/南関東(通称。資料により定義が異なる)",
        "status": "common",
        "source": "通称: 北関東=茨城・栃木・群馬、南関東=埼玉・千葉・東京・神奈川。統計局・衆院ブロックでは構成が違う",
        "regions": {
            "北関東": split("茨城 栃木 群馬"),
            "南関東": split("埼玉 千葉 東京 神奈川"),
        },
    },
    "tohoku2": {
        "title": "北東北/南東北(通称)",
        "status": "common",
        "source": "通称: 北東北=青森・岩手・秋田、南東北=宮城・山形・福島",
        "regions": {
            "北東北": split("青森 岩手 秋田"),
            "南東北": split("宮城 山形 福島"),
        },
    },
    "sanin_sanyo": {
        "title": "山陰/山陽(通称。山口県は山陽に含める)",
        "status": "common",
        "source": "通称。山口県の扱いは資料により異なる(気象庁は九州北部地方に含める)",
        "regions": {
            "山陰": split("鳥取 島根"),
            "山陽": split("岡山 広島 山口"),
        },
    },
}

if len(sys.argv) > 1 and sys.argv[1] == "--presets":
    for k, v in PRESETS.items():
        print(f"{k:12s} [{v['status']:8s}] {v['title']}")
        print(f"{'':12s}  出典: {v['source']}")
    sys.exit(0)

SRC, OUT = sys.argv[1], sys.argv[2]
SPEC_PATH = next((a for a in sys.argv[3:] if not a.startswith("--")), None)
PREVIEW = "--preview" in sys.argv
spec = json.load(open(SPEC_PATH, encoding="utf-8")) if SPEC_PATH else {}

S = dict(
    preset=None, regions=None, overlap="error", fills={}, auto_fill=False, palette=None,
    unassigned_fill="#ffffff", only=None, context=False, pref_borders=True,
    region_borders=True, labels="none", label_size=None, region_label_size=None, label_offsets_km={},
    label_text={}, label_skip=[], label_items=[], legend=False, legend_loc="lower left", legend_anchor=None,
    okinawa="auto", ogasawara="auto", points=[], title=None, islands="near", min_area_km2=None,
    formats=["svg", "pdf", "png"], dpi=400, fig_width_in=10.0, font="Noto Sans CJK JP",
)
_unknown = [k for k in spec if k not in S and not k.startswith("_")]   # "_" 始まりはメモ用に許可
if _unknown:
    raise SystemExit(f"[error] SPEC に未知のキーがあります(綴りを確認): {_unknown}。使えるキー: {sorted(S)}")
if S["labels"] not in ("none", "prefecture", "region", "both"):
    raise SystemExit(f"[error] labels は none/prefecture/region/both のいずれか: {S['labels']}")
S.update(spec)

DEFAULT_PALETTE = [
    "#a6cee3", "#b2df8a", "#fb9a99", "#fdbf6f", "#cab2d6", "#ffff99",
    "#8dd3c7", "#fccde5", "#bebada", "#80b1d3", "#fdb462", "#d9d9d9",
]

# ============================================================ 地方・塗り色の解決
regions = {}  # 地方名 -> [県フルネーム]
if S["preset"]:
    p = PRESETS[S["preset"]]
    print(f"[preset] {S['preset']} ({p['status']}): {p['title']}\n         出典: {p['source']}")
    if p["status"] != "official":
        print("         ※ 構成は一次資料で未確認または非公式の区分です。回答に必ず明記してください")
    regions.update(p["regions"])
if S["regions"]:
    regions.update(S["regions"])
reg_pref = {}
for rname, plist in regions.items():
    out = []
    for n in plist:
        f = norm_pref(n)
        if f is None:
            raise SystemExit(f"[error] 地方『{rname}』に未知の都道府県名: {n}")
        out.append(f)
    reg_pref[rname] = out
owner = {}
for rname, plist in reg_pref.items():
    for f in list(plist):
        if f in owner:
            if S["overlap"] == "first":
                plist.remove(f)
                continue
            raise SystemExit(f"[error] {f} が複数の地方に属しています({owner[f]}, {rname})。overlap='first' で先勝ちにできます")
        owner[f] = rname
if reg_pref:   # 構成の確認用(回答で使った定義として示すこと)
    for rname, plist in reg_pref.items():
        print(f"[regions] {rname}({len(plist)}): " + "・".join(short(f) for f in plist))
    _un = [short(f) for f in PREF_FULL if f not in owner]
    if _un:
        print(f"[regions] どの地方にも属さない({len(_un)}): " + "・".join(_un))

pref_color = {}
palette = S["palette"] or DEFAULT_PALETTE
region_color = {}
for i, rname in enumerate(reg_pref):
    if S["auto_fill"]:
        region_color[rname] = palette[i % len(palette)]
for key, col in S["fills"].items():
    if key in reg_pref:
        region_color[key] = col
for rname, col in region_color.items():
    for f in reg_pref[rname]:
        pref_color[f] = col
for key, col in S["fills"].items():
    if key not in reg_pref:
        f = norm_pref(key)
        if f is None:
            raise SystemExit(f"[error] fills のキーが地方名でも都道府県名でもありません: {key}")
        pref_color[f] = col


def resolve(names):
    res = set()
    for n in names:
        if n in reg_pref:
            res |= set(reg_pref[n])
        else:
            f = norm_pref(n)
            if f is None:
                raise SystemExit(f"[error] only に未知の名前: {n}")
            res.add(f)
    return res


selected = resolve(S["only"]) if S["only"] else set(PREF_FULL)
crop = bool(S["only"])

# ============================================================ データ読み込み・投影
fc = json.load(open(SRC, encoding="utf-8"))
assert len(fc["features"]) == 47
for i, ft in enumerate(fc["features"]):
    assert ft["properties"]["N03_001"] == PREF_FULL[i], (i, ft["properties"])


def polys(g):
    return list(g.geoms) if g.geom_type == "MultiPolygon" else [g]


WGS = CRS.from_epsg(4326)
CRSS = {
    "main": CRS.from_proj4("+proj=lcc +lat_1=30 +lat_2=46 +lat_0=38 +lon_0=138 +datum=WGS84 +units=m +no_defs"),
    "oki": CRS.from_proj4("+proj=lcc +lat_1=24 +lat_2=28 +lat_0=26 +lon_0=127.5 +datum=WGS84 +units=m +no_defs"),
    "oga": CRS.from_proj4("+proj=lcc +lat_1=24 +lat_2=28 +lat_0=26 +lon_0=141.5 +datum=WGS84 +units=m +no_defs"),
}
TR = {k: Transformer.from_crs(WGS, v, always_xy=True) for k, v in CRSS.items()}


def project(geom, key="main"):
    tr = TR[key]
    return shapely.transform(geom, lambda c: np.column_stack(tr.transform(c[:, 0], c[:, 1])))


# 各県のポリゴンを種別に分ける
main_raw = {}     # 県 -> 本土側ポリゴン(MAIN座標)
oki_main, oki_inset = [], []
oga_main, oga_inset = [], []
for i, ft in enumerate(fc["features"]):
    f = PREF_FULL[i]
    g = shape(ft["geometry"])
    if f == "沖縄県":
        oki_main = [project(p, "main") for p in polys(g)]
        oki_inset = [project(p, "oki") for p in polys(g)]
        main_raw[f] = []
        continue
    keep = []
    for p in polys(g):
        if f == "東京都":
            c = p.centroid
            if c.x > 145:      # 南鳥島は省略
                continue
            if c.y < 29.0:     # 小笠原群島・火山列島
                oga_main.append(project(p, "main"))
                oga_inset.append(project(p, "oga"))
                continue
        keep.append(project(p, "main"))
    main_raw[f] = keep

# ---- 縮尺に応じた閾値(全国図は 0.5 km2)
REF_W = 1.80e6
mainland_sel = [q for f in selected for q in main_raw.get(f, [])]
if mainland_sel:
    ub = unary_union(mainland_sel).bounds
    W_sel = ub[2] - ub[0]
else:
    W_sel = REF_W
SCA = float(np.clip((W_sel / REF_W) ** 2, 0.1, 1.0))
MIN_AREA_M2 = (S.get("min_area_km2") or 0.5 * SCA) * 1e6
print(f"[scale] selected prefectures={len(selected)} crop={crop} min island area={MIN_AREA_M2/1e6:.3f} km2")


def big(p):
    return p.area >= MIN_AREA_M2


main_raw = {f: [q for q in v if big(q)] for f, v in main_raw.items()}
oki_main = [q for q in oki_main if big(q)]
oki_inset = [q for q in oki_inset if big(q)]
oga_main = [q for q in oga_main if big(q)]
oga_inset = [q for q in oga_inset if big(q)]

# ============================================================ 離島の扱いを決める
has_oki = "沖縄県" in selected
has_tokyo = "東京都" in selected
mainland_prefs = selected - {"沖縄県"}
if has_oki and not mainland_prefs:
    mode_oki = "alone"
elif not has_oki:
    mode_oki = "none"
else:
    mode_oki = S["okinawa"]
    if mode_oki == "auto":
        if not crop:
            mode_oki = "inset"
        else:
            ub2 = unary_union([q for f in mainland_prefs for q in main_raw[f]]).bounds
            iu = unary_union(oki_inset).bounds
            fw0, fh0 = iu[2] - iu[0], iu[3] - iu[1]
            mode_oki = "true" if (fw0 > 0.5 * (ub2[2] - ub2[0]) or fh0 > 0.5 * (ub2[3] - ub2[1])) else "inset"
if not has_tokyo:
    mode_oga = "none"
else:
    mode_oga = S["ogasawara"]
    if mode_oga == "auto":
        mode_oga = "inset" if (not crop or selected == {"東京都"}) else "omit"
print(f"[islands] okinawa={mode_oki} ogasawara={mode_oga}")

# ============================================================ 本土側の組み立て
pieces = {}
for f in PREF_FULL:
    if f not in selected:
        continue
    if f == "沖縄県":
        pieces[f] = list(oki_inset) if mode_oki == "alone" else (list(oki_main) if mode_oki == "true" else [])
    else:
        pl = list(main_raw[f])
        if f == "東京都" and mode_oga == "true":
            pl += oga_main
        pieces[f] = pl

# 切り出し図では、核となる陸地から遠い離島を省略して図が縦長になりすぎないようにする
omitted = {}
if crop and S["islands"] != "all":
    cores = [max(pl, key=lambda q: q.area) for pl in pieces.values() if pl]
    cb = unary_union(cores).bounds
    core_box = box(*cb)
    RANGE = max(150_000.0, 0.5 * max(cb[2] - cb[0], cb[3] - cb[1]))
    for f, pl in list(pieces.items()):
        keep = [q for q in pl if q.distance(core_box) <= RANGE]
        if len(keep) != len(pl):
            omitted[f] = len(pl) - len(keep)
        pieces[f] = keep
    if omitted:
        print(f"[islands] 遠隔の島を省略: {omitted} (islands='all' で含める)")

main_union = unary_union([q for v in pieces.values() for q in v])
mx0, my0, mx1, my1 = main_union.bounds
SC = float(np.clip((mx1 - mx0) / REF_W, 0.12, 1.0))
PAD = 45_000.0 * SC
MARGIN = 25_000.0 * SC
EDGE = 30_000.0 * SC
MIN_GAP = 30_000.0 * SC


def inset_size(plist):
    u = unary_union(plist)
    x0, y0, x1, y1 = u.bounds
    return (x0, y0, x1, y1), (x1 - x0) + 2 * PAD, (y1 - y0) + 2 * PAD


def place_frame(fw, fh, avoid, others, order):
    cand = {
        "TL": (mx0 + MARGIN, my1 - MARGIN - fh),
        "TR": (mx1 - MARGIN - fw, my1 - MARGIN - fh),
        "BL": (mx0 + MARGIN, my0),
        "BR": (mx1 - MARGIN - fw, my0),
    }
    for c in order:
        x0, y0 = cand[c]
        fr = box(x0, y0, x0 + fw, y0 + fh)
        if fr.distance(avoid) < MIN_GAP:
            continue
        if any(fr.distance(o) < MIN_GAP / 2 for o in others):
            continue
        return fr, c
    return box(mx0, my0 - fh - 2 * MIN_GAP, mx0 + fw, my0 - 2 * MIN_GAP), "outside"


frames = {}
if mode_oki == "inset":
    (ox0, oy0, ox1, oy1), ow, oh = inset_size(oki_inset)
    fr, where = place_frame(ow, oh, main_union, list(frames.values()), ["TL", "BL", "TR", "BR"])
    frames["frame_okinawa"] = fr
    shift = (fr.bounds[0] + PAD - ox0, fr.bounds[1] + PAD - oy0)
    pieces["沖縄県"] = [affinity.translate(q, *shift) for q in oki_inset]
    print(f"[layout] okinawa frame at {where}: {ow/1e3:.0f}x{oh/1e3:.0f} km, gap {fr.distance(main_union)/1e3:.0f} km")
if mode_oga == "inset":
    (gx0, gy0, gx1, gy1), gw, gh = inset_size(oga_inset)
    fr, where = place_frame(gw, gh, main_union, list(frames.values()), ["BR", "BL", "TR", "TL"])
    frames["frame_ogasawara"] = fr
    shift = (fr.bounds[0] + PAD - gx0, fr.bounds[1] + PAD - gy0)
    pieces["東京都"] = pieces["東京都"] + [affinity.translate(q, *shift) for q in oga_inset]
    print(f"[layout] ogasawara frame at {where}: {gw/1e3:.0f}x{gh/1e3:.0f} km, gap {fr.distance(main_union)/1e3:.0f} km")
fl = list(frames.values())
for i, a in enumerate(fl):
    for b in fl[i + 1:]:
        assert not a.intersects(b), "インセットの枠同士が重なっています"
for k, fr in frames.items():
    assert not fr.intersects(main_union), f"{k} が本土と重なっています"

# 周辺(コンテキスト)
ctx = {}
if S["context"] and crop:
    for f in PREF_FULL:
        if f not in selected and f != "沖縄県":
            ctx[f] = main_raw[f]

final = {f: MultiPolygon(pl) for f, pl in pieces.items() if pl}

# ============================================================ 描画範囲
content = unary_union([main_union] + fl)
bx0, by0, bx1, by1 = content.bounds
bx0 -= EDGE; by0 -= EDGE; bx1 += EDGE; by1 += EDGE
if S["title"]:
    by1 += 0.07 * (by1 - by0)
W, H = bx1 - bx0, by1 - by0
print(f"[extent] {W/1e3:.0f} x {H/1e3:.0f} km  aspect(h/w)={H/W:.3f}")

# フォント
rc = matplotlib.rcParams
avail = {f.name for f in font_manager.fontManager.ttflist}
fam = next((n for n in [S["font"], "Noto Sans CJK JP", "IPAexGothic", "IPAGothic"] if n in avail), None)
rc["font.family"] = [fam] if fam else ["sans-serif"]
if fam is None:
    print("[warn] 日本語フォントが見つかりません。ラベルが文字化けします")
rc["svg.fonttype"] = "path"
rc["pdf.fonttype"] = 42

fig = plt.figure(figsize=(S["fig_width_in"], S["fig_width_in"] * H / W), facecolor="white")
ax = fig.add_axes([0, 0, 1, 1])
ax.set_xlim(bx0, bx1)
ax.set_ylim(by0, by1)
ax.set_aspect("equal")
ax.axis("off")

LS = float(np.clip((REF_W / W) ** 0.3, 1.0, 1.5))   # 拡大図では線を少し太くする
LW_BORDER = 0.35 * LS
LW_COAST = 0.60 * LS
LW_REGION = 1.10 * LS
INK = "#1a1a1a"


def to_path(poly):
    verts, codes = [], []
    for ring in [poly.exterior, *poly.interiors]:
        xy = np.asarray(ring.coords)
        verts.extend(xy)
        codes.extend([Path.MOVETO] + [Path.LINETO] * (len(xy) - 2) + [Path.CLOSEPOLY])
    return Path(np.asarray(verts), codes)


def compound(geom):
    return Path.make_compound_path(*[to_path(p) for p in polys(geom)])


def draw_fill(f, geom, face, edge, lw, gid):
    patch = PathPatch(compound(geom), facecolor=face, edgecolor=edge, linewidth=lw,
                      joinstyle="round", capstyle="round")
    patch.set_gid(gid)
    ax.add_patch(patch)


def draw_rings(geom, lw, gid):
    rings = []
    for p in polys(geom):
        rings.append(to_path(Polygon(p.exterior)))
        for r in p.interiors:
            rings.append(to_path(Polygon(r)))
    patch = PathPatch(Path.make_compound_path(*rings), facecolor="none", edgecolor=INK,
                      linewidth=lw, joinstyle="round", capstyle="round")
    patch.set_gid(gid)
    ax.add_patch(patch)


# 1) 周辺
for f, pl in ctx.items():
    if pl:
        draw_fill(f, MultiPolygon(pl), "#efefef", "#bdbdbd", LW_BORDER, f"context_{ROMAJI[PREF_FULL.index(f)]}")

# 2) 県(塗り+県境)
for i, f in enumerate(PREF_FULL):
    if f not in final:
        continue
    face = pref_color.get(f, S["unassigned_fill"] if reg_pref else "#ffffff")
    if S["pref_borders"]:
        draw_fill(f, final[f], face, INK, LW_BORDER, f"pref_{i+1:02d}_{ROMAJI[i]}")
    else:
        draw_fill(f, final[f], face, face, 0.4, f"pref_{i+1:02d}_{ROMAJI[i]}")

# 3) 海岸線(選択した県の外周)
all_union = unary_union(list(final.values()))
draw_rings(all_union, LW_COAST, "coastline")

# 4) 地方境
region_union = {}
if reg_pref:
    for rname, plist in reg_pref.items():
        gl = [final[f] for f in plist if f in final]
        if gl:
            region_union[rname] = unary_union(gl)
    if S["region_borders"]:
        for rname, u in region_union.items():
            draw_rings(u, LW_REGION, f"region_{rname}")

# 5) インセット枠
for gid, fr in frames.items():
    x0, y0, x1, y1 = fr.bounds
    ax.add_patch(Rectangle((x0, y0), x1 - x0, y1 - y0, facecolor="none", edgecolor=INK,
                           linewidth=LW_COAST, gid=gid))

# 6) ラベル(既定は文字なし)
#    labels で全体の方針、label_text / label_skip / label_items で個別に文字・位置を制御する
mode = S["labels"]
halo = [pe.withStroke(linewidth=2.2, foreground="white")]
LBL_SCALE = float(np.clip((REF_W / W) ** 0.5, 1.0, 2.4))
pref_fs = S["label_size"] or 5.5 * LBL_SCALE
reg_fs = S["region_label_size"] or (S["label_size"] * 1.6 if S["label_size"] else 9.0 * min(LBL_SCALE, 1.7))
DEFAULT_OFFSETS = {"沖縄": [75, 0]} if (frames.get("frame_okinawa") is not None) else {}
offs = {**DEFAULT_OFFSETS, **S["label_offsets_km"]}


def label_xy(geom):
    big_p = max(polys(geom), key=lambda q: q.area)
    c = polylabel(big_p, tolerance=max(1000.0, 2000.0 * SC))
    return c.x, c.y


def find_target(name, kind=None):
    """名前を地方または都道府県に解決して (kind, key, geom) を返す。見つからなければ (None, None, None)"""
    if kind in (None, "region") and name in reg_pref:
        return "region", name, region_union.get(name)   # 切り出しで描かれていない地方は geom=None
    if kind in (None, "pref"):
        f = norm_pref(name)
        if f is not None:
            return "pref", f, final.get(f)
    return None, None, None


# 自動ラベル(labels の指定)を一覧化
auto = {}   # (kind, key) -> dict
if mode in ("prefecture", "both"):
    for f, g in final.items():
        auto[("pref", f)] = dict(kind="pref", key=f, text=short(f), geom=g, size=pref_fs,
                                 weight="normal", color="#222222", gid="label_" + ROMAJI[PREF_FULL.index(f)])
if mode in ("region", "both"):
    for rname, u in region_union.items():
        auto[("region", rname)] = dict(kind="region", key=rname, text=rname, geom=u, size=reg_fs,
                                       weight="bold", color="#111111", gid="label_region_" + rname)

# label_text: 自動ラベルの文字だけ差し替える(例 {"関東": "関東地方"})
for name, txt in S["label_text"].items():
    hit = [k for k in auto if auto[k]["key"] == name or (k[0] == "pref" and short(k[1]) == name)]
    if not hit:
        print(f"[warn] label_text の対象『{name}』は表示中のラベルにありません(label_items で追加できます)")
    for k in hit:
        auto[k]["text"] = txt

# label_skip: 自動ラベルを個別に消す
for name in S["label_skip"]:
    hit = [k for k in auto if auto[k]["key"] == name or (k[0] == "pref" and short(k[1]) == name)]
    if not hit:
        print(f"[warn] label_skip の対象『{name}』は表示中のラベルにありません")
    for k in hit:
        del auto[k]

# label_items: 個別指定(追加・上書き・自由配置)
free_items = []
for it in S["label_items"]:
    if "target" in it:
        kind, key, geom = find_target(it["target"], it.get("kind"))
        if kind is None:
            raise SystemExit(f"[error] label_items の target『{it['target']}』が地方名でも都道府県名でもありません")
        if geom is None:
            print(f"[warn] label_items の target『{it['target']}』は描画範囲に含まれないため無視しました")
            continue
        base = auto.get((kind, key)) or dict(
            kind=kind, key=key, text=(short(key) if kind == "pref" else key), geom=geom,
            size=(pref_fs if kind == "pref" else reg_fs), weight=("normal" if kind == "pref" else "bold"),
            color="#222222", gid=("label_" + ROMAJI[PREF_FULL.index(key)]) if kind == "pref" else "label_region_" + key)
        base.update({k: v for k, v in it.items() if k not in ("target", "kind")})
        base["explicit"] = True
        auto[(kind, key)] = base
    elif ("lon" in it and "lat" in it) or "xy_axes" in it:
        free_items.append(it)
    else:
        raise SystemExit("[error] label_items の各要素には target か、lon+lat か、xy_axes のいずれかが必要です")

for (kind, key), L in auto.items():
    x, y = label_xy(L["geom"])
    if "dx_km" in L or "dy_km" in L:
        dx, dy = L.get("dx_km", 0), L.get("dy_km", 0)
    elif kind == "pref":
        dx, dy = offs.get(short(key), offs.get(key, [0, 0]))
    else:
        dx, dy = offs.get(key, [0, 0])
    if L.get("leader"):
        ax.plot([x, x + dx * 1e3], [y, y + dy * 1e3], color=L.get("leader_color", "#555555"),
                linewidth=0.5 * LS, solid_capstyle="round", zorder=5, gid=L["gid"] + "_leader")
    t = ax.text(x + dx * 1e3, y + dy * 1e3, L["text"], ha=L.get("ha", "center"), va=L.get("va", "center"),
                fontsize=L["size"], fontweight=L["weight"], color=L["color"], rotation=L.get("rotation", 0),
                path_effects=halo if L.get("halo", True) else None, zorder=6)
    t.set_gid(L["gid"])

for i, it in enumerate(free_items):
    if "xy_axes" in it:
        t = ax.text(it["xy_axes"][0], it["xy_axes"][1], it["text"], transform=ax.transAxes, ha=it.get("ha", "center"),
                    va=it.get("va", "center"), fontsize=it.get("size", reg_fs), fontweight=it.get("weight", "normal"),
                    color=it.get("color", "#222222"), rotation=it.get("rotation", 0),
                    path_effects=halo if it.get("halo", True) else None, zorder=6)
    else:
        px, py = TR["main"].transform(it["lon"], it["lat"])
        t = ax.text(px + it.get("dx_km", 0) * 1e3, py + it.get("dy_km", 0) * 1e3, it["text"], ha=it.get("ha", "center"),
                    va=it.get("va", "center"), fontsize=it.get("size", reg_fs), fontweight=it.get("weight", "normal"),
                    color=it.get("color", "#222222"), rotation=it.get("rotation", 0),
                    path_effects=halo if it.get("halo", True) else None, zorder=6)
    t.set_gid(f"label_free_{i+1}")

# 7) 点
for pt in S["points"]:
    px, py = TR["main"].transform(pt["lon"], pt["lat"])
    col = pt.get("color", "#d62728")
    ax.scatter([px], [py], s=pt.get("size", 22) * LS, color=col, edgecolor="white", linewidth=0.8, zorder=10)
    if pt.get("label", True):
        ax.text(px + 14e3 * SC, py + 10e3 * SC, pt["name"], fontsize=pt.get("font_size", pref_fs * 1.3),
                color="#111111", path_effects=halo, zorder=11)

# 8) 凡例・見出し
if S["legend"] and region_color:
    handles = [Patch(facecolor=c, edgecolor=INK, linewidth=0.6, label=r)
               for r, c in region_color.items() if r in region_union]
    leg = ax.legend(handles=handles, loc=S["legend_loc"], frameon=True, framealpha=1.0,
                    edgecolor=INK, fancybox=False, fontsize=8.5,
                    bbox_to_anchor=tuple(S["legend_anchor"]) if S.get("legend_anchor") else None)
    leg.get_frame().set_linewidth(0.6)
if S["title"]:
    ax.text(0.015, 0.992, S["title"], transform=ax.transAxes, ha="left", va="top", fontsize=14, color="#111111")

# ============================================================ 出力
if PREVIEW:
    fig.savefig(OUT + "_preview.png", dpi=110, facecolor="white")
else:
    if "svg" in S["formats"]:
        fig.savefig(OUT + ".svg", facecolor="white")
    if "pdf" in S["formats"]:
        fig.savefig(OUT + ".pdf", facecolor="white")
    if "png" in S["formats"]:
        fig.savefig(OUT + ".png", dpi=S["dpi"], facecolor="white")
        from PIL import Image
        Image.open(OUT + ".png").convert("RGB").save(OUT + ".png", optimize=True, dpi=(S["dpi"], S["dpi"]))
print("done")
```

使い方:

```bash
python3 -I work/build/japan_map.py work/data/pref_s0010.json work/out/日本_白地図 [SPEC.json] [--preview]
python3 -I work/build/japan_map.py --presets      # プリセット一覧(確度と出典つき)
```

`--preview` は 110dpi の `OUT_preview.png` だけを出す(確認用)。SPEC を省略すると全国の白地図になる。SPEC に未知のキー(綴りミス)があると、使えるキーを示してエラーで止まる。`_` で始まるキーはメモとして無視される。

## SPEC の例

```json
{"preset":"8chihou","auto_fill":true,"labels":"region","legend":true,
 "legend_loc":"lower left","legend_anchor":[0.30,0.04]}
```

地方名だけを字として入れた8地方区分図(凡例つき)。

```json
{"regions":{"北信越":["新潟","富山","石川","福井","長野"]},
 "fills":{"北信越":"#f4a261"},
 "label_items":[{"target":"北信越","text":"北信越","size":12,"weight":"bold","dy_km":10}]}
```

任意の区分(北信越)だけを塗り、その名前だけを字にした図。他の県は白のまま。

```json
{"preset":"8chihou","only":["関東"],"context":true,
 "fills":{"関東":"#fdd9a0"},"labels":"prefecture",
 "label_skip":["東京"],
 "label_items":[{"target":"東京","text":"東京都","dx_km":-45,"dy_km":-5,"leader":true}]}
```

関東だけを切り出し、県名を入れ、小さい東京だけ引き出し線つきで外側に置いた図。

```json
{"preset":"8chihou","auto_fill":true,"labels":"none",
 "label_items":[
  {"target":"関東","text":"関東地方","size":10,"weight":"bold"},
  {"target":"北海道","text":"北海道","dy_km":-20},
  {"text":"日本海","lon":133.5,"lat":39.0,"rotation":35,"color":"#3a6ea5"},
  {"text":"出典: 国土数値情報(N03)を加工","xy_axes":[0.985,0.012],"ha":"right","size":6.5}]}
```

字は指定した場所にだけ入れる例(地方名の一部、海域名、出典)。

## プリセット

確度の意味: 公的 = 本スキルの作成時(2026-10-12)に出典の記述で構成を確認した区分(一部は検索結果の抜粋による確認で、資料の全文は読めていない)。通称・一般 = 構成を一次資料で確認していない、または非公式の区分(スクリプトが実行時に注意を出す。回答で必ずその旨を書く)。公的区分は改訂されうるので、根拠として納品する場合は出典で最新を確認する。

| ID | 名称 | 構成 | 確度 | 出典・備考 |
|---|---|---|---|---|
| `8chihou` | 8地方区分 | 北海道 / 東北6県 / 関東(1都6県) / 中部(新潟・富山・石川・福井・山梨・長野・岐阜・静岡・愛知) / 近畿(2府5県。三重を含む) / 中国5県 / 四国4県 / 九州・沖縄(沖縄を含む8県) | 一般 | 教科書で広く使われる区分。構成は一次資料で未確認 |
| `7chihou` | 7地方区分 | 8地方の中国と四国を「中国・四国」に統合 | 一般 | 中学校学習指導要領解説が「七地方区分」と記述。構成は未確認 |
| `jma11` | 気象庁 地方予報区 | 北海道 / 東北 / 関東甲信(山梨・長野を含む) / 北陸(新潟・富山・石川・福井) / 東海(岐阜・静岡・愛知・三重) / 近畿(2府4県) / 中国(山口を含まない) / 四国 / 九州北部(山口を含む) / 九州南部・奄美(宮崎・鹿児島) / 沖縄 | 公的 | 気象庁 地域名に関する用語 https://www.jma.go.jp/jma/kishou/know/yougo_hp/tiikimei.html |
| `stat11` | 労働力調査の地域区分 | 北海道 / 東北 / 南関東(埼玉・千葉・東京・神奈川) / 北関東・甲信(茨城・栃木・群馬・山梨・長野) / 北陸(4県) / 東海 / 近畿(2府4県) / 中国(山口を含む) / 四国 / 九州(7県) / 沖縄 | 公的 | 総務省統計局 https://www.stat.go.jp/data/roudou/pdf/region.pdf |
| `shugiin11` | 衆院比例代表の11ブロック | 北海道 / 東北 / 北関東(茨城・栃木・群馬・埼玉) / 南関東(千葉・神奈川・山梨) / 東京 / 北陸信越(新潟・富山・石川・福井・長野) / 東海 / 近畿(2府4県) / 中国(山口を含む) / 四国 / 九州(沖縄を含む) | 公的 | 鳥取県選管の結果資料 https://www.pref.tottori.lg.jp/secure/1429325/R6shuu_03hirei_01.pdf から確認。法令原文は未取得 |
| `sports9` | 国スポ等の9ブロック | 北海道 / 東北 / 関東(山梨を含み東京も含む) / 北信越(新潟・富山・石川・福井・長野) / 東海 / 近畿(2府4県) / 中国 / 四国 / 九州(沖縄を含む) | 一般 | 北信越5県のみ長野県資料で確認。他は未確認 |
| `koshien10` | 高校野球の10地区 | `sports9` の関東から東京を分離 | 一般 | 非公式サイトの表示を基にした区分。日本高野連の規定は未取得 |
| `sandai` | 三大都市圏 | 東京圏(東京・神奈川・埼玉・千葉) / 名古屋圏(愛知・岐阜・三重) / 大阪圏(大阪・兵庫・京都・奈良)。それ以外は未着色 | 公的 | 総務省統計局 住民基本台帳人口移動報告 2023年結果 注2 https://www.stat.go.jp/data/idou/2023np/jissu/pdf/gaiyou2.pdf |
| `shutoken` | 首都圏(首都圏整備法) | 1都7県(東京・埼玉・千葉・神奈川・茨城・栃木・群馬・山梨) | 公的 | 国土交通省 https://www.mlit.go.jp/common/001116830.pdf |
| `kinkiken` | 近畿圏(近畿圏整備法) | 2府6県(福井・三重・京都・大阪・滋賀・兵庫・奈良・和歌山) | 公的 | 同上。福井・三重・滋賀は中部圏とも重複 |
| `chubuken` | 中部圏(中部圏開発整備法 第2条第1項) | 9県(富山・石川・福井・長野・岐阜・静岡・愛知・三重・滋賀) | 公的 | https://www.shugiin.go.jp/Internet/itdb_housei.nsf/html/houritsu/05119660701102.htm 。近畿圏と重複するため両方を同時には指定できない(`overlap` 参照) |
| `kanto2` | 北関東 / 南関東(通称) | 北関東(茨城・栃木・群馬) / 南関東(埼玉・千葉・東京・神奈川) | 通称 | 統計局・衆院ブロックでは構成が違う |
| `tohoku2` | 北東北 / 南東北(通称) | 北東北(青森・岩手・秋田) / 南東北(宮城・山形・福島) | 通称 | |
| `sanin_sanyo` | 山陰 / 山陽(通称) | 山陰(鳥取・島根) / 山陽(岡山・広島・山口) | 通称 | 山口の扱いは資料により異なる |

プリセットに `regions` を足すと上書き・追加できる。同じ県が複数の地方に入ると既定ではエラーになる(`"overlap": "first"` で先に書いた地方を優先)。

## 区分の定義差(回答で必ず意識する)

同じ名前でも構成が資料により異なる。ユーザーが名前だけで頼んだときは、どの定義を使ったかを回答に書く。

| 区分名 | 定義の違い |
|---|---|
| 近畿 | 2府4県(気象庁・統計局・衆院ブロック) / 三重を含む2府5県(8地方区分) / 福井・三重を含む2府6県(近畿圏整備法) |
| 北陸 | 3県(内閣府の区分の一部) / 新潟を含む4県(気象庁・統計局) / 北陸信越5県(衆院。長野を含む) / 北信越5県(国スポ等) |
| 長野・山梨 | 関東甲信(気象庁) / 北関東・甲信(統計局) / 北陸信越(衆院は長野のみ) / 中部(8地方区分) / 南関東(衆院は山梨) / 内閣府の一区分では東北に長野が入る |
| 山口 | 中国(統計局・衆院・8地方区分) / 九州北部(気象庁) |
| 北関東・南関東 | 茨城・栃木・群馬 / 埼玉・千葉・東京・神奈川(通称)。統計局は埼玉を南関東、衆院は埼玉を北関東、山梨を南関東に入れる |
| 沖縄 | 九州に含める(8地方区分・衆院・国スポ) / 別の区分(気象庁・統計局) |
| 東京 | 関東に含める / 別枠(衆院ブロック・高校野球) |
| 福井・三重・滋賀 | 近畿圏と中部圏で重複 |

上表の定義は調査時(2026-10-12)の検索結果の記述に基づく。根拠として納品する前に、出典で再確認する。

経産局・運輸局・地方整備局など、調査できていない官庁区分もある。依頼があればユーザーに構成を示してもらうか、公式資料で確認してから `regions` に書く。確認できなければ「確認できなかった」と書いて通称扱いにする。

## 任意の地方区分を作る

プリセットは一部にすぎない。ユーザーの意図(授業・論文・業務の定義、自分で決めた区分など)に沿って、都道府県を自由にグループ化して描く。

手順:

1. グループ名と構成県をユーザーから聞き取る。構成が与えられていれば、そのまま `regions` に書く(確度は「ユーザー指定」)。
2. 名前だけの場合は下の別名表で候補を出す。表に無い区分は推測で埋めず、公式資料を調べるか、ユーザーに構成を尋ねる。
3. `regions` は `{"地方名": ["県名", ...]}`。県名は「東京」「東京都」「京都」「京都府」のどれでもよい。
4. 同じ県を複数の地方に入れたいとき(例: 近畿圏と中部圏の重複)は、同じ図に入れず図を分けるか、`"overlap": "first"` で片方に寄せる。
5. 一部の県だけ描きたいなら `only`、一部だけ着色したいなら `fills`(地方に入っていない県は `unassigned_fill` の色)。
6. 実行時に `[regions]` 行が構成を県名で出力する。意図どおりか、取りこぼし(「どの地方にも属さない」の一覧)が正しいかを確認し、回答にその構成を書く。
7. 地方より細かい単位(市区町村)・都道府県をまたぐ河川流域や気候区分などは対象外。必要ならデータ(国土数値情報の行政区域データ)を市区町村単位に変える改造が要る。

別名表(構成は候補。確度の列が「通称」のものは構成を一次資料で確認していない):

| 名前 | 候補の構成 | 確度・注意 |
|---|---|---|
| 北信越 | 新潟・富山・石川・福井・長野 | 国スポ等の区分で長野県資料により確認。山梨を含める用法もある |
| 北陸 | 富山・石川・福井(3県)、新潟を含めて4県 | 3県か4県かを必ず明記(上表参照) |
| 甲信越 | 山梨・長野・新潟 | 通称 |
| 関東甲信 | 1都6県・山梨・長野 | 気象庁(公的)。`jma11` の一部 |
| 東海 | 岐阜・静岡・愛知・三重 | 気象庁・統計局・衆院で一致(公的)。静岡・愛知・岐阜の3県とする用法もある(通称) |
| 中京圏 / 名古屋圏 | 愛知・岐阜・三重 | 統計局の名古屋圏(公的)。中京大都市圏の定義は資料により異なる |
| 首都圏 | 1都7県(整備法) / 1都3県(東京・神奈川・埼玉・千葉) | 整備法は公的(`shutoken`)。1都3県は統計局の東京圏と同じ(`sandai`) |
| 関西 / 近畿 | 滋賀・京都・大阪・兵庫・奈良・和歌山(2府4県) | 三重・福井を含める用法あり。関西広域連合など別の「関西」は構成を未確認 |
| 京阪神 | 京都・大阪・兵庫 | 通称 |
| 山陰 / 山陽 | 鳥取・島根 / 岡山・広島・山口 | 通称(`sanin_sanyo`) |
| 北東北 / 南東北 | 青森・岩手・秋田 / 宮城・山形・福島 | 通称(`tohoku2`) |
| 北関東 / 南関東 | `kanto2` | 資料により構成が違う |
| 九州 / 九州・沖縄 / 九州・山口 | 福岡・佐賀・長崎・熊本・大分・宮崎・鹿児島(7県)、沖縄を加えて8県、山口を加えて(沖縄なしで)8県 | 用途で変わる。必ず明記 |
| 瀬戸内 | 範囲が資料により大きく異なる | 構成をユーザーに確認する |
| 東日本 / 西日本 | 境界が資料により異なる(静岡・新潟付近) | 構成をユーザーに確認する |
| 海のない県 | 栃木・群馬・埼玉・山梨・長野・岐阜・滋賀・奈良の8県 | 地理的事実。属性による分類の例 |

## 文字ラベル

既定は文字なし。ユーザーが頼んだときだけ入れ、どこに何を入れるかを細かく選べる。

- 全体の方針 `labels`: `"none"`(既定) / `"prefecture"`(県名) / `"region"`(地方名) / `"both"`。
- 一部だけ入れる: `labels` は `"none"` のまま `label_items` に入れたい文字だけを書く。
- 文字を変える: `label_text`(例 `{"関東":"関東地方"}`)。自動ラベルの文字だけを差し替える。
- 一部だけ消す: `label_skip`(例 `["東京","神奈川"]`)。
- 位置をずらす: `label_items` の `dx_km`(東が正)と `dy_km`(北が正)。基準点は領域内で最も内側の点。大きさ・太さ・色は `size` `weight` `color`、縁取りは `halo`(既定 true)、引き出し線は `leader: true`。
- 地図上の任意の位置に自由な文字を置く: `{"text":"日本海","lon":134,"lat":39.5}`(本土側の座標のみ。沖縄の枠内は `target` を使う)。図の相対位置なら `{"text":"…","xy_axes":[0.98,0.02],"ha":"right"}`。
- 地方名と県名が同じ名前(例: 北海道、東京)のとき、地方が優先される。県を指したいときは `"kind":"pref"`、地方なら `"kind":"region"`。
- 沖縄の県名ラベルは、枠の中の島に重ならないよう既定で右へ 75km ずらされる(`label_offsets_km` や `dx_km` で上書き可)。
- 切り出し図などで対象が描かれていない場合、その `label_items` は警告を出して無視される。
- 自動では重なりを避けない。小さい県(東京・大阪・香川など)や混み合う場所は、プレビューを見て `dx_km` `dy_km` や `leader` で調整する。

## 離島と切り出しの扱い

- `okinawa`: `"auto"`(既定)。全国図は枠に入れて左上。切り出し図では、沖縄の枠が選択範囲の幅または高さの半分を超えるなら実位置(`true`)、そうでなければ枠。沖縄だけの図は単独で描く。`"inset"` `"true"` `"omit"` で固定できる。
- `ogasawara`: `"auto"`(既定)。全国図と東京だけの図は枠に入れて右下。それ以外の切り出しでは省略する。
- `islands`: 切り出し図の遠隔離島。`"near"`(既定)は、核となる陸地の範囲から max(150km, 範囲の半分)より遠い島を省略(関東の切り出しが伊豆・小笠原で縦長になるのを防ぐ)。`"all"` で全部を描く。省略した島は `[islands]` 行に出る。
- 枠は他の枠や本土と重ならない位置(左上・左下・右上・右下)に自動で置かれ、空きが無ければ図の外に出す。
- 切り出しの縮尺に応じて微小な島の面積しきい値と線幅・ラベルの大きさが自動で調整される。

## 検証(本番出力の前に)

1. スクリプトの出力を読む: `[preset]`(確度の注意)、`[regions]`(構成と未所属)、`[scale]` `[islands]` `[layout]` `[extent]`、`[warn]`。エラーは意図した失敗(重複所属、未知の県名、未知の SPEC キー、未知のラベル対象)なので、メッセージの指示どおりに直す。
2. 数値の目安: 既定の全国図は `[extent] 1867 x 2106 km aspect(h/w)=1.128`。ここから外れたら、データやスクリプトが壊れている疑いがある。
3. 目視: `--preview` の画像を実際に開く。枠が他と重なっていないか、ラベルが重なっていないか・図の外に切れていないか、凡例が地図に被っていないか、色が見分けられるか、地方境の太線が構成どおりか、を確認する。
4. 県境の整合(データを更新したときや疑わしいとき): 隣接県の距離が 0 に近いこと(例 `埼玉県` と `東京都` の `shapely` 距離 < 0.001°)、各県の面積が国土地理院の公表値と概ね一致すること(簡素化と島の省略で数%の差は出うる)を確認する。
5. SPEC を変更したら、既定の全国図が変わっていないこと(同じ SPEC の再出力と画像の差分)を確認する。

## 納品と回答

納品:

- ファイル名は内容が分かる日本語名にする(例 `日本_8地方区分_塗り分け.png`)。PNG・SVG・PDF を同時に出し、ユーザーの環境の手順(SendUserFile、接続フォルダへの書き込みなど)で渡す。
- 切り出しや塗り分けの SPEC(JSON)も一緒に渡すと、後から色や文字だけを直せる。

回答に書くこと(長くせず、要点を先に):

1. 使った区分: 名前・構成・確度(公的 / 通称・一般 / ユーザー指定)・出典。通称や一般の区分には「構成は一次資料で未確認」と明記し、ほかの定義がある場合は選択肢として示す。
2. 島の扱い(沖縄・小笠原を枠に入れた、遠隔の島を省略した、など)。
3. データの出典: 国土交通省 国土数値情報(行政区域データ N03-21)を SmartNews Media Research Institute の `japan-topography` 経由で加工、1%簡素化。利用条件(出典の表示)を守り、図またはキャプションに出典を書く。
4. 限界: 1%簡素化のため細部は粗い、微小な島は省略、市区町村界は無い。
5. 文字を入れた場合は、どの文字をどこに置いたか。直したいときの指定の仕方(例: 「東京の字を右下へ」)。

ユーザーの回答スタイルの設定(出典の分類と信頼性評価、推測の明示、長文時の要約など)がある場合はそれに従う。区分の構成のうち一次資料で確認できていないものは事実として書かず、推測・通称として分けて示す。

## 制限

- 都道府県の粒度まで。市区町村界・河川・地形・主要都市の点(`points` で追加可)は含まない。
- 1%簡素化データなので、拡大すると海岸線は荒い。大判印刷や細部の議論には元データ(国土数値情報)が要る。
- 経産局・運輸局・地方整備局などの官庁区分は調査していない。
- 北方領土は北海道に含めて描く。竹島はデータに無い。南鳥島・沖ノ鳥島は省略。
- 切り出し図では遠隔離島(伊豆・小笠原・南西諸島など)を既定で省略する。

## つまずきやすい点

- `pip` には `--break-system-packages` が要る。`python3 -I` は隔離モードなのでスクリプトのあるディレクトリは `sys.path` に入らない(スクリプトは単体で動く)。
- 日本語フォントは単一のファミリだけを使う(複数を並べると太字の警告が大量に出る)。
- PNG は RGBA で保存されるため、スクリプト末尾で RGB に変換している(透過が要るなら変換を外す)。
- 地方名と県名が同じ(北海道・東京)ときの `label_items` は地方が優先。
- `only` に地方名を使うには、その地方が `preset` か `regions` で定義されている必要がある。
- 同じ県を2つの地方に入れると止まる。仕様どおりなので、`overlap` で解決する。
- `fills` の県名キーは地方キーより優先される(地方全体を塗ってから一部の県だけ別の色にできる)。
- クラウド環境から取得できなかったサイトを別経路で取りに行かない(取得できない旨を伝える)。

## 拡張するときの指針

- プリセットを足すときは `PRESETS` に `title` `status`(`official` / `common`) `source` `regions` を書く。`official` にしてよいのは、出典の構成を実際に確認できたものだけ。
- 配色を変えるときは `palette`(色の配列)を使う。地方数が12を超えると色が循環する。
- 地点(都市の点)は `points`: `{"name":"東京","lon":139.69,"lat":35.69,"color":"#d62728","label":true}`。
- 線の太さや色を変えるときは、スクリプトの `LW_BORDER` `LW_COAST` `LW_REGION` `INK` を直す。