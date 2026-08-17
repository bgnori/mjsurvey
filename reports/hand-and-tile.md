このファイルでは、手牌と牌の表現について記述する。

# 調査対象
mapping.txtを参照のこと。それぞれdirectoryが掘られ、git submoduleで対象が取得済みである。

Mortal https://github.com/Equim-chan/Mortal
mjx https://github.com/mjx-project/mjx
mjai-manue https://github.com/gimite/mjai-manue
Majiang https://github.com/kobalab/Majiang
RiichiEnv https://github.com/smly/RiichiEnv

# 調査結果
## Mortal
牌は [`Tile`](../Mortal/libriichi/src/tile.rs) という `u8` の newtype で表現される。通常の 34 種に加えて赤5 (`5mr`, `5pr`, `5sr`) と未知牌 (`?`) を含む 38 種の mjai 牌IDを持ち、文字列表現は `1m`、`E`、`5mr` のような mjai形式である。`deaka`、`akaize`、`is_aka` により、赤牌を通常の5と相互変換・判定できる。

手牌の計算用表現は [`hand.rs`](../Mortal/libriichi/src/hand.rs) の `hand` / `hand_with_aka` が示す通り、牌種ごとの枚数を持つ配列である。赤牌を区別する場合は 37 要素（34種 + 赤5 3種）、区別しない場合は 34 要素に集約する。ゲーム状態の [`PlayerState`](../Mortal/libriichi/src/state/player_state.rs) では、自家の手牌 `tehai` を赤牌を含めない `[u8; 34]` の牌種別カウントで保持し、赤牌の有無は `akas_in_hand: [bool; 3]` に分離する。鳴きは `chis`、`pons`、`minkans`、`ankans` などの牌種配列として別管理され、mjaiイベントでは `consumed: [Tile; N]` として物理的に消費した牌を渡す。

## mjx
牌は [`Tile`](../mjx/mjx/tile.py) でラップされた 0〜135 の物理牌IDである。同じ牌種の4枚に連続したIDを割り当て、`id // 4` で牌種 (`TileType`, 0〜33) を得る。赤牌は万子5・筒子5・索子5の物理ID 16、52、88で判定するため、牌種と物理牌を区別できる。

手牌は [`Hand`](../mjx/mjx/hand.py) の C++実装をPythonからラップしており、`closed_tiles()` は物理牌IDのリスト、`closed_tile_types()` は34種の牌種リストを返す。プロトコル上も [`mjx.proto`](../mjx/include/mjx/internal/mjx.proto) の `Hand` は `closed_tiles` と `opens` に分離される。鳴き (`Open`) は Tenhou形式のビット列として1面子を1整数に符号化し、[`open.py`](../mjx/mjx/open.py) の `tiles`、`tiles_from_hand`、`stolen_tile` などで復号する。したがって、手牌は配列、鳴きは圧縮された面子コード、牌は物理IDという三層構造である。

## mjai-manue
牌は [`Pai`](../mjai-manue/coffee/pai.coffee) オブジェクトで、牌種ID (`id`) は `m`、`p`、`s` 各9種と字牌7種の計34種（0〜33）である。文字列は `1m`〜`9s`、`E/S/W/N/P/F/C` を使い、赤牌は通常の牌種IDを変えず `red` フラグで表す。従って `equal` は牌種と赤フラグの両方、`hasSameSymbol` は牌種だけを比較する。

手牌や山は [`PaiSet`](../mjai-manue/coffee/pai_set.coffee) の牌種別カウント配列で管理する。`toPais` でカウントから `Pai` の配列へ戻せるが、`PaiSet` 自体は赤牌を別カウントしないため、赤牌の個体性が必要な処理では `Pai` 配列を使う。鳴きは手牌カウントとは別に、mjaiの `pai` と `consumed` を使うイベント／AI側の構造で扱う設計で、MortalやRiichiEnvのような専用の `Meld` 型は確認できない。

## Majiang
本リポジトリの [`majiang.js`](../Majiang/src/js/majiang.js) は牌・手牌のコア実装を直接含まず、`@kobalab/majiang-core` を `global.Majiang` に公開する構成である（依存バージョンは [`package.json`](../Majiang/package.json) の `@kobalab/majiang-core`）。本体コードから確認できる公開表現は、`Majiang.Shoupai` の文字列記法である。

手牌は `m123p123s123z11` のように、数字をまとめて `m`（萬子）、`p`（筒子）、`s`（索子）、`z`（字牌）を付ける文字列で表す。赤5は `0m`、`0p`、`0s` として表し、通常の5と同じ牌種の中で赤の個体性を表現する。`Shoupai.fromString` / `toString` がこの形式を入出力し、内部では牌種別の `bingpai` カウントに加えて、鳴きの配列 `fulou`、ツモ牌 `_zimo`、立直状態などを持つ。実際に [`paili.js`](../Majiang/src/js/paili.js) や [`hule.js`](../Majiang/src/js/hule.js) では `Shoupai.fromString`、`toString`、`_zimo`、`_fulou` を利用している。鳴きは `m123-`、`p1-23`、`z222=` のように文字列中へ埋め込まれ、呼び出し元や向きを記号で保持する。

## RiichiEnv
牌はRustコアおよびPython APIで 0〜135 の物理牌ID (`int`) を使う。牌種は `tile // 4` で求め、赤牌は各色の5の特定の物理ID（16、52、88）として保持される。これは [`HandEvaluator`](../RiichiEnv/src/riichienv/hand.py) の `tiles_136`、`_tiles_to_string`、および `Meld.tiles` に現れる。

手牌は閉じた牌の物理IDリスト `tiles_136` と、鳴きの [`Meld`](../RiichiEnv/src/riichienv/_riichienv.pyi) リストに分離される。`Meld` は `meld_type`（Chi、Pon、Daiminkan、Ankan、Kakan）、構成牌 `tiles`、公開／暗槓を表す `opened`、鳴いた相手 `from_who` を持つ。[`hand_from_text`](../RiichiEnv/src/riichienv/hand.py) は `123m405p...` のような文字列表現をRustの `parse_hand` で物理牌へ変換し、`to_text` は閉じた牌と面子を再構成する。ただし現状のPython側 `to_text` は `Meld` に呼び出し位置が保存されないため、鳴きの元文字列を完全には復元せず、近似的な面子文字列を生成する。