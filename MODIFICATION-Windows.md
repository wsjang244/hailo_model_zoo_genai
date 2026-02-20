# hailo_model_zoo_genai Windows移植ガイド

## 概要

[hailo_model_zoo_genai](https://github.com/hailo-ai/hailo_model_zoo_genai) はLinux専用プロジェクトだが、HailoRT 5.2.0がWindowsに対応しているため、ソースコードの修正によりWindows上でのビルド・実行が可能。

---

## 前提条件

- Windows 10/11 64bit
- Visual Studio 2022 Build Tools（MSVC 19.41）
- CMake 4.2+
- HailoRT 5.2.0（`C:\Program Files\HailoRT` にインストール済み）
- Hailo-10H モジュール（PCIe接続）

---

## 1. OpenSSLの準備（vcpkg経由）

```powershell
cd C:\
git clone https://github.com/microsoft/vcpkg
cd .\vcpkg
.\bootstrap-vcpkg.bat
.\vcpkg.exe install openssl:x64-windows

# vcpkg の DLL にパスを通す（OpenSSL等の実行時に必要）
$env:PATH += ";C:\vcpkg\installed\x64-windows\bin"
```

---

## 2. ソースコードの編集

### 2-1. `CMakeLists.txt`（トップレベル）

元の内容にWindows対応の定義を追加：

```cmake
cmake_minimum_required(VERSION 3.20)
option(HAILO_BUILD_UT "Build Unit Tests" OFF)
project(hailo-ollama)
include(FetchContent)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_CXX_STANDARD 20)
find_package(OpenSSL REQUIRED)
find_package(HailoRT 5 REQUIRED)
add_subdirectory(src)
add_subdirectory(thirdparty)
if(HAILO_BUILD_UT)
    add_subdirectory(test)
    enable_testing()
    add_test(project-tests hailo-ollama-test)
endif()
if (WIN32)
  target_compile_definitions(hailo-ollama-lib PUBLIC NOMINMAX WIN32_LEAN_AND_MEAN)
endif()
```

### 2-2. `src/library/controller/controller.cpp`（271行目付近）

oatpp `Int64` への `long` 代入が曖昧（MSVCでは `long` = 32bit）：

```cpp
// 変更前
choice->index = 0L;  // long型
// 変更後
choice->index = static_cast<v_int64>(0L);
```

### 2-3. `src/library/generation_context/generation_context.cpp`（125行目付近）

`std::filesystem::path` → `const std::string &` の暗黙変換がMSVCで不可：

```cpp
// 変更前
llm_params.set_model(m_last_path, ""s);

// 変更後
llm_params.set_model(m_last_path.string(), ""s);
```

### 2-4. `src/library/model/blob_resource.cpp`（38行目付近）

同じく `filesystem::path` → `string` の変換：

```cpp
// 変更前
return m_blob_dir / ("sha256_"s + resource);

// 変更後
return (m_blob_dir / ("sha256_"s + resource)).string();
```

### 2-5. `src/library/utils/time.cpp`（31行目付近）

`gmtime_r` はPOSIX関数でWindows非対応。`gmtime_s` で代替（引数の順序が逆）：

```cpp
// 変更前
struct tm result;
stream << std::put_time(gmtime_r(&epoch_seconds, &buf), "%FT%T"); // POSIX: gmtime_r(time_t*, tm*)

// 変更後
struct tm result;
#ifdef _WIN32
gmtime_s(&buf, &epoch_seconds);
stream << std::put_time(&buf, "%FT%T"); // Windows: gmtime_s(tm*, time_t*)
#else
stream << std::put_time(gmtime_r(&epoch_seconds, &buf), "%FT%T"); // POSIX: gmtime_r(time_t*, tm*)
#endif
```

### 2-6. `src/apps/server/CMakeLists.txt`

`hailo-ollama-lib` をリンクする側（サーバーexe）にHailoRTのインクルードパスを通す。

```cpp
// 変更前
struct tm result;
target_link_libraries(hailo-ollama hailo-ollama-lib)

// 変更後
target_link_libraries(hailo-ollama hailo-ollama-lib HailoRT::libhailort)
```

---

## 3. ビルド手順

```powershell
cd C:\Workspace\Windows_demo\hailo_model_zoo_genai
mkdir build
cd build

# CMake生成
cmake -G "Visual Studio 17 2022" -A x64 `
  -DCMAKE_TOOLCHAIN_FILE=C:\vcpkg\scripts\buildsystems\vcpkg.cmake `
  -DVCPKG_TARGET_TRIPLET=x64-windows `
  -DOATPP_BUILD_TESTS=OFF `
  -DHAILO_BUILD_UT=OFF `
  ..

# ビルド
cmake --build . --config Release
```

成果物: `build\src\apps\server\Release\hailo-ollama.exe`

---

## 4. 設定ファイルの配置

```powershell
# 設定ファイル
mkdir "$env:USERPROFILE\.config\hailo-ollama"
Copy-Item ..\config\hailo-ollama.json "$env:USERPROFILE\.config\hailo-ollama\"

# モデルデータ
mkdir "$env:USERPROFILE\.local\share\hailo-ollama"
Copy-Item -Recurse ..\models\* "$env:USERPROFILE\.local\share\hailo-ollama\models\"
```

配置確認：

```powershell
Test-Path "$env:USERPROFILE\.config\hailo-ollama\hailo-ollama.json"   # True
Test-Path "$env:USERPROFILE\.local\share\hailo-ollama\models\manifests" # True
```

---

## 5. サーバー起動

```powershell
# vcpkg の DLL にパスを通す（OpenSSL等の実行時に必要）
$env:PATH += ";C:\vcpkg\installed\x64-windows\bin"

# HOME環境変数の設定（path.cppがXDG規約ベースのため必須）
$env:HOME = $env:USERPROFILE

# サーバー起動
.\src\apps\server\Release\hailo-ollama.exe
```

---

## 6. 基本操作（PowerShell）

### モデル一覧

```powershell
curl.exe --silent http://localhost:8000/hailo/v1/list
```

### Invoke-RestMethod での代替

```powershell
Invoke-RestMethod http://localhost:8000/hailo/v1/list
```

### モデルのダウンロード（Pull）

PowerShellではInvoke-RestMethodが安定しています：

```powershell
Set-Content -Path body.json -Value '{ "model": "qwen2.5-coder:1.5b", "stream": true }' -Encoding UTF8
curl.exe --silent http://localhost:8000/api/pull -H "Content-Type: application/json" -d "@body.json"
Invoke-RestMethod http://localhost:8000/api/pull -Method Post -ContentType "application/json" -Body '{ "model": "qwen2.5-coder:1.5b", "stream": true }'
```

### チャット

```powershell
Invoke-RestMethod http://localhost:8000/api/pull -Method Post -ContentType "application/json" -Body '{"model": "qwen2.5-coder:1.5b", "messages": [{"role": "user", "content": "Tell me a joke"}]}'
```


---

## 7. Open WebUI（オプション）

### インストール

```powershell
py.exe -m pip install open-webui
```

### 起動

```powershell
$env:OLLAMA_BASE_URL = "http://127.0.0.1:8000"
$env:DATA_DIR = "$env:USERPROFILE\.open-webui"
open-webui serve
```

ブラウザで `http://localhost:8080` にアクセス。初回はローカルアカウントを作成。

---

## 補足：再ビルドの手順

| 変更内容 | 必要な操作 |
|---|---|
| ソースファイル（.cpp/.hpp）のみ変更 | `cmake --build . --config Release` のみ |
| CMakeLists.txt を変更 | cmake 再生成 → ビルド（2ステップ） |
| buildフォルダを消して最初から | 依存ライブラリ変更時やおかしくなった時のみ |
