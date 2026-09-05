# LightCP Editor 実装用詳細設計書
Version 0.1

---

# 1. 文書の目的

本書は、軽量な競技プログラミング向けコードエディタ「LightCP Editor」を実装するための詳細設計を定義する。

対象は主として Windows 環境とし、C++ によるネイティブアプリケーションとして実装する。

本書では以下を定義する。

- ソースコード構成
- モジュール責務
- クラス設計
- データ構造
- イベント処理
- 描画方式
- Text Buffer
- Undo/Redo
- ファイル入出力
- 構文ハイライト
- 検索
- 外部プロセス実行
- 設定
- テスト
- 性能計測
- 実装フェーズ

---

# 2. 基本設計方針

## 2.1 最重要目標

本プロジェクトでは機能数よりも以下を優先する。

1. 起動速度
2. メモリ使用量
3. 入力応答性
4. 大規模ファイルへの耐性
5. 実装の単純性
6. 安定性

---

## 2.2 プロセス分離

エディタ本体は可能な限り小さくする。

```text
editor.exe
    |
    +-- compiler.exe
    +-- test_runner.exe
    +-- submitter.exe
    +-- library_manager.exe
    +-- formatter.exe
```

外部ツールは必要時のみ起動する。

---

## 2.3 editor.exe に持たせない責務

以下は原則として外部プロセスまたは外部コマンドに委譲する。

- コンパイル
- 実行
- テストケース実行
- フォーマット
- ライブラリ展開
- オンラインジャッジへの提出

---

# 3. 対象環境

## 3.1 OS

初期版：

- Windows 10
- Windows 11

---

## 3.2 コンパイラ

開発時：

- MSVC

実行対象：

- GCC / MinGW
- Clang

競技プログラミング用途では g++ を第一優先とする。

---

# 4. 使用技術

## 4.1 言語

C++20 以上。

---

## 4.2 GUI

初期実装：

- Win32 API
- DirectWrite
- Direct2D

---

## 4.3 ビルド

CMake を使用する。

---

## 4.4 外部依存

初期版では可能な限り第三者ライブラリを減らす。

原則：

```text
標準ライブラリ
+
Windows API
+
DirectWrite
+
Direct2D
```

で実装する。

第三者ライブラリを導入する場合は、

- メモリ使用量
- ライセンス
- ビルド難易度
- 保守コスト

を評価する。

---

# 5. ソースコード構成

```text
LightCPEditor/
│
├── CMakeLists.txt
│
├── src/
│   ├── main.cpp
│   │
│   ├── app/
│   │   ├── Application.h
│   │   └── Application.cpp
│   │
│   ├── platform/
│   │   ├── Win32Window.h
│   │   ├── Win32Window.cpp
│   │   ├── Win32Input.h
│   │   ├── Win32Input.cpp
│   │   └── Process.h
│   │   └── Process.cpp
│   │
│   ├── editor/
│   │   ├── Editor.h
│   │   ├── Editor.cpp
│   │   ├── Cursor.h
│   │   ├── Cursor.cpp
│   │   ├── Selection.h
│   │   ├── Selection.cpp
│   │   ├── UndoManager.h
│   │   └── UndoManager.cpp
│   │
│   ├── document/
│   │   ├── Document.h
│   │   └── Document.cpp
│   │
│   ├── buffer/
│   │   ├── TextBuffer.h
│   │   ├── PieceTable.h
│   │   ├── PieceTable.cpp
│   │   ├── LineIndex.h
│   │   └── LineIndex.cpp
│   │
│   ├── renderer/
│   │   ├── Renderer.h
│   │   ├── Renderer.cpp
│   │   ├── FontManager.h
│   │   └── FontManager.cpp
│   │
│   ├── syntax/
│   │   ├── Lexer.h
│   │   ├── Lexer.cpp
│   │   ├── Token.h
│   │   └── TokenCache.h
│   │
│   ├── search/
│   │   ├── SearchEngine.h
│   │   └── SearchEngine.cpp
│   │
│   ├── completion/
│   │   ├── CompletionEngine.h
│   │   ├── CompletionEngine.cpp
│   │   └── SnippetManager.h
│   │
│   ├── project/
│   │   ├── Project.h
│   │   └── Project.cpp
│   │
│   ├── config/
│   │   ├── Config.h
│   │   └── Config.cpp
│   │
│   └── ui/
│       ├── TabBar.h
│       ├── TabBar.cpp
│       ├── FileTree.h
│       ├── FileTree.cpp
│       ├── StatusBar.h
│       └── StatusBar.cpp
│
├── tools/
│   ├── compiler/
│   ├── tester/
│   ├── library/
│   └── submitter/
│
├── tests/
│   ├── buffer/
│   ├── editor/
│   ├── syntax/
│   ├── search/
│   └── benchmark/
│
├── resources/
│   ├── fonts/
│   ├── snippets/
│   └── icons/
│
└── docs/
```

---

# 6. Application

## 6.1 責務

Application はアプリケーション全体のライフサイクルを管理する。

```cpp
class Application {
public:
    bool initialize();
    int run();
    void shutdown();

private:
    std::unique_ptr<class EditorWindow> window_;
};
```

---

# 7. Win32Window

## 7.1 責務

- ウィンドウ生成
- Windows message loop
- ウィンドウサイズ取得
- 再描画要求
- キーボードイベント
- マウスイベント

```cpp
class Win32Window {
public:
    bool create(
        const wchar_t* title,
        int width,
        int height
    );

    int runMessageLoop();

    HWND handle() const;

    void invalidate();
};
```

---

# 8. Editor

Editor はUIとDocumentの橋渡しを担当する。

## 8.1 責務

- 編集操作
- カーソル
- 選択
- キーバインド
- コピー・ペースト
- Undo/Redo
- スクロール
- 検索開始
- 保存

```cpp
class Editor {
public:
    void openDocument(std::shared_ptr<Document>);
    void save();
    void saveAs(const std::filesystem::path&);

    void insertText(std::string_view text);
    void deleteBackward();
    void deleteForward();

    void undo();
    void redo();

    void moveCursorLeft();
    void moveCursorRight();
    void moveCursorUp();
    void moveCursorDown();

    void pageUp();
    void pageDown();

    void selectAll();
    void copy();
    void cut();
    void paste();

    void render(Renderer& renderer);
};
```

---

# 9. Document

Document は「1つのファイル」を表す。

```cpp
class Document {
public:
    bool load(const std::filesystem::path& path);
    bool save();

    TextBuffer& buffer();
    const TextBuffer& buffer() const;

    const std::filesystem::path& path() const;

    bool modified() const;
    void markModified();
    void clearModified();

private:
    std::filesystem::path path_;
    TextBuffer buffer_;
    bool modified_ = false;
};
```

---

# 10. TextBuffer

TextBuffer はDocumentから独立させる。

## 10.1 基本API

```cpp
class TextBuffer {
public:
    size_t size() const;
    bool empty() const;

    char charAt(size_t offset) const;

    std::string getText(
        size_t offset,
        size_t length
    ) const;

    void insert(
        size_t offset,
        std::string_view text
    );

    void erase(
        size_t offset,
        size_t length
    );

    size_t lineCount() const;

    size_t lineStart(size_t line) const;

    size_t offsetFromLineColumn(
        size_t line,
        size_t column
    ) const;

    std::pair<size_t, size_t>
    lineColumnFromOffset(size_t offset) const;
};
```

---

# 11. Piece Table

## 11.1 バッファ

2種類のバッファを持つ。

```text
Original Buffer
Add Buffer
```

Original Buffer：

読み込んだファイルを保持する。

Add Buffer：

編集中に追加された文字列を保持する。

---

## 11.2 Piece

```cpp
enum class PieceSource {
    Original,
    Add
};

struct Piece {
    PieceSource source;
    uint64_t offset;
    uint64_t length;
};
```

---

## 11.3 Piece管理

Pieceの集合は編集が高速になるコンテナを使用する。

候補：

- B-tree
- Rope
- implicit treap

初期実装では実装容易性を優先し、簡潔な平衡木ベースの構造を採用する。

実装後、ベンチマークによって変更可能とする。

---

# 12. LineIndex

改行位置を管理する。

必要な操作：

```text
offset -> line
line -> offset
```

例えば、

```cpp
size_t lineFromOffset(size_t offset) const;
size_t lineStart(size_t line) const;
```

を提供する。

---

## 12.1 更新

文字列の挿入・削除時は影響範囲のみ更新する。

ファイル全体を毎回走査してはならない。

---

# 13. Cursor

```cpp
struct Cursor {
    size_t offset = 0;
    size_t preferredColumn = 0;
};
```

`preferredColumn` は上下移動時に使用する。

例：

```text
    abcdef
      ^
      3
    xyz
```

上下に移動して行長が違っても、可能な限り元の列を維持する。

---

# 14. Selection

```cpp
struct Selection {
    size_t anchor = 0;
    size_t active = 0;

    bool empty() const {
        return anchor == active;
    }

    size_t start() const {
        return std::min(anchor, active);
    }

    size_t end() const {
        return std::max(anchor, active);
    }
};
```

---

# 15. UndoManager

## 15.1 設計方針

ファイル全体を保存しない。

操作単位で管理する。

---

## 15.2 EditCommand

```cpp
enum class EditType {
    Insert,
    Delete,
    Replace
};

struct EditCommand {
    EditType type;
    size_t position;
    std::string oldText;
    std::string newText;
};
```

---

## 15.3 グループ化

文字を1文字ずつ入力したとき、

```text
h
e
l
l
o
```

を5個の独立履歴にするのではなく、状況に応じて

```text
"hello"
```

としてグループ化する。

---

## 15.4 メモリ制限

Undo履歴に上限を設定する。

初期目標：

```text
32MB
```

を超えた場合、古い履歴から破棄する。

---

# 16. Renderer

## 16.1 責務

- 背景
- 行番号
- テキスト
- カーソル
- 選択
- ハイライト
- スクロールバー
- UI

を描画する。

---

# 17. DirectWrite

フォント描画はDirectWriteを使用する。

基本的な流れ：

```text
Text Buffer
    ↓
visible text
    ↓
DirectWrite TextLayout
    ↓
Direct2D
```

---

# 18. 描画範囲

描画対象は画面に表示される範囲のみ。

```cpp
struct VisibleRange {
    size_t firstLine;
    size_t lastLine;
};
```

---

# 19. スクロール

エディタは、

```text
firstVisibleLine
horizontalOffset
```

を保持する。

1行の高さは固定値として管理する。

例：

```text
fontSize = 14
lineHeight = 20
```

初期版では可変行高をサポートしない。

---

# 20. テキスト描画

各可視行について、

```text
line number
syntax tokens
text
```

を描画する。

100万行のファイルでも、画面に30行しか見えていなければ原則30行程度のみ描画する。

---

# 21. FontManager

フォント情報を管理する。

```cpp
class FontManager {
public:
    bool initialize();
    IDWriteTextFormat* textFormat() const;

    float lineHeight() const;
    float charWidth() const;
};
```

等幅フォントを基本とする。

初期フォント候補：

- Consolas
- Cascadia Mono

---

# 22. Syntax Lexer

## 22.1 方針

コンパイラ完全互換の解析はしない。

表示用トークン化のみ行う。

---

## 22.2 Token

```cpp
enum class TokenKind {
    Normal,
    Keyword,
    Type,
    Number,
    String,
    Character,
    Comment,
    Preprocessor,
    Function
};

struct Token {
    size_t start;
    size_t length;
    TokenKind kind;
};
```

---

# 23. C++ Lexer

初期対応対象：

```text
keyword
identifier
number
string
character
comment
preprocessor
operator
```

---

## 23.1 コメント

対応：

```cpp
// comment
```

および

```cpp
/*
 comment
*/
```

---

# 24. Syntax Cache

行単位でキャッシュする。

```cpp
struct SyntaxLineCache {
    uint64_t version;
    std::vector<Token> tokens;
};
```

Documentに変更が発生した場合、その行のバージョンを変更する。

---

# 25. 検索

SearchEngineを独立させる。

```cpp
struct SearchQuery {
    std::string pattern;
    bool caseSensitive = false;
    bool wholeWord = false;
    bool regex = false;
};
```

初期版では通常検索を優先する。

---

## 25.1 API

```cpp
std::optional<size_t> findNext(
    const TextBuffer& buffer,
    size_t start,
    const SearchQuery& query
);
```

---

# 26. CompletionEngine

## 26.1 方針

Language Serverを常駐させない。

---

## 26.2 補完候補

候補源：

```text
keyword dictionary
standard library dictionary
contest snippets
document identifiers
```

---

## 26.3 CompletionItem

```cpp
struct CompletionItem {
    std::string label;
    std::string insertText;
};
```

---

# 27. SnippetManager

スニペットは外部ファイルで定義する。

例：

```text
resources/snippets/cpp.txt
```

形式は単純な独自形式とする。

例：

```text
rep => for (int i = 0; i < n; ++i) {\n}
```

---

# 28. Project

Projectは作業ディレクトリを表す。

```cpp
class Project {
public:
    bool open(const std::filesystem::path& root);

    const std::filesystem::path& root() const;

    std::vector<std::filesystem::path>
    sourceFiles() const;
};
```

---

# 29. Tab

```cpp
struct Tab {
    std::shared_ptr<Document> document;
    bool active;
};
```

複数Documentを開く。

---

# 30. ファイル読み込み

基本方針：

1. ファイルサイズ確認
2. バイナリとして読み込み
3. UTF-8として扱う
4. 改行コードを認識
5. Original Bufferへ格納

対応：

```text
LF
CRLF
```

---

# 31. 改行コード

内部表現ではLFを基本とする。

保存時にDocumentの設定に応じて、

```text
LF
CRLF
```

へ変換する。

初期実装ではファイルの元の改行コードを保存時のデフォルトとする。

---

# 32. UTF-8

内部文字列はUTF-8を基本とする。

ASCII範囲では1 byte = 1 columnとして高速処理する。

Unicodeについては初期版では、

- UTF-8保存
- UTF-8表示

を実装対象とし、複雑な文字幅処理は後段階で対応する。

---

# 33. クリップボード

Windows Clipboard APIを使用する。

```text
Ctrl+C
Ctrl+X
Ctrl+V
```

をサポートする。

---

# 34. キー入力

キーイベントはEditorへ変換する。

```cpp
enum class EditorCommand {
    InsertChar,
    Backspace,
    Delete,
    CursorLeft,
    CursorRight,
    CursorUp,
    CursorDown,
    Undo,
    Redo,
    Save,
    Open,
    Find,
    Copy,
    Cut,
    Paste
};
```

Win32のキーコードとEditorCommandを分離する。

---

# 35. キーバインド

将来的な変更を考慮し、

```text
キー入力
 ↓
KeyBinding
 ↓
EditorCommand
```

とする。

例：

```text
Ctrl+S -> Save
Ctrl+Z -> Undo
Ctrl+Shift+Z -> Redo
Ctrl+F -> Find
```

---

# 36. メインイベントループ

基本構造：

```text
Windows Message
      ↓
Input/Event Translation
      ↓
Editor Command
      ↓
Document変更
      ↓
Dirty Flag
      ↓
InvalidateRect
      ↓
Render
```

---

# 37. 再描画

変更がない場合は再描画しない。

再描画原因：

- 文字入力
- カーソル移動
- スクロール
- ウィンドウサイズ変更
- 選択変更
- Syntax Cache更新

---

# 38. カーソル点滅

タイマーを使用する。

ただし高頻度ポーリングは禁止する。

例：

```text
500ms
```

ごとにカーソル表示を切り替える。

---

# 39. CPU省電力

アプリがアイドル状態の場合、

```text
Wait
```

する。

常時、

```cpp
while (...) {
    render();
}
```

を実行してはならない。

---

# 40. 外部プロセス

Processクラスを実装する。

```cpp
class Process {
public:
    bool start(
        const std::wstring& executable,
        const std::vector<std::wstring>& args
    );

    void terminate();

    int exitCode() const;

    std::string stdoutText() const;
    std::string stderrText() const;
};
```

---

# 41. コンパイル

コンパイルは外部プロセス。

```text
editor
  ↓
Process
  ↓
g++.exe
```

コンパイル結果：

```cpp
struct ProcessResult {
    int exitCode;
    std::string stdoutText;
    std::string stderrText;
    uint64_t elapsedMs;
};
```

---

# 42. Test Runner

テストケースディレクトリ：

```text
tests/
    01.in
    02.in
    03.in
```

を読み込む。

各ケースについて：

```text
compile
 ↓
run
 ↓
input
 ↓
capture output
 ↓
compare
```

---

# 43. テスト結果

```cpp
enum class TestStatus {
    AC,
    WA,
    RE,
    TLE,
    CE
};

struct TestResult {
    TestStatus status;
    uint64_t elapsedMs;
    std::string output;
    std::string error;
};
```

---

# 44. Library Manager

Editorとは独立したツールとして作る。

```text
library_manager.exe
```

責務：

- ライブラリ登録
- 依存関係解析
- コード展開
- 提出用コード生成

---

# 45. Library形式

例：

```text
libraries/
├── dsu.hpp
├── fenwick.hpp
├── segtree.hpp
└── modint.hpp
```

---

# 46. ライブラリ依存関係

例えば、

```text
segtree
 └── utility
```

などを表現できるようにする。

初期版では専用の依存関係記述ファイルを用意する。

---

# 47. 提出処理

```text
Editor
 ↓
Submit
 ↓
Library Manager
 ↓
submission.cpp
 ↓
Submitter
 ↓
Online Judge
```

Submitterは別プロセスとする。

---

# 48. 設定ファイル

プロジェクト設定：

```text
.cpeditor/config.txt
```

ユーザー設定：

```text
%APPDATA%/LightCPEditor/config.txt
```

---

# 49. Config

```cpp
struct EditorConfig {
    std::string fontName;
    int fontSize;
    int tabWidth;

    bool insertSpaces;

    std::string compilerPath;
    std::string compileFlags;

    std::string theme;
};
```

---

# 50. UI構成

```text
+------------------------------------------------------+
| TabBar                                               |
+----------------+-------------------------------------+
|                |                                     |
| FileTree       | Editor                              |
|                |                                     |
|                |                                     |
|                |                                     |
|                |                                     |
+----------------+-------------------------------------+
| StatusBar                                            |
+------------------------------------------------------+
```

---

# 51. FileTree

初期版では、

- ディレクトリ
- ファイル

だけを表示する。

Gitステータスなどは後から追加する。

---

# 52. StatusBar

表示：

```text
Ln 120, Col 5
UTF-8
CRLF
C++
Spaces: 4
```

など。

コンパイル結果も表示可能。

---

# 53. エラーパネル

下部に結果表示領域を設ける。

```text
+------------------------------------------------------+
| Build Output                                         |
| main.cpp:13: error: ...                              |
|                                                      |
+------------------------------------------------------+
```

初期版では単純なテキストログとする。

---

# 54. 大規模ファイル対策

## 54.1 目標

100万行級のファイルでも、

- 開く
- スクロール
- 編集
- 保存

を可能にする。

---

## 54.2 禁止事項

以下を避ける。

```text
全文のstd::stringコピー
全行をstd::stringとして保持
全行のTextLayoutを保持
全文のASTを保持
無制限Undo
無制限キャッシュ
```

---

# 55. メモリ予算

初期目標：

```text
Application        < 5 MB
UI                 < 5 MB
Text Buffer       < 15 MB
Undo              < 10 MB
Syntax Cache       < 5 MB
Other              < 10 MB
----------------------------
Total              < 50 MB
```

これは初期設計目標であり、実測値に応じて調整する。

---

# 56. キャッシュの上限

すべてのキャッシュに上限を設ける。

例：

```text
Syntax Cache      8 MB
Completion Cache  2 MB
Search Cache      2 MB
```

超過した場合、LRU等で破棄する。

---

# 57. スレッド

初期版では最小限とする。

メインスレッド：

```text
UI
Editor
Renderer
```

ワーカースレッド候補：

```text
Syntax analysis
Search
Background file scan
```

ただし、必要性が確認されるまでは増やさない。

---

# 58. スレッド安全性

TextBufferは原則としてメインスレッドから変更する。

バックグラウンド処理はスナップショットまたは読み取り専用データを使用する。

原則：

```text
Main Thread
    |
    +-- mutate document

Worker
    |
    +-- analyze snapshot
```

---

# 59. ファイル監視

初期版では必須ではない。

将来的に外部変更検出を実装する場合は、

```text
ReadDirectoryChangesW
```

などのWindows APIを利用する。

---

# 60. 自動保存

初期版では任意機能。

自動保存を実装する場合：

```text
変更
 ↓
一定時間待機
 ↓
未保存
 ↓
autosave
```

常時保存は禁止する。

---

# 61. エラー処理

ファイル読み込み失敗：

```text
Error dialog
```

コンパイル失敗：

```text
Build Output
```

外部プロセス起動失敗：

```text
Process Error
```

とする。

---

# 62. クラッシュ対策

外部プロセスはeditor.exeと完全分離する。

Editor内部については、

- nullptr
- 範囲外アクセス
- 無効なPiece
- 不正なCursor
- 不正なSelection

をアサーションとテストで検出する。

---

# 63. ロギング

リリース版ではログ量を最小化する。

開発版：

```text
DEBUG
INFO
WARN
ERROR
```

を使用可能にする。

---

# 64. テスト方針

単体テストを中心とする。

特に以下を重点的にテストする。

```text
TextBuffer
PieceTable
LineIndex
UndoManager
Lexer
SearchEngine
Process
```

---

# 65. PieceTableテスト

例：

```text
初期
abc

insert(1, "XYZ")
aXYZbc

erase(1, 3)
abc
```

---

# 66. LineIndexテスト

例：

```text
abc
def
ghi
```

について、

```text
line 0 -> offset 0
line 1 -> offset 4
line 2 -> offset 8
```

などを検証する。

---

# 67. Undoテスト

```text
abc
insert XYZ
aXYZbc
undo
abc
redo
aXYZbc
```

---

# 68. Lexerテスト

以下を入力し、期待Tokenを比較する。

```cpp
int main() {
    // test
    return 0;
}
```

---

# 69. 性能テスト

以下のサイズのファイルを生成する。

```text
10,000 lines
100,000 lines
1,000,000 lines
```

測定項目：

- 起動時間
- 開く時間
- 初回描画時間
- 入力時間
- スクロール時間
- 保存時間
- メモリ使用量

---

# 70. ベンチマークツール

専用ベンチマークを作成する。

```text
tests/benchmark/
```

例：

```text
benchmark_buffer
benchmark_lexer
benchmark_search
benchmark_startup
benchmark_large_file
```

---

# 71. Startup Benchmark

計測区間：

```text
Process Start
      ↓
Window Created
      ↓
First Frame
```

目標：

```text
< 500 ms
```

---

# 72. 入力性能

1文字入力から表示更新までの遅延を測定する。

目標：

```text
通常操作 < 50 ms
```

---

# 73. Idle CPU

10秒間何も操作せず、

```text
CPU usage
```

を測定する。

目標：

```text
平均 1% 以下
```

---

# 74. 実装フェーズ

## Phase 1：Window

実装：

```text
Win32Window
Application
Renderer
```

完成条件：

- ウィンドウ表示
- 再描画
- キー入力取得

---

## Phase 2：TextBuffer

実装：

```text
PieceTable
LineIndex
Document
```

完成条件：

- Open
- Insert
- Delete
- Save

---

## Phase 3：基本Editor

実装：

```text
Cursor
Selection
UndoManager
```

完成条件：

- 文字入力
- Backspace
- Delete
- Copy/Paste
- Undo/Redo

---

## Phase 4：描画

実装：

```text
DirectWrite
Line Number
Cursor
Selection
Scrolling
```

完成条件：

- 実用的にコードを書ける

---

## Phase 5：基本機能

実装：

```text
Open
Save
Save As
Tab
Find
```

---

## Phase 6：C++対応

実装：

```text
Lexer
Syntax Highlight
Completion
Snippets
```

---

## Phase 7：競プロ機能

実装：

```text
Compile
Run
Test Runner
```

---

## Phase 8：Library Manager

実装：

```text
Dependency resolution
Expansion
Submission generation
```

---

## Phase 9：Submitter

OJ連携を追加する。

---

## Phase 10：最適化

測定結果に基づいて、

```text
Memory
Startup
Input Latency
Large File Performance
```

を最適化する。

---

# 75. MVP完成条件

以下を満たした時点でMVPとする。

```text
[ ] Windowsで起動する
[ ] ファイルを開ける
[ ] ファイルを保存できる
[ ] 文字入力ができる
[ ] Delete/Backspaceが使える
[ ] Cursor移動ができる
[ ] 範囲選択ができる
[ ] Copy/Pasteができる
[ ] Undo/Redoが使える
[ ] 行番号を表示できる
[ ] スクロールできる
[ ] C++の基本的な構文ハイライトがある
[ ] g++でコンパイルできる
[ ] プログラムを実行できる
[ ] テストケースを実行できる
```

---

# 76. 実装上の禁止事項

以下は特別な理由がない限り行わない。

### 禁止1

エディタ本体から外部ツールを同期実行してUIを長時間停止させる。

### 禁止2

ファイル全文のコピーを編集操作のたびに生成する。

### 禁止3

無制限のキャッシュ。

### 禁止4

無制限Undo履歴。

### 禁止5

不要なバックグラウンドスレッド。

### 禁止6

毎フレームの全画面再計算。

### 禁止7

アイドル状態での常時ポーリング。

---

# 77. コーディング規約

- C++20
- RAII
- `std::unique_ptr` を所有権管理に使用
- 所有権がないポインタは raw pointer でも可
- 所有オブジェクトの `new/delete` を原則直接使用しない
- const correctnessを徹底
- 符号付き/符号なしの混在に注意
- 大容量データはコピーを避ける
- `std::string_view` を積極的に使用
- APIは責務ごとに分離する

---

# 78. API設計方針

モジュール間の依存関係は一方向にする。

```text
UI
 ↓
Editor
 ↓
Document
 ↓
TextBuffer
```

RendererはDocumentを直接変更しない。

```text
Renderer
 ↓
read only TextBuffer
```

ProcessはEditorの内部状態を直接操作しない。

```text
Editor
 ↓
Process API
 ↓
Process Result
```

---

# 79. 依存関係

理想形：

```text
Application
   ↓
EditorWindow
   ↓
Editor
   ↓
Document
   ↓
TextBuffer
```

Renderer：

```text
Renderer
   ↓
read-only Editor/Document state
```

Process：

```text
Process
   ↑
Compiler/Test/Submit
```

---

# 80. 将来の拡張に対するルール

新しい機能を追加するときは、

1. editor.exeに常駐させる必要があるか
2. 外部プロセスで代替できるか
3. メモリ使用量はいくら増えるか
4. CPU使用量はいくら増えるか
5. 起動速度に影響するか
6. 大規模ファイルに悪影響がないか

を確認する。

---

# 81. 将来追加候補

MVP完成後に検討する。

```text
LSP
Git
Debugger
Regex Search
Multiple Cursor
Mini Map
Markdown
AtCoder連携
Codeforces連携
Contest Timer
Problem Statement View
Automatic Sample Test
```

ただし、これらは軽量性を壊さない範囲で導入する。

---

# 82. 最終アーキテクチャ

```text
                    ┌───────────────────────┐
                    │      editor.exe       │
                    │                       │
                    │ Application           │
                    │      │                │
                    │      ▼                │
                    │     Editor            │
                    │      │                │
                    │      ▼                │
                    │   Document            │
                    │      │                │
                    │      ▼                │
                    │   TextBuffer          │
                    │      │                │
                    │   PieceTable          │
                    │                       │
                    │ Renderer              │
                    │ Syntax                │
                    │ Search                │
                    │ Completion            │
                    └───────────┬───────────┘
                                │
             ┌──────────────────┼─────────────────┐
             │                  │                 │
             ▼                  ▼                 ▼
       compiler.exe       test_runner.exe   library_manager.exe
             │                                    │
             ▼                                    ▼
            g++                              submission.cpp
                                                  │
                                                  ▼
                                            submitter.exe
```

---

# 83. 開発時の優先順位

優先順位は次の通りとする。

```text
1. 正しく編集できる
2. 編集が高速
3. 安定している
4. メモリが少ない
5. 起動が速い
6. 競プロ機能
7. 追加機能
```

ただし、設計段階から軽量性を損なう構造は採用しない。

---

# 84. 完成イメージ

最終的には、

```text
┌──────────────────────────────────────────────────────┐
│ main.cpp │ test.cpp │                               │
├──────────┬───────────────────────────────────────────┤
│          │  1 #include <bits/stdc++.h>               │
│ Files    │  2 using namespace std;                    │
│          │  3                                         │
│ main.cpp │  4 int main() {                           │
│ test.cpp │  5     int n;                             │
│          │  6     cin >> n;                          │
│ tests/   │  7 }                                       │
│          │                                             │
├──────────┴───────────────────────────────────────────┤
│ Build: OK | Test: 10/10 | Ln 7, Col 2 | C++23       │
└──────────────────────────────────────────────────────┘
```

という、競技プログラミングに必要な操作へ最短で到達できるUIを目指す。

---

# 85. 実装完了の判断

本プロジェクトは、機能数が多いことではなく、

```text
「競プロを快適に行うための機能が揃っている」
+
「非常に軽い」
+
「非常に速い」
```

ことを完成条件とする。

したがって、不要な機能追加よりも、

```text
メモリ削減
起動高速化
入力遅延削減
大規模ファイル対応
安定性向上
```

を優先する。