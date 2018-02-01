### 表参道.rb #31
- - -
## LINE でお天気bot をつくった話

---

## 自己紹介
- - -

* なまえ : おしょー（[@pink_bangbi](https://twitter.com/pink_bangbi)）
* Ruby（not on Rails）
  * 役に立たない gem をつくってる
  * Ruby にパッチ投げつけてる
* C++（コンパイル時）
* JavaScript（Electron + Vue.js）
* Vim（のプラグインを量産）
* [Osushi](https://osushi.love/pink_bangbi)
 
---

## [今年作った gem](https://rubygems.org/profiles/osyo-manga)
- - -

* [toplevel](https://github.com/osyo-manga/gem-toplevel)
  * ファイルローカルメソッドみたいなのを定義する

---


## 今日話す内容
- - -

## LINE でお天気bot をつくった話

---

#### 今回つくる Bot
- - -

* LINE でお天気情報を返す Bot をつくる
* 『今日の東京の天気は』というような発言があった場合に『今日』の『東京』のお天気情報を返す

![](https://i.gyazo.com/84781b92e56d5444d7d9b243eb92a833.png)

---

## 今回利用するもの
- - -

* LINE Message API
  * LINE が提供している Bot で利用できる API
* Ruboty
  * Ruby 製のチャット Bot フレームワーク
* お天気情報などを取得するための API
* サーバ（heroku）

---

# Ruboty について

---

## Ruboty とは
- - -

* Ruby でチャットBot を作るためのフレームワーク      <!-- .element: class="fragment" -->
* 各種チャットサービスと Ruby を接続する部分（アダプタ）と実際に発言を拾って処理を行う部分（ハンドラ）を分けて作ることが出来る    <!-- .element: class="fragment" -->
* アダプタとハンドラが分かれていることでそれらを組み合わせて Bot を構築する事が出来る    <!-- .element: class="fragment" -->
* 他人が作成したハンドラを簡単に取り込んだりすることが出来るので汎用性が高い    <!-- .element: class="fragment" -->

>>>

#### アダプタ
- - -

各種チャットサービスと Ruby を接続する部分

* [ruboty-slack](https://github.com/r7kamura/ruboty-slack)
* [ruboty-twitter](https://github.com/r7kamura/ruboty-twitter)
* [ruboty-chatwork](https://github.com/mhag/ruboty-chatwork)
* [ruboty-discord](https://github.com/ykzts/ruboty-discord)

>>>

#### ハンドラ
- - -

発言を受け取ってパースし、処理を行う部分

* [ruboty-youtube](https://github.com/kaihar4-archive/ruboty-youtube)
  * Youtube で検索
* [ruboty-github](https://github.com/r7kamura/ruboty-github)
  * github-issues を立てたり
* [ruboty-vimhelp](https://github.com/osyo-manga/gem-ruboty-vimhelp)
  * Vim の :help したり
* etc...

>>>

#### その他のプラグイン
- - -

* [ruboty-alias](https://github.com/r7kamura/ruboty-alias)
  * 発言をエイリアスしたり
* [ruboty-redis](https://github.com/r7kamura/ruboty-redis)
  * 記憶をRedisに永続化したり

---

## 今回 Ruboty でつくるもの
- - -

* LINE の Message API アダプタ
* 発言をパースしてお天気情報を返すハンドラ

---

## LINE の Message API を使う

---

## LINE の Message API
- - -

* LINE の [Messaging API](https://developers.line.me/ja/docs/messaging-api/overview/) というサービスを使って LINE 上の Bot とやり取りを行う
* 作成した Bot に Webhook URL を登録することで、その URL に発言などが JSON 形式で POST される
* 詳しくは Messaging API のサイトを読んでね

>>>

![](https://i.gyazo.com/bb2bf6e978793eaa155cce81e6488d34.png)

---


## Message API を使ってみる
- - -

以下の言語では API 側で SDK が用意されているので簡単に使うことが出来る

* [Java](https://github.com/line/line-bot-sdk-java)
* [PHP](https://github.com/line/line-bot-sdk-php)
* [Go](https://github.com/line/line-bot-sdk-go)
* [Perl](https://github.com/line/line-bot-sdk-perl)
* [Ruby](https://github.com/line/line-bot-sdk-ruby)
* [Python](https://github.com/line/line-bot-sdk-python)
* [Node.js](https://github.com/line/line-bot-sdk-nodejs)

>>>

Ruby だと sinatra を使ったサンプルが書かれている

```ruby
# app.rb
require 'sinatra'
require 'line/bot'

def client
  @client ||= Line::Bot::Client.new { |config|
    config.channel_secret = ENV["LINE_CHANNEL_SECRET"]
    config.channel_token = ENV["LINE_CHANNEL_TOKEN"]
  }
end

post '/callback' do
  body = request.body.read

  signature = request.env['HTTP_X_LINE_SIGNATURE']
  unless client.validate_signature(body, signature)
    error 400 do 'Bad Request' end
  end

  events = client.parse_events_from(body)
  events.each { |event|
    case event
    when Line::Bot::Event::Message
      case event.type
      when Line::Bot::Event::MessageType::Text
        message = {
          type: 'text',
          text: event.message['text']
        }
        client.reply_message(event['replyToken'], message)
      when Line::Bot::Event::MessageType::Image, Line::Bot::Event::MessageType::Video
        response = client.get_message_content(event.message['id'])
        tf = Tempfile.open("content")
        tf.write(response.body)
      end
    end
  }

  "OK"
end
```

---

## ruboty-line をつくった
- - -

* Message API を使った ruboty のアダプタをつくった
* [ruboty-line](https://github.com/osyo-manga/gem-ruboty-line)
* 以下の環境変数を設定しておくだけで使える（はず

```
RUBOTY_LINE_CHANNEL_SECRET - YOUR LINE BOT Channel Secret.
RUBOTY_LINE_CHANNEL_TOKE   - YOUR LINE BOT Channel token.
RUBOTY_LINE_ENDPOINT       - LINE bot endpoint(Callback URL). (e.g. '/message/reply'
LOG_LEVEL                  - Use Ruboty logger. If LOG_LEVEL=0, output debug log.
```

---

## 注意点
- - -

* LINE の Bot API は 2016年4月頃にトライアル版として公開された
* その後、2016年10月頃に LINE Message API が正式版としてリリースされた
* トライアル版と正式版では微妙に仕様が違うのでネットの記事などを参照する場合は注意する

---

## OpenWeatherMap から
## お天気情報を取得する

---

## ruboty-japan_weather を使う(？)
- - -

ruboty-japan_weather というまさにお天気を表示するためのハンドラがハンドラが存在する

```
> @ruboty 明日の秋葉原の天気教えて
東京都 千代田区の明日の天気は曇時々雨 最高気温25度 最低気温17度 降水確率は60% です。
> @ruboty きょうのテヘランの天気は？
イラン テヘランの現在の天気は(17時00分 現在)晴時々曇 です。
```

しかし、残念ながら内部で使用されている[Microsoft の MSN 天気サービスの API](https://support.microsoft.com/ja-jp/help/3210143/msn-weather-service-api-has-been-retired)が2016年で終了しており動作しなかった…(´・ω・`)

---

## しょうがないので自作する

---

## OpenWeatherMap を使う
- - -

OpenWeatherMap が公開している API を使う  
API を使うには OpenWeatherMap にアカウント登録して API キーを取得する必要がある

```ruby
# 今の天気を取得
url = "http://api.openweathermap.org/data/2.5/weather"
request = {
	q:  "shinjuku",							# 取得したい地域名
	APPID: ENV["OPEN_WEATHER_MAP_APPID"],	# OpenWeatherMap のキー
	lang: "ja"								# 言語
}

res = Faraday.get url, request
result = JSON[res.body, symbolize_names: true]
pp result
```

>>>

```
{:coord=>{:lon=>139.71, :lat=>35.7},
 :weather=>[{:id=>500, :main=>"Rain", :description=>"小雨", :icon=>"10d"}],
 :base=>"stations",
 :main=>
  {:temp=>278.96,
   :pressure=>1025,
   :humidity=>60,
   :temp_min=>278.15,
   :temp_max=>280.15},
 :visibility=>16093,
 :wind=>{:speed=>2.6, :deg=>40},
 :clouds=>{:all=>90},
 :dt=>1517457360,
 :sys=>
  {:type=>1,
   :id=>7622,
   :message=>0.0319,
   :country=>"JP",
   :sunrise=>1517434892,
   :sunset=>1517472497},
 :id=>1850144,
 :name=>"Shinjuku",
 :cod=>200}
```

---

# 問題点

---

## OpenWeatherMap に渡す地名
- - -

* 地名はローマ字にする必要がある
* Bot に対しての発言は日本語なのでそれをローマ字に変換する必要があるが…
  * 新宿 → shinhuku
  * 八王子 → hachioji(hachiouji ではない)
* 表参道(omotesando)など検知しない地名もある…
* つらたん…

---

#### ジオコーディングを利用する
- - -

* ジオコーディングとは住所から地理的座標（緯度経度）に変換する技術
* Google Maps API を利用することで簡単に変換する事が出来る

```ruby
url = "http://api.openweathermap.org/data/2.5/weather"
request = {
	# 横浜市の緯度経度
	lat: 35.4437078,		# 緯度
	lon: 139.6380256,		# 経度
	APPID: ENV["OPEN_WEATHER_MAP_APPID"],	# OpenWeatherMap のキー
	lang: "ja",								# 言語
}

res = Faraday.get url, request
result = JSON[res.body, symbolize_names: true]
```

>>>

```
{:coord=>{:lon=>139.64, :lat=>35.44},
 :weather=>[{:id=>500, :main=>"Rain", :description=>"小雨", :icon=>"10d"}],
 :base=>"stations",
 :main=>
  {:temp=>277.66,
   :pressure=>1024,
   :humidity=>86,
   :temp_min=>277.15,
   :temp_max=>278.15},
 :visibility=>10000,
 :wind=>{:speed=>6.7, :deg=>20},
 :clouds=>{:all=>75},
 :dt=>1517459640,
 :sys=>
  {:type=>1,
   :id=>7619,
   :message=>0.37,
   :country=>"JP",
   :sunrise=>1517434878,
   :sunset=>1517472544},
 :id=>1848354,
 :name=>"Yokohama-shi",
 :cod=>200}
```

---

## ruboty-tenki つくった
- - -

* [ruboty-tenki](https://github.com/osyo-manga/gem-ruboty-tenki)
* 日本語の地名からジオコーディングして座標を取得し、その座標を使用して OpenWeatherMap からお天気情報を取得するハンドラ
* 使用する場合は以下の環境変数が必要

```
RUBOTY_TENKI_GOOGLE_MAP_APIKEY      - Google Maps Geocoding API:https://developers.google.com/maps/documentation/geocoding/start
RUBOTY_TENKI_OPEN_WEATHER_MAP_APPID - OpenWeatherMap API:https://openweathermap.org/
```

>>>

```
> ruboty 今日の東京の天気は
日本、東京都東京
2018/02/01 の天気
00:00           雲    3℃   75%  1038hPa
03:00           雲    7℃   91%  1038hPa
06:00        曇りがち    6℃   82%  1037hPa
09:00          小雨    3℃   87%  1038hPa
12:00          小雨    2℃   92%  1039hPa
15:00          小雨    1℃   94%  1040hPa
18:00          小雨    0℃   96%  1039hPa
21:00           雪    0℃   92%  1039hPa
```

---


## 動作確認をしてみる
- - -

Ruboty には CLI のアダプタがあるのでそれを利用して CLI 上で動作確認することが出来る

```shell
# repl を起動する
$ bundle exec ruboty -l line.rb
```

---

#### 全体の流れのまとめ
- - -

0. LINE の Bot アカウントに対して『今日の東京の天気は』と発言する    <!-- .element: class="fragment" -->
0. 発言内容が Message API 経由で Bot アカウントに登録してある Webhook URL に POST される    <!-- .element: class="fragment" -->
0. 発言内容から『地名』と『日時』をパース     <!-- .element: class="fragment" -->
0. Google Maps API で『地名』から『座標』を取得     <!-- .element: class="fragment" -->
0. 取得した『座標』で OpenWeatherMap から数日間の天気の情報を取得     <!-- .element: class="fragment" -->
0. そこから指定された『日時』の天気を取得     <!-- .element: class="fragment" -->
0. Messaging API 経由で Bot にいい感じで返信     <!-- .element: class="fragment" -->

---

## 今後
- - -

* テキストじゃなくてもっといい感じに表示したい
* よく使う地域を登録しておいて毎回打ち込まないで表示したい
  * リッチメニューで表示したい

---

## まとめ
- - -

* Ruboty を使うことで汎用性が高い Ruby の Bot を作ることができる
* LINE の Message API を使えば比較的簡単に LINE Bot が作れる
* API に依存しているプラグインは突然死ぬことがあるのでつらい

---

## 参照リンク
- - -

* [Herokuでサンプルボットを作成する](https://developers.line.me/ja/docs/messaging-api/building-sample-bot-with-heroku/)
* [チャットボットフレームワーク Ruboty を振り返る – r7kamura – Medium](https://medium.com/@r7kamura/%E3%83%81%E3%83%A3%E3%83%83%E3%83%88%E3%83%9C%E3%83%83%E3%83%88%E3%83%95%E3%83%AC%E3%83%BC%E3%83%A0%E3%83%AF%E3%83%BC%E3%82%AF-ruboty-%E3%82%92%E6%8C%AF%E3%82%8A%E8%BF%94%E3%82%8B-be95e56d2400)
* [Rubotyが殺されるのを見届けた話 | task blog](http://task-blog.net/2015/12/24/ray-killed-by-dark/)
* [気象情報API比較してみた - Qiita](https://qiita.com/Barbara/items/93ae7969691164c7c2bc)

---

## ご清聴
## ありがとうございました
