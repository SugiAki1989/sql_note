## :memo: Overview

ここでは Google Tag Manager でデータを生成する方法についてまとめておく。SQL関係ないのでは？と思った人もいるが、GA4のデータをBigQueryに連携したり、各種マーケティングツールでは、JavaScriptを書いて、ユーザーの行動をデータとして取得することも行われる。

そのような時に、データを生成する部分から知っておいても損はないので、ここでは Google Tag Managerを利用して、Webサイトの基本的なユーザー行動をデータとして取得する方法について学んだ内容をまとめておく。内容が誤っている可能性もある、かつ、JavaScriptも久々に書いたので、こちらも効率の良い書き方になってないかもしれない。

## :floppy_disk: Database

Google Tag Manager

## :bookmark: Tag

`javascript`

## :pencil2: Google Tag Manager

Google Tag ManagerはWebサイトにJavaScriptなどを簡単に追加、管理できるサービス。Google Tag ManagerのタグがWebサイトに挿入されていれば、Webサーバーに保存されているHTML、JavaScriptのファイルなどを修正しなくても、Google Tag ManagerからJavaScriptを追加できるので、Webサイトの機能を拡張できる。Google Tag ManagerではJavaScriptを必ずしも記述する必要はないが、複雑な機能を実装する場合などはJavaScriptを書く必要がでてくる。

ここではjimdoで作成した[ダミーサイト(内容に意味はない)](https://tokyo2kyoto.jimdofree.com/)を利用して、Google Tag ManagerやGA4の機能を確認する。

![DummySite](https://github.com/SugiAki1989/sql_note/blob/main/image/p133-hover_photo0.png)

## :pencil2: 基本的な使い方

まずはGoogle Tag Managerの管理画面から、タグを新規発行することで、ダミーサイトのコンソールに文字列を出力する。簡単な例を通して、Google Tag Managerの設定自体に問題がないかを検証する。

- タグの設定: カスタムHTML
- トリガー: All Pages

```
<script>
console.log('GTM tags are working fine');
</script>
```

ダミーサイトにアクセスして、コンソールにこの文字列が表示されていればOK。このWebサイトには、移動手段に関する商品ページ(`/transportation/`)が用意されているので、そのページの商品名と価格を取得してみる。ここでは、3つ商品がある中の`WALKING`と`¥ 0`を取得する。また、トリガーは`/transportation/`にアクセスしたとき。`/transportation/walking/`というページもあるので、最後が一致にしている。

- タグの設定: カスタムHTML
- トリガー: Page View 
  - 設定: PagePath - 最後が一致 - `/transportation/`

```
<script>
var name = document.querySelector('#cc-m-header-14474180327').textContent
var price = document.querySelector('#cc-m-14474219327').textContent.trim()
console.log('transfer: ' + name)
console.log('price: ' + price)
</script>
```

## :pencil2: GA4のイベントをデバッグする

基本的なGoogle Tag Managerの使い方はわかったので、GA4にイベントを送る方法を確認しておく。`gtag`という関数を利用することで、GA4にイベントを送ることが出来るようになるので、あとはこの関数に送りたいイベントの値を取得して、引数に渡せば良い。

ここでは、TOPページにある東京の画像をホバーしたら、ホバーした分だけイベントをGA4に連携するスクリプトを書く。Webページの特定の要素に対する特定のイベントが発生したタイミングでイベントを飛ばす処理が実務で必要かはさておき、とりあえず練習がてらやってみる。

- タグの設定: カスタムHTML
- トリガー: Page View 
  - 設定: PagePath - 等しい - `/`

```
<script>
var tokyo_img = document.querySelector('#cc-m-imagesubtitle-image-14474174927')
var tokyo_name = document.querySelector('#cc-m-header-14474175027').textContent

tokyo_img.addEventListener('mouseover', function(){
  gtag('event', 'hover_photo', {photo_name: tokyo_name})
})
</script>
```

GTMでイベントを収集するスクリプトを記述した際に、動作検証としてGoogle Analytics Debuggerという便利なChrome拡張がある。このChrome拡張は、自分のブラウザで行った動作(イベント)がGA4に問題なく連携されているのかを確認できるツール。使い方はChrome拡張をインストールして、検証したWebサイトで機能をONにすれば、検証ツールのコンソールにログが表示される。他にも、GA4の管理画面かも確認できる。管理画面の左下のギアアイコンから、検証したいプロパティのリストの中のDebugViewをクリックすると、Webサイトから送られてきているイベントを時系列で見やすく表示してくれる。

実際のイベントを発生せた際のデバッグ情報は下記の通り。

![Google Analytics Debugger](https://github.com/SugiAki1989/sql_note/blob/main/image/p133-hover_photo2.png)
![GA4 Debug View1](https://github.com/SugiAki1989/sql_note/blob/main/image/p133-hover_photo1.png)
![GA4 Debug View2](https://github.com/SugiAki1989/sql_note/blob/main/image/p133-hover_photo3.png)


特定の要素ではなく、特定の共通的な要素を持つ要素に対して、イベントリスナーを設定してみる。ここでは、東京、富士山、京都の画像の1枚づつ表示されているので、それをクリックしたら、写真の名前を使ってイベントネームを加工して、写真名とともにイベントを送信してみる。ちょっとJavaScriptがいまいちな感じなのはさておき、これでイベントを送信できる。

```
const photoContainers = document.querySelectorAll('#cc-m-14474174827 > .cc-m-hgrid-column > [id^="cc-matrix-"]');

photoContainers.forEach(photoContainer => {
  const photo = photoContainer.querySelector('.j-module.n.j-imageSubtitle');
  const photoNameElement = photoContainer.querySelector('.j-module.n.j-header');
  
  photo.addEventListener('click', () => {
    const photoName = photoNameElement.textContent;
    gtag('event', 'click_photo_'+photoName, {photo_name: photoName})
  });
});
```

![GA4 Debug View3](https://github.com/SugiAki1989/sql_note/blob/main/image/p133-hover_photo4.png)

## :closed_book: Reference

- [イベントを設定する](https://developers.google.com/analytics/devguides/collection/ga4/events?hl=ja&client_type=gtm)
- [e コマースを測定する](https://developers.google.com/analytics/devguides/collection/ga4/ecommerce?client_type=gtag&sjid=18231440941452277038-AP&hl=ja#add_or_remove_an_item_from_a_shopping_cart)
