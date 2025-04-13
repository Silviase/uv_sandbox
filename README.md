# uv による swallow-eval 環境構築ガイド

# Table of Contents

- [uv による swallow-eval 環境構築ガイド](#uv-による-swallow-eval-環境構築ガイド)
- [Table of Contents](#table-of-contents)
  - [CUDA バージョンの違いを使い分けるには](#cuda-バージョンの違いを使い分けるには)
    - [1. CUDA 別の設定ファイル一覧](#1-cuda-別の設定ファイル一覧)
      - [例：CUDA 12.4 環境での初期化](#例cuda-124-環境での初期化)
    - [2. `.uv/cuda124.toml` はどのように作られたか？](#2-uvcuda124toml-はどのように作られたか)
    - [確認：CUDA 12.4 環境（selene）](#確認cuda-124-環境selene)
    - [切り替え：CUDA 11.8 環境（musa）](#切り替えcuda-118-環境musa)
      - [手順（再構築方式）](#手順再構築方式)
  - [`isolated` オプションによる環境の使い分け](#isolated-オプションによる環境の使い分け)

## CUDA バージョンの違いを使い分けるには

`torch`や`torchvision`などの GPU 対応ライブラリは、CUDA バージョンごとに最適化されたパッケージが提供されている。`uv`では明示的な index 分離と依存グループにより、環境ごとの管理が可能。

本プロジェクトでは、以下のように **CUDA ごとの構成ファイルを `.uv/` に保存**し、必要なときに `pyproject.toml` としてコピーする運用を採用している。

### 1. CUDA 別の設定ファイル一覧

- `.uv/cuda118.toml`
- `.uv/cuda124.toml`

#### 例：CUDA 12.4 環境での初期化

```bash
cp .uv/cuda124.toml pyproject.toml
uv sync --group common --group gpu
```

### 2. `.uv/cuda124.toml` はどのように作られたか？

`.uv/cuda124.toml` は以下の手順で作成された：

1. `uv init` により、初期構成ファイル（`pyproject.toml`）を生成。
2. 生成された `[project]` セクションに必要な値（name, version, python バージョンなど）を入力。
3. `torch` と `torchvision` の取得元を指定するため、`[tool.uv.sources]` と `[[tool.uv.index]]` セクションを**手書きで追加**：

   ```toml
   [tool.uv.sources]
   torch = { index = "pytorch-cu124" }
   torchvision = { index = "pytorch-cu124" }

   [[tool.uv.index]]
   name = "pytorch-cu124"
   url = "https://download.pytorch.org/whl/cu124"
   ```

4. GPU 系パッケージを追加：

   ```bash
   uv add torch --group gpu
   ```

5. 共通パッケージを追加：

   ```bash
   uv add ruff --group common
   ```

これにより、`torch` は CUDA 12.4 対応インデックスからインストールされ、`ruff` は PyPI から取得される構成が完成する。

### 確認：CUDA 12.4 環境（selene）

まず、CUDA 12.4 環境である `selene` 上で `.uv/cuda124.toml` を `pyproject.toml` にコピーし、同期を行う：

```sh
cp .uv/cuda124.toml pyproject.toml
uv sync --group common --group gpu
```

その後、テストスクリプトを実行：

```sh
python hello.py
```

実行結果（要約）：

```sh
PyTorch version: 2.6.0+cu124
CUDA available: True
CUDA version: 12.4
cuDNN version: 90100
Device count: 3
[GPU 0] NVIDIA RTX 6000 Ada Generation
[GPU 1] NVIDIA RTX 6000 Ada Generation
[GPU 2] NVIDIA RTX 6000 Ada Generation
```

このように、CUDA 12.4 に対応した `torch` が正しくインストールされていることが確認できた。

---

### 切り替え：CUDA 11.8 環境（musa）

次に、CUDA 11.8 環境である `musa` にて `.uv/cuda118.toml` を使用する。

#### 手順（再構築方式）

1. 既存の依存環境を強制再インストール：

   ```sh
   uv sync --force-reinstall
   ```

2. `torch` を CUDA 11.8 対応版として再追加：

   ```sh
   uv add torch --group gpu
   ```

3. 依存を再同期：

   ```sh
   uv sync --group common --group gpu
   ```

4. テストスクリプトを実行：

   ```sh
   uv run python hello.py
   ```

実行結果（要約）：

```sh
PyTorch version: 2.6.0+cu118
CUDA available: True
CUDA version: 11.8
cuDNN version: 90100
Device count: 10
[GPU 0-9] NVIDIA GeForce RTX 2080 Ti
```

このように、環境に応じて適切な CUDA バージョンの `torch` を導入・切り替えできることが確認された。

---

## `isolated` オプションによる環境の使い分け

`uv run --isolated --extra torch python hello.py` とすると、extra `torch` が持つ依存パッケージのみがインストールされ、一時的な仮想環境が作成される。
したがって、以下の shell script の実行結果のように、`torch` をインストールした環境では GPU が認識されるが、`vis` をインストールした環境では GPU が認識されない。

```sh
maeda-k@selene:~/Project/uv_sandbox$ uv run --isolated --extra torch python hello.py
Installed 39 packages in 4.89s
PyTorch version: 2.6.0+cu124
CUDA available: True
CUDA version: 12.4
cuDNN version: 90100
Device count: 3
[GPU 0] NVIDIA RTX 6000 Ada Generation
[GPU 1] NVIDIA RTX 6000 Ada Generation
[GPU 2] NVIDIA RTX 6000 Ada Generation

maeda-k@selene:~/Project/uv_sandbox$ uv run --isolated --extra vis python hello.py
Installed 121 packages in 1.47s
Traceback (most recent call last):
  File "/home/maeda-k/Project/uv_sandbox/hello.py", line 1, in <module>
    import torch
ModuleNotFoundError: No module named 'torch'
```

---

_Appendix A: assisted by observer Y._
