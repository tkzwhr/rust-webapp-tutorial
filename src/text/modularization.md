# モジュール化

コードが増えてくると、1ファイルの実装では見通しが悪くなってくるため、このタイミングでモジュール化を行います。

<a class="source" href="https://github.com/tkzwhr/rustwi/compare/2...3?diff=split" target="_blank" rel="noopener noreferrer">
    <div class="icon">&nbsp;</div>
    <span>今回のコード</span>
</a>

## モジュール概要

モジュールについては [The Rust Programming Language 日本語版](https://doc.rust-jp.rs/book-ja/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html) の7章にて解説されているため、そちらに沿ってファイルに分割していきます。

## 階層化のポイント

rustでは **ファイルを作って `mod` 宣言しただけではモジュールにならない** ので注意が必要です。特にディレクトリを階層構造にする場合に、整理整頓のためだけのサブディレクトリであっても、rustではモジュールとして扱うため、モジュール宣言をする必要があります。サブディレクトリを含むモジュール宣言にはいくつか方式があります。

#### 方式A) ディレクトリ構造は `lib.rs` に記載する

階層化の階数やファイル数が多くなると `lib.rs` の記載量が多くなりますが、作成するファイルが少なくてすむため、今回はこの方式で実装してみます。

```text
src
└── lib.rs
    ├── dir_a
    │   └── module_b.rs
    └── module_c.rs
```

<div class="filename"><div>src/lib.rs</div></div>

```rust,ignore
mod dir_a {
    mod module_b;
    pub use module_b::fn_b;
}

mod module_c;

pub use dir_a::fn_b;
pub use module_c::fn_c;
```

#### 方式B)  ディレクトリ名と同名のファイルを作成する

ディレクトリと同名のファイルを作成することで、そのディレクトリの内部にあるモジュールを宣言することができます。

```text
src
└── lib.rs
    ├── dir_a.rs
    ├── dir_a
    │   └── module_b.rs
    └── module_c.rs
```

<div class="filename"><div>src/lib.rs</div></div>

```rust,ignore
mod dir_a;
mod module_c;

pub use dir_a::fn_b;
pub use module_c::fn_c;
```

<div class="filename"><div>src/dir_a.rs</div></div>

```rust,ignore
mod module_b;

pub use module_b::fn_b;
```

#### 方式C)  `mod.rs` を作成する

`mod.rs` をディレクトリ内に作成することで、そのディレクトリにあるモジュールを宣言することができます。

> **Note:** Rust 2018 Editionからは方式Bがスタンダードになりました。この方式はRust 2015 Editionとの後方互換のために残されているようです。

```text
src
└── lib.rs
    ├── dir_a
    │   ├── mod.rs
    │   └── module_b.rs
    └── module_c.rs
```

<div class="filename"><div>src/lib.rs</div></div>

```rust,ignore
mod dir_a;
mod module_b;

pub use dir_a::fn_b;
pub use module_c::fn_c;
```

<div class="filename"><div>src/dir_a/mod.rs</div></div>

```rust,ignore
mod module_b;

pub use module_b::fn_b;
```
