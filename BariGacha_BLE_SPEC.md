# 魔法のバリフリガチャ BLE通信仕様 / BariGacha BLE Communication Spec

魔法のバリフリガチャ（BariGacha）は、重度肢体不自由者がスイッチひとつでガチャガチャを楽しめるように開発した電動ガチャ制御装置です。  
BLE GATT を使ってスマートフォン・タブレット・PCのブラウザからロック解除・ガチャ回転・状態確認を行えます。

---

## 1. BLE 接続情報

| 項目 | 値 |
|------|-----|
| デバイス名プレフィックス | `BariGacha-BLE-XXXX` ※末尾4文字は MAC アドレス下位2バイト（16進大文字） |
| Service UUID | `A6B00040-BAFE-4B52-8E2D-9B1C3D4E5F60` |
| Command Characteristic UUID | `A6B00041-BAFE-4B52-8E2D-9B1C3D4E5F60` |
| State Characteristic UUID | `A6B00042-BAFE-4B52-8E2D-9B1C3D4E5F60` |

### Characteristic プロパティ

| Characteristic | プロパティ | 方向 |
|---------------|-----------|------|
| Command (`...0041`) | Write / Write Without Response | クライアント → BariGacha |
| State (`...0042`) | Read / Notify | BariGacha → 全接続クライアント |

---

## 2. コマンド仕様（クライアント → BariGacha）

Command Characteristic へ **Write** することで BariGacha を操作します。

### 基本フォーマット

```
#BFGC<コマンド名>!
```

| フィールド | 内容 |
|-----------|------|
| `#` | STX（コマンド開始） |
| `BFGC` | Barrier Free Gacha Command の略 |
| `<コマンド名>` | 下記コマンド一覧参照 |
| `!` | ETX（コマンド終了） |

**エンコード: UTF-8 テキスト**

### コマンド一覧

| コマンド文字列 | 内容 |
|--------------|------|
| `#BFGCUNLOCK!` | ロック解除（ガチャ操作を可能にする） |
| `#BFGCLCK!` | 再ロック（ガチャ操作を禁止する） |
| `#BFGCPRE!` | ドラムロール予告（ガチャ演出開始） |
| `#BFGCMOVE!` | ガチャ回転開始 |
| `#BFGCSTOP!` | ガチャ回転停止 |

---

## 3. 状態通知仕様（BariGacha → クライアント）

State Characteristic を **Subscribe（Notify）** することで BariGacha の状態変化をリアルタイム受信できます。  
接続中のすべてのクライアントへ同時に通知されます。

### 基本フォーマット

```
#BFGS<状態名>!
```

| フィールド | 内容 |
|-----------|------|
| `#` | STX（通知開始） |
| `BFGS` | Barrier Free Gacha State の略 |
| `<状態名>` | 下記状態一覧参照 |
| `!` | ETX（通知終了） |

### 状態一覧

| 通知文字列 | 状態 | 内容 |
|-----------|------|------|
| `#BFGSLOCK!` | LOCK | ロック中（操作不可） |
| `#BFGSUNLOCK!` | UNLOCK | アンロック待機中（操作可能） |
| `#BFGSPREMOVE!` | PREMOVE | ドラムロール予告中 |
| `#BFGSMOVE!` | MOVE | ガチャ回転中 |
| `#BFGSGET!` | GET | カプセル排出検出 |

> **切断状態（DISCONNECT）** は BariGacha からの通知値ではなく、クライアント側が BLE 切断を検出した際にローカルで設定する状態です。

---

## 4. 状態遷移

```
[LOCK]
  │  #BFGCUNLOCK! を受信
  ▼
[UNLOCK]
  │  #BFGCPRE! を受信
  ├──────────────────► [PREMOVE]
  │                       │ #BFGCMOVE! を受信
  │  #BFGCMOVE! を受信    ▼
  ├──────────────────► [MOVE]
  │                       │ カプセル排出センサー検出
  │                       ▼
  │                    [GET]
  │                       │ 3秒後に自動遷移
  │  #BFGCLCK! を受信     ▼
  └──────────────────── [LOCK]
```

---

## 5. マルチ接続

- BariGacha は最大 3 クライアントの同時 BLE 接続を許可します
- 接続中も広告を継続するため、追加クライアントの接続が可能です
- コマンドは接続中の任意クライアントから受け付けます
- 状態変化時は全接続クライアントへ同時に Notify します

---

## 6. 実装サンプル（Web Bluetooth API / JavaScript）

```javascript
const SVC_UUID   = 'a6b00040-bafe-4b52-8e2d-9b1c3d4e5f60';
const CMD_UUID   = 'a6b00041-bafe-4b52-8e2d-9b1c3d4e5f60';
const STATE_UUID = 'a6b00042-bafe-4b52-8e2d-9b1c3d4e5f60';

let cmdChar = null;

// BLE 接続 + 状態通知の購読
async function connect() {
  const device = await navigator.bluetooth.requestDevice({
    filters: [{ namePrefix: 'BariGacha-BLE-' }],
    optionalServices: [SVC_UUID]
  });
  const server    = await device.gatt.connect();
  const service   = await server.getPrimaryService(SVC_UUID);
  cmdChar         = await service.getCharacteristic(CMD_UUID);
  const stateChar = await service.getCharacteristic(STATE_UUID);

  // 状態通知の購読
  await stateChar.startNotifications();
  stateChar.addEventListener('characteristicvaluechanged', onStateChanged);

  device.addEventListener('gattserverdisconnected', () => {
    console.log('切断されました');
    cmdChar = null;
  });
}

// 状態通知のハンドラ
function onStateChanged(event) {
  const state = new TextDecoder().decode(event.target.value);
  console.log('状態:', state);
  // 例: '#BFGSLOCK!' / '#BFGSUNLOCK!' / '#BFGSMOVE!' / '#BFGSGET!' など
}

// コマンド送信
async function sendCommand(cmd) {
  if (!cmdChar) return;
  await cmdChar.writeValueWithoutResponse(new TextEncoder().encode(cmd));
}

// 使用例
await connect();
await sendCommand('#BFGCUNLOCK!');  // ロック解除
await sendCommand('#BFGCMOVE!');   // ガチャ回転開始
await sendCommand('#BFGCSTOP!');   // 回転停止
await sendCommand('#BFGCLCK!');    // 再ロック
```

---

## 7. 注意事項

- **iOS / iPadOS**: Safari は Web Bluetooth API 非対応です。**[Bluefy](https://apps.apple.com/app/bluefy-web-ble-browser/id1492912960)**（App Store）をお使いください。
- **Android / Windows**: Chrome ブラウザで Web Bluetooth API が利用可能です。
- **macOS**: Chrome / Edge で Web Bluetooth API が利用可能です。
- デバイス名末尾4文字（例: `AD3E`）で複数台の BariGacha を識別できます。
- State Characteristic は接続後に現在の状態を Notify します。明示的に現在状態を取得したい場合は Read も可能です。

---

## 8. 参考

- 魔法のバリフリガチャ 紹介ページ: https://protopedia.net/prototype/4728
