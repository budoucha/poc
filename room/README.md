# knock — サーバレス P2P マッチングデモ

ブラウザ単体で動作する WebRTC (Web Real-Time Communication) P2P (Peer-to-Peer) 接続のミニマムデモ。シグナリングサーバを持たず、URL フラグメントに SDP (Session Description Protocol) を埋め込んで人手で受け渡す方式で接続を確立する。接続後は DataChannel を通じて「ノック」メッセージを送り合い、相手側で画面フラッシュとトースト通知を表示する。

## 構成

`index.html` と、ホスト側の QR 読み取りフォールバック用に同梱した `vendor/jsQR.js` で構成する。外部 CDN やビルドプロセスには依存しない。

## 全体フロー

```
[Host]                                    [Guest]
   |                                         |
   | (1) createOffer + gather ICE            |
   | -- offer URL / QR (#offer=...) ---->    |
   |                                         |
   |                          (2) setRemoteDescription
   |                              createAnswer + gather ICE
   |     <------- answer payload / QR -------- |
   |                                         |
   | (3) paste or scan answer QR              |
   |     setRemoteDescription                |
   |                                         |
   | ============ DataChannel open =========  |
   |                                         |
   | -- {type:'knock'} -------------------->  |
   |                            (画面フラッシュ + トースト)
```

## シグナリング方式

WebRTC 接続の確立には、両ピアが SDP (offer/answer) と ICE (Interactive Connectivity Establishment) 候補を交換する必要がある。通常はこれをサーバ経由のシグナリングチャネルで行うが、本デモでは以下の方法でサーバを排除している。

1. Offer は `CompressionStream('gzip')` で JSON 文字列を圧縮し、`offer=oz1:...` として URL の `#` フラグメントに埋め込む
2. ホスト側はそのフル URL を QR コードとして表示するため、ゲストはカメラアプリで読み取ってすぐ開ける
3. Answer は同じく gzip 圧縮し、`answer=az1:...` の短いペイロードとして表示する
4. ゲスト側は同じ answer ペイロードを QR コードとして表示する
5. ホスト側は answer ペイロードを貼り付けるか、画面内のカメラ読み取り、または QR 画像読み取りで接続する

`#` 以降の文字列はブラウザの HTTP リクエストに含まれないため、URL を静的ホスティング経由で開いてもサーバ側に SDP が漏れない。

`CompressionStream` 非対応ブラウザでは、offer / answer ともに従来の URL-safe base64 形式にフォールバックする。ホスト側の貼り付け欄は full URL、`answer=...`、`answer=az1:...`、base64 本体のいずれも受け付ける。

### Non-trickle ICE

ICE 候補の収集には数百ミリ秒〜数秒かかる。本デモでは `pc.iceGatheringState === 'complete'` まで待ってから SDP を URL に書き出す（**non-trickle ICE**）。これにより一回の URL 交換で全候補が相手に届く。

```javascript
function waitIce(pc) {
  return new Promise(resolve => {
    if (pc.iceGatheringState === 'complete') return resolve();
    pc.addEventListener('icegatheringstatechange', () => {
      if (pc.iceGatheringState === 'complete') resolve();
    });
    setTimeout(resolve, 3000); // フォールバック
  });
}
```

3 秒のフォールバックタイムアウトは、STUN サーバが応答しない場合や ICE gathering が `complete` イベントを発火しない一部ブラウザ実装への対策。

### URL の長さ

典型的な SDP は 1.5〜3 KB 程度。base64 でも 4 KB 強に収まり、現代のブラウザの URL 長制限（Chrome 系で約 32 KB、Safari でも 80 KB 程度）には十分収まる。

Offer 側はカメラアプリでそのまま開けるように full URL を QR 化する。Answer 側は QR 化しやすくするため、full URL ではなく `answer=` 部分だけを返送対象にする。どちらも gzip 圧縮しており、SDP の情報自体は削らないため接続条件は変わらない。QR は静的 HTML 内の生成処理で描画しており、外部 CDN には依存しない。QR のサイズは内容量に応じて最小のバージョンを選ぶ。

ホスト側の QR 読み取りは `getUserMedia` でカメラ映像を取得し、ブラウザ標準の `BarcodeDetector` が使える場合はそれを優先する。`BarcodeDetector` が無い環境では、同梱した `jsQR` で動画フレームを解析する。`getUserMedia` 自体が使えない環境では、QR 画像を選択して `jsQR` で読み取れる。

## DataChannel

接続確立後、Host 側が `pc.createDataChannel('knock')` で開いたチャネルを使う。Guest 側は `pc.addEventListener('datachannel', ...)` でリモート由来のチャネルを受け取る。

メッセージは JSON 1 種類のみ：

```json
{"type": "knock", "t": 1718870400000}
```

受信側は `dc.onmessage` でハンドルし、画面端のオレンジフラッシュとトースト通知、可能なら `navigator.vibrate(50)` を発火する。

## NAT 越えと接続成立条件

本デモは STUN サーバとして Google 公開 STUN (`stun:stun.l.google.com:19302`) を 1 つだけ指定しており、TURN (Traversal Using Relays around NAT) は設定していない。これによる接続成功率は以下のとおり：

| シナリオ                              | 接続可否            |
|---------------------------------------|---------------------|
| 同一マシンの 2 タブ                   | 確実 (loopback)      |
| 同一 LAN の 2 端末                    | ほぼ確実 (host candidate) |
| LAN ↔ インターネット (Full/Restricted Cone NAT) | 成立 (srflx candidate) |
| LAN ↔ インターネット (Symmetric NAT)  | 不可                |
| UDP (User Datagram Protocol) ブロックされた企業ネットワーク | 不可                |
| CGNAT (Carrier-Grade NAT) 環境のモバイル回線 | 大半が不可          |

完全な NAT 突破には TURN サーバの導入が必要。`iceServers` 配列に `{urls: 'turn:...', username: '...', credential: '...'}` を追加するだけで TURN 経由のリレーが選択肢に入る。

接続経路の検証には Chrome 系の `chrome://webrtc-internals/` で ICE candidate pair の選択状況を確認できる。

## ホスティングについて

本ファイルは静的 HTML なので、両端が同じ URL にアクセスできれば配信方法は問わない。

| 配置先                     | 可否 | 備考                                   |
|---------------------------|------|----------------------------------------|
| `file://` でローカル開く  | 一部 | WebRTC は動くが clipboard API やカメラ API が secure context 必須のため Chrome では制限あり |
| `python -m http.server` + `http://localhost:8000/` | ◯ | 単一端末またはホストPC上のカメラテスト向け |
| ローカル PC + LAN 内アクセス | △    | `http://192.168.x.x:8000/` 形式では多くのブラウザでカメラ API が無効。QR 画像読み取りや貼り付けは利用可能 |
| GitHub Pages / Cloudflare Pages | ◎ | HTTPS なので LAN 越境テストとカメラ読み取りに適する |

「マッチングサーバ」と呼べるものは存在しない。HTML 配信に使うサーバは状態を持たない単なる静的ファイルサーバである。

## ブラウザ互換性

`RTCPeerConnection`, `RTCDataChannel`, `navigator.clipboard` を使用。主要モダンブラウザ（Chrome 90+, Firefox 88+, Safari 14+, Edge 90+）で動作確認可能。Safari は ICE candidate の gathering 挙動がやや異なる場合があるが、本デモのフォールバックタイムアウトでカバーされる。

ホスト側の answer QR ライブ読み取りにはカメラ権限と secure context が必要。`BarcodeDetector` 非対応ブラウザでは同梱の `jsQR` フォールバックで読み取る。カメラ API が使えない場合は、QR 画像読み取りか貼り付け入力を使う。

## 既知の制約

- **片方向の URL 交換のみで完結しない**：offer URL と answer ペイロードの往復が必要。1 URL 招待にしたい場合は別途シグナリングチャネル（公開リレーサーバ等）が必要
- **接続後の再接続不可**：ページリロードで状態が消える
- **DataChannel 1 本のみ**：複数チャネルでの優先度制御等は未実装
- **エラーハンドリングが最小限**：SDP 解析失敗時の `alert()` のみ
- **タイムアウト未実装**：相手が answer を返さない場合の自動キャンセルはない
- **QR 読み取りの互換性**：ホスト側スキャンは `BarcodeDetector` または同梱 `jsQR` で実行する。カメラ API が使えない環境では貼り付け入力に戻る

## 拡張のヒント

- **TURN 統合**：`iceServers` に TURN エントリを追加し、Symmetric NAT 環境をサポート
- **QR (Quick Response) コード共有**：offer は full URL QR、answer は短縮ペイロード QR で対面マッチングを簡略化
- **SDP 圧縮**：offer / answer ともに gzip 圧縮。さらに短縮したい場合は candidate フィルタリング等を検討
- **複数チャネル**：チャット用・状態同期用・バイナリ転送用などを分離
- **ルーム概念**：公開シグナリング（BitTorrent tracker、Nostr リレー等）を組み合わせれば、URL 交換不要の自動マッチングに発展可能
