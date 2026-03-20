---
model: haiku
created: 2026-03-20
modified: 2026-03-20
reviewed: 2026-03-20
name: mypy-type-checking
description: |
  mypy static type checker workflow for Python projects. Covers running mypy,
  configuring mypy in pyproject.toml, fixing common type errors without type: ignore,
  strict mode, type stubs, and integration with uv projects.
  Use this skill whenever the user mentions mypy, type annotations, type errors,
  type checking Python code, or wants to add type safety to their Python project.
  Triggered by: mypy, type checking, type errors, type annotations, type stubs,
  reveal_type, strict mypy, Any type, Optional, Union, TypeVar, Protocol, overload.
user-invocable: false
allowed-tools: Bash(mypy *), Bash(uv *), Bash(python *), Read, Edit, Write, Grep, Glob
---

# mypy Type Checking

mypy は Python の静的型チェッカー。型アノテーションを検証し、実行前にバグを発見する。

**重要制約**: `# type: ignore` は使用しない。型エラーは正しく修正する。

## インストールと実行

```bash
# uv プロジェクトへ開発依存として追加
uv add --dev mypy

# 基本実行
uv run mypy src/

# 特定ファイルのみ
uv run mypy src/auto_trading/core/logging.py

# ストリクトモードで実行
uv run mypy --strict src/

# 型情報を表示
uv run mypy --reveal-type src/module.py
```

## pyproject.toml 設定

```toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_configs = true
warn_redundant_casts = true
warn_unused_ignores = true
disallow_untyped_defs = true
disallow_any_generics = true
no_implicit_reexport = true

# サードパーティライブラリのスタブがない場合
[[tool.mypy.overrides]]
module = ["pandas.*", "numpy.*"]
ignore_missing_imports = true
```

### 段階的移行

```toml
# 既存コードベースを段階的に型付け
[tool.mypy]
python_version = "3.12"
# まず基本チェックのみ
disallow_untyped_defs = false
warn_return_any = false

# 新規モジュールは厳格に
[[tool.mypy.overrides]]
module = ["auto_trading.core.*", "auto_trading.domain.*"]
disallow_untyped_defs = true
strict = true
```

## よくある型エラーと正しい修正方法

### `None` の可能性がある値

```python
# NG: None を考慮していない
def get_user(user_id: int) -> str:
    user = db.find(user_id)
    return user.name  # error: Item "None" of "User | None" has no attribute "name"

# OK: 明示的に None チェック
def get_user(user_id: int) -> str:
    user = db.find(user_id)
    if user is None:
        raise ValueError(f"User {user_id} not found")
    return user.name

# OK: Optional を戻り値の型に含める
def get_user(user_id: int) -> str | None:
    user = db.find(user_id)
    if user is None:
        return None
    return user.name
```

### `Any` を返す関数の戻り値

```python
import json
from typing import Any

# NG: json.loads は Any を返す
def get_config() -> dict[str, str]:
    data = json.loads(config_text)
    return data  # error: Returning Any from function declared to return "dict[str, str]"

# OK: 明示的にキャストして型を付ける
def get_config() -> dict[str, str]:
    data: dict[str, Any] = json.loads(config_text)
    result: dict[str, str] = {}
    for key, value in data.items():
        if not isinstance(key, str) or not isinstance(value, str):
            raise ValueError(f"Invalid config entry: {key!r}={value!r}")
        result[key] = value
    return result
```

### `TypedDict` で辞書に型を付ける

```python
from typing import TypedDict

# NG: 辞書が Any だらけ
def parse_response(data: dict[str, Any]) -> str:
    return data["name"]  # str でも Any でも通る

# OK: TypedDict で構造を定義
class ResponseData(TypedDict):
    name: str
    age: int

def parse_response(data: ResponseData) -> str:
    return data["name"]  # str と確定する
```

### ジェネリクスの具体化

```python
from collections.abc import Sequence

# NG: ジェネリクスが不完全
def first(items: list) -> None:  # error: Missing type parameters for generic type "list"
    pass

# OK: 型パラメータを指定
def first(items: list[int]) -> int | None:
    return items[0] if items else None

# OK: 型変数を使う (複数の型に対応させる場合)
from typing import TypeVar
T = TypeVar("T")

def first_generic(items: list[T]) -> T | None:
    return items[0] if items else None
```

### `Union` / `Optional` の扱い

```python
from __future__ import annotations  # Python 3.10 未満でも | 構文を使う

# OK: Python 3.10+ スタイル
def process(value: int | str | None) -> str:
    if value is None:
        return ""
    if isinstance(value, int):
        return str(value)
    return value  # ここで value は str と確定
```

### プロトコルで Duck Typing を型付け

```python
from typing import Protocol

# NG: Any に頼るか、共通基底クラスが必要
def save(obj: Any) -> None:
    obj.save()

# OK: Protocol でインターフェースを定義
class Saveable(Protocol):
    def save(self) -> None: ...

def save(obj: Saveable) -> None:
    obj.save()
```

### `cast` を使う場合 (最終手段)

`type: ignore` の代わりに `cast` を使うと型情報を明示できる:

```python
from typing import cast

# 外部ライブラリが Any を返すが、型が確実に分かっている場合
raw: Any = some_external_lib.get_result()
result = cast(dict[str, int], raw)  # 型チェッカーに型を伝える
```

> **注意**: `cast` は実行時には何もしない。型が確実な場合のみ使う。
> `isinstance` チェックの方が安全で推奨。

## 型スタブ

```bash
# 公式スタブを確認 (typeshed 収録済みかどうか)
uv run mypy --install-types  # 不足スタブを自動インストール

# サードパーティスタブをインストール
uv add --dev types-requests types-pyyaml types-redis

# スタブが存在しないライブラリは overrides で無視
```

```toml
# スタブがないライブラリのオーバーライド
[[tool.mypy.overrides]]
module = ["some_untyped_lib.*"]
ignore_missing_imports = true
```

## reveal_type でデバッグ

```python
# mypy 実行時のみ有効 (実行時は削除すること)
x = some_complex_expression()
reveal_type(x)  # mypy が推論した型を表示: note: Revealed type is "int"
```

## CI 統合

```yaml
# .github/workflows/type-check.yml
- name: Type check
  run: uv run mypy src/
```

```toml
# pre-commit の場合
[tool.mypy]
# 設定はここに集約し、コマンドは引数なしで実行できるようにする
```

## よくあるエラーコード早見表

| エラーコード | 意味 | 解決策 |
|------------|------|--------|
| `attr-defined` | 属性が存在しない | isinstance チェックで型を絞り込む |
| `return-value` | 戻り値の型が不一致 | 戻り値の型アノテーションを修正 |
| `arg-type` | 引数の型が不一致 | 引数を正しい型に変換する |
| `assignment` | 代入の型が不一致 | 変数アノテーションを修正 |
| `no-untyped-def` | 型アノテーションなし | 全引数・戻り値に型を付ける |
| `import-untyped` | スタブなしライブラリ | `types-*` パッケージをインストール |
| `misc` | その他 | エラーメッセージを詳しく読む |

## See Also

- `python-code-quality` - ruff と組み合わせたコード品質管理
- `basedpyright-type-checking` - より高速な型チェッカー (mypy 代替)
- `python-testing` - pytest と型チェックの組み合わせ
- `uv-project-management` - 依存関係管理

## References

- mypy 公式ドキュメント: https://mypy.readthedocs.io/
- typeshed (標準スタブ): https://github.com/python/typeshed
- mypy エラーコード一覧: https://mypy.readthedocs.io/en/stable/error_codes.html
- 型システム仕様 (PEP 484): https://peps.python.org/pep-0484/
