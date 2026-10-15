---
title: "Windowsでgit worktreeを切るたびのフルビルドを短くした（ccache + MSVC）"
emoji: "🧱"
type: "tech"
topics: ["windows", "flutter", "ccache", "cmake", "msvc"]
published: true
publication_name: "bestat"
---

Claude Code や Codex に作業させるとき、タスクごとに git worktree を切っている。並列で何本も走らせられて便利なんだけど、worktree を切るたびに Windows のビルドがまっさらからになって、1本目のビルドに12分くらい待たされていた。

iOS だと Xcode の Compilation Cache を worktree 間で共有する方法があるらしい。

https://zenn.dev/sryu/articles/worktree-compilation-cache

Windows（MSVC）で同じことができないか試して、ついでにビルド構成の無駄も削った。

![](/images/windows-worktree-build-cache/build-time.png)
*新しく切った worktree で `flutter build windows --profile` を3回（cold、変更なし、Dart を1行変更）流したときの時間*

対象は社内で開発中の3Dのソフトウェアで、画面は Flutter、描画は C++ と Vulkan、データ処理の一部は Rust で書いている。3Dを扱う構成なので Debug ビルドだとパフォーマンスが出なくて、いちいち Profile ビルドして動作確認していたのもストレスだった。Windows 版は Flutter の runner から C++ の DLL を読み込む構成で、ビルドは CMake の Visual Studio generator（MSBuild）。

## 環境

- Windows 11、8論理コア、メモリ32GB
- Visual Studio 2022（MSVC 14.44）、CMake 3.31（VS 同梱）
- Flutter 3.41.5、Rust 1.94.1
- ccache 4.14.1（winget で入れたもの）

## 何に時間がかかっていたか

cold build（12分弱）の内訳はこんな感じだった。

- C++ の native DLL が5〜7分。Draco や basisu などの依存も全部ソースからビルドしている
- Rust のライブラリを debug と release の両方でビルドしていて、並列で4分くらい CPU を取り合っていた（Profile ビルドなのに debug は使わない）
- Firebase の C++ SDK（約800MB の zip）を worktree ごとにダウンロードして、約7GB に展開していた

C++ をキャッシュする前に、そもそも要らない仕事をしていた。

## ビルド構成の見直し

ccache を入れなくても、これだけで cold は704秒から545秒、変更なしの再ビルドは31秒から21秒になった。

Rust は CMake の custom command で cargo を呼んでいて、debug と release の両方に依存していた。選んだ構成だけビルドするようにした。

```cmake
add_custom_command(
    OUTPUT "${TARGET_DIR}/$<IF:$<CONFIG:Debug>,debug,release>/core.lib"
    COMMAND ${CMAKE_COMMAND} -E env "CARGO_TARGET_DIR=${TARGET_DIR}"
            ${CARGO_EXECUTABLE} build --profile "$<IF:$<CONFIG:Debug>,dev,release>"
            --manifest-path "${CRATE_DIR}/Cargo.toml"
    ...)
```

Firebase の SDK は、firebase_core の Windows plugin が環境変数 `FIREBASE_CPP_SDK_DIR` を見てくれる。そこに展開済みの SDK があってバージョンが一致すればダウンロードしない。なので plugin の configure より前に、SDK のバージョンごとの共有ディレクトリへ一度だけ展開して環境変数をセットするようにした。

```cmake
# generated_plugins.cmake より前で
file(LOCK "${CACHE_ROOT}/${FIREBASE_SDK_VERSION}.lock" GUARD FUNCTION)
if(NOT EXISTS "${SDK_DIR}/include/firebase/version.h")
  # ダウンロードして staging に展開し、終わったら rename
endif()
set(ENV{FIREBASE_CPP_SDK_DIR} "${SDK_DIR}")
```

バージョンは plugin の CMakeLists.txt の `FIREBASE_SDK_VERSION` から読む。worktree を同時に configure することがあるので `file(LOCK)` で排他して、途中で落ちても壊れた SDK を読まないよう、展開が終わってから rename している。Flutter で Firebase を使っている Windows アプリなら、これだけで worktree ごとに7GB と1〜2分浮くので、一番手軽だと思う。

## ccache を MSVC で使う

Visual Studio generator は `CMAKE_CXX_COMPILER_LAUNCHER` を見てくれない。代わりに MSBuild の `CLToolExe` と `CLToolPath` で、呼ぶ cl.exe を差し替える。ccache.exe を cl.exe という名前でコピーしておくと、ccache は PATH 上の本物の cl.exe を探して呼んでくれる。

```cmake
# project() より前
cmake_policy(SET CMP0141 NEW)
set(CMAKE_POLICY_DEFAULT_CMP0141 NEW)
# /Zi の共有 PDB はキャッシュできないので /Z7 にする
set(CMAKE_MSVC_DEBUG_INFORMATION_FORMAT "$<$<CONFIG:Debug,RelWithDebInfo>:Embedded>")
list(APPEND CMAKE_VS_GLOBALS
    "CLToolExe=cl.exe"
    "CLToolPath=${PROXY_DIR}"    # ccache.exe を cl.exe としてコピーしたディレクトリ
    "UseMultiToolTask=true"      # 1回の cl 呼び出しに1ソースだけ渡す
    "TrackFileAccess=false")     # 後述
```

`UseMultiToolTask` がないと MSBuild は複数ソースを1回の cl.exe にまとめて渡すので、ccache がキャッシュできない。

worktree をまたいでヒットさせるのが今回の本題で、ccache の `base_dir` を使う。`base_dir` 配下の絶対パスは、コンパイル時の作業ディレクトリからの相対パスに書き換えられてからキーになる。worktree が `D:\work\a` と `D:\work\b` にあっても、`build\x\..\..\src\foo.cpp` のような同じ相対パスになるので、別の worktree の結果をそのまま使える。

```ini
# %LOCALAPPDATA%\ccache\ccache.conf
base_dir = C:\Users;D:\work
hash_dir = false
max_size = 30G
```

Windows では `;` 区切りで複数書ける。`hash_dir=false` は、デバッグ情報付きのコンパイルで作業ディレクトリをキーに入れないための設定。設定ファイルに書いておけば、MSBuild から起動される ccache も環境変数なしで読んでくれる。

sccache にも `SCCACHE_BASEDIRS` があるけど、こっちは前方一致で削るだけなので、`D:\work` を消しても `a\src\foo.cpp` と `b\src\foo.cpp` で別物になる。worktree ごとのルートを全部列挙する必要があって、しかも設定は server の起動時に読むっぽいので、worktree をどんどん増やす使い方には合わなかった。

## ハマったところ

### file tracking を残すと古い obj が残る

`TrackFileAccess=false` にすると MSBuild が依存関係を追わなくなるので、変更がなくても全ソースが ccache を通って、リンクも毎回走る。これが嫌で `true` のまま使えないか、小さな VS プロジェクトを2つ作って試した。

結果、キャッシュにヒットしたあとでヘッダーを書き換えても再コンパイルされず、古い obj のまま実行ファイルができた。キャッシュヒット時は本物の cl.exe が動かないので、MSBuild の tracker が依存を正しく記録できないっぽい。ccache の inode cache や統計の書き込みを切ると通る回もあったけど、実行ごとに結果が揺れたので諦めた。ccache を挟まなければ `UseMultiToolTask` と tracking の組み合わせは正常だった。

なので `false` 必須で、その代わり反復ビルドが30秒くらい遅くなる（グラフの右2つ）。worktree を切った直後は4分半速くなるけど、1つの worktree で9回以上ビルドするなら ccache なしのほうが速い計算になる。なのでチームの既定にはせず、使いたい人が入れる opt-in にした。

### 一部のライブラリだけ毎回ミスする

2本目の worktree でも、約4%（102件）が毎回キャッシュミスしていた。ccache のログとデバッグ出力を有効にすると、何をハッシュに入れたかが obj ごとにテキストで出る。

```powershell
$env:CCACHE_LOGFILE = "D:\tmp\ccache.log"
$env:CCACHE_DEBUG = "1"
$env:CCACHE_DEBUGDIR = "D:\tmp\ccache-debug"
```

ミスしたのは全部 Draco（glTF の圧縮で使っている）で、`*.ccache-input-text` を worktree 名で grep したらすぐ見つかった。

```text
DRACO_FLAGS_SRCDIR="D:/work/a/build/_deps/draco-src"
```

Draco の CMake が、全ターゲットに checkout の絶対パスを `-D` で埋め込んでいた。`base_dir` が書き換えるのは引数そのものか `=` の後ろの値だけで、引用符付きの値は書き換えない。値は Draco のテスト用でライブラリのソースからは参照されていなかったので、FetchContent のあとで取り除いた。

```cmake
FetchContent_MakeAvailable(draco)
get_property(targets DIRECTORY "${draco_SOURCE_DIR}" PROPERTY BUILDSYSTEM_TARGETS)
foreach(target IN LISTS targets)
    get_target_property(defs ${target} COMPILE_DEFINITIONS)
    if(defs)
        list(FILTER defs EXCLUDE REGEX "^DRACO_FLAGS_SRCDIR=")
        set_property(TARGET ${target} PROPERTY COMPILE_DEFINITIONS "${defs}")
    endif()
endforeach()
```

これで2本目の worktree は832件ヒット、ミス0件になった。自分のコードで絶対パスを `-D` に入れる場合も同じことになるので、そういう定義は1ファイルだけに閉じ込めるようにしている。

## Rust はやめた

Rust も共有したかったけど、2つ試してどっちも採用しなかった。

sccache を `RUSTC_WRAPPER` に入れて、target directory だけ違う2回のビルドを流したら、2回目もヒット0件だった。target directory のパスがキーに入るっぽい。

cargo の `build.build-dir`（中間生成物の置き場を target directory と分ける設定）を worktree 間で共有すると、依存クレートは再利用できた。ただ、2本目の worktree のビルドが0.3秒で終わって、1本目の worktree の自前クレートの成果物をそのまま使っていた。ソースが同じだったから結果は正しいけど、ブランチごとに Rust のコードが違うと古いライブラリをリンクしかねない。怖いのでやめた。

Rust の release ビルドは単体だと1分くらい。ただ ccache で C++ が速くなった今は、C++ と CPU を取り合いながらクリティカルパスに残っているので、次に削るならここかな。

## 起動したまま DLL を差し替える

冒頭に書いた Debug ビルドの重さは、Flutter の Debug ビルドだと native まで最適化なしの構成になるのが原因だった。Profile ビルドで確認していると、Dart を1行変えるたびに1分待つことになる。

そこで、Debug の runner でも native は Release 構成の DLL を使うように変えた。DLL は C の ABI で `LoadLibraryW` / `GetProcAddress` して使っていて、確保したメモリも DLL 側の関数で解放しているので、Debug の runner（`/MDd`）と Release の DLL を混ぜても問題ない。これで Dart は hot reload しつつ、native は最適化済みのもので動く。

残るのは native を変えたときで、アプリを終了して起動し直す必要がある。Android だと `.so` を別名で置いて読み直しながら開発するやり方があるので、同じことができないか試した。

ところが、この DLL は `FreeLibrary` できなかった。DLL 自身が `GetModuleHandleExW` の PIN フラグでプロセス終了まで固定されていて、止める API のないワーカースレッドも持っている。無理に外すとそのスレッドが落ちる。

そこで、別名で置いて読み直す（ダブルバッファ）考え方はそのままに、古い DLL は解放しないことにした。Windows はフルパスが違えば同じ名前の DLL も別のモジュールとして読めるので、新しい DLL を世代ごとのディレクトリにコピーして読み込む。

```text
runner\Debug\
  viewer_bridge.dll           起動時に読んだもの（解放しない）
  native-reload\<pid>-3\
    viewer_bridge.dll         hot restart で読んだもの
    shaders\*.spv             DLL の隣から読まれる shader も一緒に置く
```

きっかけは Flutter の hot restart にした。このアプリは Dart の起動処理で native のランタイム初期化を呼んでいて、hot restart すると起動処理が走り直すので、初期化がもう一度呼ばれる。そのタイミングで native のビルド出力と読み込み中の DLL を比べて、違っていれば差し替える。

```powershell
# flutter run を起動したまま、別のターミナルで
cmake --build build-win --config Release --target bridge
# flutter run の画面で R
```

比較は更新時刻ではなく中身でやっている。runner のビルドが DLL をコピーするときに使う `cmake -E copy_if_different` は更新時刻を保たないので、時刻で比べると最初の hot restart で必ず差し替わってしまう。

差し替え前に開いていた画面は古い DLL のまま動き続けて、開き直すと新しい DLL を使う作りにした。古い DLL はメモリに残り続けるので、長く使ったら起動し直す前提（Debug ビルドだけの機能にしている）。

ビルドの待ち時間が減って、Profile で確認する機会も減りそう。worktree をどんどん切る使い方だと、こういう地味な待ち時間が積み重なるとけっこう大きいなと思った。
