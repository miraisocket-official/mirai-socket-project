# 自作OS開発のためのKnow言語（.kw）公式教科書・マスターデータ

## 第1章：Know言語の概要と設計思想

### 1.1 開発の背景と目的
Know言語（以下、Knowlang）は、独自OS開発（低レイヤ開発）における「C言語の圧倒的な制御力」を維持しつつ、「可読性の低さ」や「初心者泣かせの難解な構文」を克服するために設計されたシステム記述言語である。
OS開発に必要な厳密なメモリ制御機能を持ちながら、ソースコードの見た目から記号のトゲトゲしさを排除し、英語の文章のように直感的に意味を「知る（Know）」ことができる環境を提供する。

### 1.2 コア・コンセプト
* **Code less, Know more.**：頻出する命令や型宣言を3文字（または2文字）の英単語に統一。タイピング量を減らし、コードの意図を明快にする。
* **脱・難解なインラインアセンブラ**：CPUへの直接命令（`hlt`, `in`, `out`等）を記述する際、C言語のトゲトゲしたインラインアセンブラ構文（`__asm__`）を完全に排除。すべて綺麗な「組み込み関数」として言語が標準サポートする。
* **C言語トランスパイル方式**：ソースファイル（拡張子 `.kw`）は専用コンパイラ `knowc` によってクリーンなC言語（`.c`）に変換される。これにより、既存の強力なビルドツール（GCC、LD、Makefile等）の資産をそのまま活用できる。

---

## 第2章：基本構文と3文字のキーワード

Knowlangのコードは、3文字（一部2文字）のキーワードが規則的なテンポを生み出すように設計されている。

### 2.1 変数宣言 (`var`)
C言語の「型名が先に来る」記述を廃止し、変数であることを示す `var` とコロン `:` を用いる。
* **構文**: `var 変数名: 型名 = 初期値;`
* **例**: `var status: unsigned int = 1;`

### 2.2 画面出力 (`prf`)
OSのデバッグや情報出力において最も多用される `printf` 命令を3文字に短縮。
* **構文**: `prf("文字列");`
* **例**: `prf("Know OS Booted!\n");`

### 2.3 条件分岐 (`if` / `els`)
条件式を囲む括弧 `( )` を完全に無くした。また、`else` は3文字の `els` に統一する。
* **構文**:
  ```know
  if 条件式 {
      // 処理
  } els {
      // 処理
  }
  ```
* **例**:
  ```know
  if status == 1 {
      prf("Success\n");
  } els {
      prf("Error\n");
  }
  ```

### 2.4 無限ループ (`lop`) とループ制御 (`brk` / `cnt`)
OSのメイン処理や入力待ちで必須となる無限ループを、呪文のような `while(1)` ではなく、直感的な `lop` で記述する。ループの脱出は `brk`（break）、次の周回へのジャンプは `cnt`（continue）に3文字統一する。
* **構文**:
  ```know
  lop {
      if 条件 { cnt; }
      if 条件 { brk; }
  }
  ```

---

## 第3章：OS開発のための低レイヤ特殊構文

Knowlangの本領である、ハードウェアを直接制御するための厳密なメモリ操作機能。

### 3.1 読みやすい型キャスト (`as`)
C言語の `(型名)値` という括弧だらけのキャストは視認性が悪い。Knowlangでは、左から右へ流れるように読める `as` を使用する。
* **例**: `0x000b8000 as unsigned char*`
* **変換後C言語**: `(unsigned char*)(0x000b8000)`

### 3.2 最適化防止 (`vlt`)
メモリマップドI/O（VRAMへのピクセル書き込みなど）において、コンパイラが「同じ場所に何度も書き込む無駄なコード」と勘違いして処理を消去するバグを防ぐため、`volatile` を3文字の `vlt` として型に付与する。
* **例**: `var vram: vlt unsigned char* = 0x000b8000 as unsigned char*;`
* **変換後C言語**: `volatile unsigned char* vram = (unsigned char*)(0x000b8000);`

### 3.3 構造体とビット詰め (`pck struct`)
データをまとめる構造体（`struct`）の内部も `var` スタイルで統一。さらに、GDT（セグメント記述子）やIDT（割り込み記述子）などの設定で必須となる、メモリの隙間をゼロにしてギチギチに詰め込む命令（`packed`属性）を、構造体の頭に **`pck`** とつけるだけで実現する。
* **例**:
  ```know
  pck struct GDTDesc {
      var limit: unsigned short;
      var base: unsigned int;
  }
  ```
* **変換後C言語**:
  ```c
  struct __attribute__((packed)) GDTDesc {
      unsigned short limit;
      unsigned int base;
  };
  ```

### 3.4 組み込みCPU関数 (脱アセンブラ)
C言語のインラインアセンブラを関数として隠蔽。
* `hlt();` ：CPUを安全に待機状態にする（`__asm__("hlt");`）
* `out8(port, data);` ：指定ポートに8ビット書き込む
* `in8(port);` ：指定ポートから8ビット読み込む

---

## 第4章：実践サンプルコード（OSメイン処理）

本教科書で学んだ仕様をすべて投入した、Knowlangの標準的なOSメインプログラム（`main.kw`）の構造。

```know
// 画面情報やセグメント情報をメモリにギチギチに配置する構造体
pck struct BootInfo {
    var vram: vlt unsigned char*;
    var scrnx: short;
    var scrny: short;
}

// C言語ベースの関数宣言（体に馴染んだスタイル）
int main() {
    struct BootInfo binfo;
    binfo.vram = 0x000b8000 as unsigned char*;
    binfo.scrnx = 320;
    binfo.scrny = 200;

    // キーボードの状態ポートからデータを読み込む
    var status: unsigned int = in8(0x64);

    // 括弧のないスッキリした if / els 分岐
    if (status and 0x01) == 1 {
        prf("OS Status: Keyboard Ready.\n");
    } els {
        prf("OS Status: Warning.\n");
    }

    // 3文字の軽量ループで入力を監視
    lop {
        var key: unsigned char = in8(0x60); // スキャンコード取得
        
        if key == 0  { cnt; }  // 押されてなければ次のループへ
        if key == 27 { brk; }  // ESCキーが押されたらループを脱出（OS終了など）
        
        // VRAM（画面）に変数の値を書き込んで色を塗る
        binfo.vram = key;
        
        hlt(); // CPUを休ませて次の割り込みを待つ
    }

    return 0;
}
```

---

## 第5章：開発環境とビルドパイプライン

### 5.1 コンパイラ `knowc` の仕組み
Knowlangは、`knowc.c` という純粋なC言語の文字列処理プログラムによって動作する。ファイル（`stdin`）からコードを1行ずつ読み込み、上記で定義されたKnowlang特有のキーワードを、正規のC言語表現へと高速に置換・再構築して出力する。

### 5.2 全自動ビルド（Makefileサフィックスルール）
開発者が毎回コマンドを叩く手間を省くため、OSの `Makefile` に以下の自動変換ルールを定義する。
```makefile
%.c: %.kw knowc
	./knowc < < > @
```
これにより、`main.kw` を編集して `make` を実行するだけで、裏で自動的に `main.c` が生成され、そのままGCCによるOSバイナリビルドへと繋がる。
