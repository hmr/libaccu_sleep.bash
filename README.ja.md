# libaccu_sleep.bash

[English](README.md)

ループ実行向けの、比較的ずれにくい Bash タイマーヘルパーです。

`libaccu_sleep.bash` は、ループごとに毎回相対時間で sleep するのではなく、
積み上げた目標時刻に向かって待機する小さな Bash ライブラリです。
これにより、ループ本体の処理時間によって発生するドリフトを抑えやすくなります。

## 要件

- Bash 5 以降を推奨
- キー入力フックを使う場合は、`sleep`、`stty`、`/dev/tty` が使える POSIX 風の環境

積算スケジューリングには Bash の `EPOCHREALTIME` を使います。`EPOCHREALTIME` が
使えない場合、ライブラリは source 時に警告を表示し、`ACCU_SLEEP` は通常の `sleep`
呼び出しにフォールバックします。このフォールバックモードでは、積算スケジューリング、
ドリフト補正、キー入力フックは利用できません。

## 使い方

ライブラリを source してから、マイクロ秒単位の間隔を指定して `ACCU_SLEEP` を呼び出します。

```bash
#!/usr/bin/env bash

source ./libaccu_sleep.bash

while true; do
    printf '%s\n' "$EPOCHREALTIME"

    # 次の 1 秒周期の目標時刻まで待機する。
    ACCU_SLEEP 1000000
done
```

`ACCU_SLEEP` は内部で次の目標時刻を保持します。たとえばループ本体に 100 ms かかり、
間隔が 1 秒の場合、次の sleep 時間はその分短くなり、元の周期に沿って動き続けます。

`EPOCHREALTIME` が使えない場合、`ACCU_SLEEP` は指定された相対時間だけ待機し、
積算スケジュールは保持しません。

新しいスケジュールを開始する場合は `ACCU_SLEEP_RESET` を使います。

```bash
ACCU_SLEEP_RESET
```

## API

### `ACCU_SLEEP <interval_us>`

次の積算スケジュール時刻まで待機します。

- `interval_us` は、マイクロ秒単位の正の整数です。
- 不正な値の場合はエラーを表示し、`2` を返します。

### `ACCU_SLEEP_RESET`

積算スケジュールをクリアします。次の `ACCU_SLEEP` 呼び出しは現在時刻を基準に開始します。

## キー入力フック

`ACCU_SLEEP_ON_KEY` という名前の関数が定義されている場合、ライブラリは待機中に
`ACCU_SLEEP_TTY` から 1 文字ずつ読み取ります。デフォルトの入力元は `/dev/tty` です。

```bash
function ACCU_SLEEP_ON_KEY() {
    local key=$1

    case "$key" in
        q)
            ACCU_SLEEP_RESTORE_TTY
            exit 0
            ;;
    esac
}

trap 'ACCU_SLEEP_RESTORE_TTY' EXIT
```

入力元のパスは、ライブラリの source 前または使用前に変更できます。

```bash
ACCU_SLEEP_TTY=/dev/tty
source ./libaccu_sleep.bash
```

TTY アクセスは任意です。フックが定義されていない場合や、設定された TTY が読めない場合、
`ACCU_SLEEP` は通常の `sleep` にフォールバックします。

## サンプル

サンプルプログラムを実行すると、各 tick のタイミング情報を表示します。

```bash
./libaccu_sleep.sample
```

このサンプルはタイミング統計を表示するため、`EPOCHREALTIME` が必要です。

Bash 3.x 環境では、date ベースのサンプルを使います。

```bash
./libaccu_sleep.bash3.sample
```

`libaccu_sleep.sample` の操作:

- `Ctrl-x`: 現在の統計を表示して継続
- `Ctrl-c`: 統計を表示して終了

Bash 3 対応サンプルはキー入力を扱いません。GNU date のナノ秒出力が使える場合は
`date +%s.%N` を使い、使えない場合は `date +%s` にフォールバックします。
統計は 100 サンプルごと、および終了時に表示します。

## 開発

構文チェック:

```bash
bash -n libaccu_sleep.bash
bash -n libaccu_sleep.sample
bash -n libaccu_sleep.bash3.sample
/bin/bash -n libaccu_sleep.bash3.sample
```

ShellCheck:

```bash
shellcheck -x libaccu_sleep.bash libaccu_sleep.sample libaccu_sleep.bash3.sample
```

サンプルでは source 先のライブラリファイルを ShellCheck に追跡させるため、`-x` を使います。

## ライセンス

GPL-3.0。詳細は [LICENSE](LICENSE) を参照してください。
