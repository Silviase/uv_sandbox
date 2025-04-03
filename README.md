# uv による swallow-eval 環境構築ガイド

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

---

_Appendix A: assisted by observer Y._
