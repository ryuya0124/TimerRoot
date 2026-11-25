# M08 KernelSU Late-Load Chain: End-to-End Analysis

FCNT arrows Alpha (M08) — `6.1.157-android14-11-g14047afecd8b-ab16012876`
GKI tag: `android14-6.1-2025-12_r29`

## Verdict: Chain Should Work — No Fundamental Blockers

---

## 1. MODULE_SIG_PROTECT=y は unsigned モジュールのロードを許可する

GKI android14-6.1 の `kernel/module/signing.c` から:

```c
#ifdef CONFIG_MODULE_SIG_PROTECT
#define sig_enforce false   // ← hardcoded macro
#endif
```

`module_sig_check()` の動作:
- unsigned モジュール → `sig_enforce` チェック → `false` → return 0 (許可)
- ソースコメント: *"Do not prevent loading of unsigned modules; as all modules except GKI modules are not signed."*
- `security_locked_down()` チェックもスキップされる

**結論: MODULE_SIG_PROTECT=y は署名なしモジュールのロードをブロックしない。**
MODULE_SIG_PROTECT の役割は「protected symbol」へのアクセスを制限すること。

Sources:
- https://android.googlesource.com/kernel/common/+/refs/heads/android14-6.1/kernel/module/signing.c

## 2. KernelSU ksuinit のモジュールロード機構

ソース: `tiann/KernelSU/userspace/ksuinit/src/lib.rs`

### 2.1 ロード手順

1. `.ko` を ELF パース (goblin)、未定義シンボル (SHN_UNDEF) を収集
2. `/proc/sys/kernel/kptr_restrict` に `1` を書き込み (デフォルト2→1に緩和)
3. `/proc/kallsyms` を CAP_SYSLOG 権限で読み取り、全カーネルシンボルアドレスを取得
4. 未定義シンボルを解決済みアドレスでパッチ (`st_shndx = SHN_ABS`, `st_value = addr`)
5. `init_module(buffer, params)` syscall でロード
6. vermagic 不一致なら .modinfo をパッチしてリトライ

### 2.2 MODULE_SIG_PROTECT のシンボル保護を完全にバイパス

ksuinit は `/proc/kallsyms` からシンボルを解決する。kallsyms には:
- エクスポートされたシンボル
- **エクスポートされていないシンボル** (通常のモジュールローダーではアクセス不可)
- **protected シンボル** (MODULE_SIG_PROTECT で保護されたもの)

解決済みシンボルは `SHN_ABS` としてパッチされるため、カーネルは再解決を試みない。
→ MODULE_SIG_PROTECT のシンボルアクセス制限は完全にバイパスされる。

### 2.3 late-load パス

`late_load.rs`:
```rust
// 3. Get kernelsu.ko from embedded assets
let ko_name = format!("{kmi}_kernelsu.ko");
let ko_data = assets::get_asset_data(&ko_name)?;
// 4. Load kernelsu.ko from memory with manual relocation
ksuinit::load_module(&ko_data, params)?;
```

ksud は KMI バージョンに対応する .ko を内蔵アセットから取得してロードする。

## 3. Capability チェーン

### 3.1 Fake cred の構成

```
c[5-9]: cap_inh/prm/eff/bset/ambient = FULLCAP (0x1ffffffffffULL = 全41ケーパビリティ)
+0x78: security ptr → blob (SID=1, kernel domain)
+0x80: user → ROOT_USER + slide
+0x88: user_ns → INIT_USER_NS + slide
```

### 3.2 execve を通じたケーパビリティ継承

```
timer fake → child (fake cred, FULLCAP)
  → fork worker
    → execve ksu_stage.sh
      ambient caps: P(amb) & P(inh) & P(prm) = FULLCAP
      → P'(prm) = P'(amb) = FULLCAP
      → P'(eff) = FULLCAP
      → sh (FULLCAP)
        → ksud (FULLCAP via ambient caps)
```

CAP_SYS_MODULE (bit 16) は FULLCAP に含まれる → `init_module()` syscall の権限チェック通過。
CAP_SYSLOG (bit 34) も含まれる → kptr_restrict=1 で kallsyms 読み取り可能。

### 3.3 SELinux

exploit で `selinux_state` のenforcing バイトをゼロ化済み。
→ MAC チェックなし → sysctl 書き込み、モジュールロード全て通過。

## 4. 完全なチェーン (19 ステップ)

```
 1. KASLR slide probe          mode=0  R + CRED_UID_OFF → slide 発見
 2. panic_on_warn zeroed       mode=0  PANIC_ON_WARN + slide → 0
 3. fake_cred applied          mode=2  child task cred → FULLCAP + SID_KERNEL
 4. SELinux enforcement off    mode=0  SELINUX_STATE + slide - 7 → 0
 5. uid/gid fixed              mode=0  fake cred uid/gid → 0
 6-10. euid/suid/sgid/egid...  mode=0  残り cred フィールド修正
11. child wakes up             pipe "A" → child reads
12. child forks worker         worker inherits fake cred
13. worker execve              ksu_stage.sh (ambient caps → FULLCAP)
14. ksud located               KernelSU Manager APK から libksud.so
15. ksuinit: kptr_restrict=1   SELinux off + FULLCAP → sysctl 書き込み成功
16. ksuinit: kallsyms read     CAP_SYSLOG → 全シンボルアドレス取得
17. ksuinit: symbol patching   .ko の未定義シンボル → SHN_ABS に変換
18. init_module() syscall      sig_enforce=false → unsigned 許可、CAP_SYS_MODULE → 通過
19. KernelSU hooks via kprobes CONFIG_KPROBES=y → フック成功
```

Post-load:
- ksud: SELinux ルール設定、feature 初期化、スクリプト実行
- ksu_stage.sh: /proc/modules で "kernelsu" 確認 → KSU_LOADED
- ksu_stage.sh: SELinux 再有効化 (`echo 1 > /sys/fs/selinux/enforce`)
- parent: `hold_forever()` → fake cred ページ保持

## 5. 残る実施項目

| # | 項目 | 状態 |
|---|------|------|
| 1 | KernelSU Manager インストール | APK取得済み (/tmp/KernelSU_v3.3.1.apk)、未インストール |
| 2 | デバイス再接続 | M08 オフライン (192.168.3.19:37225) |
| 3 | TimerRoot APK ビルド・インストール | コード修正済み、要ビルド |
| 4 | `timer fake` 実行 | 未テスト |
| 5 | KMI バージョン確認 | ksud が自動検出、android14-6.1 系 |

## 6. リスク要因

| リスク | 影響 | 対策 |
|--------|------|------|
| KMI 不一致 | ksud が .ko を見つけられない | ksuinit の vermagic リトライで自動対応 |
| kallsyms 解決失敗 | 一部シンボル未解決 → ロード失敗 | カーネルの通常解決にフォールバック |
| exploit 中のカーネル破壊 | panic / reboot | 10回の stage_write は既知リスク、リトライ対応 |
| kprobes フック失敗 | KernelSU 機能不全 | CONFIG_KPROBES=y 確認済み |

## 7. M08 カーネル構成 (関連項目)

```
CONFIG_MODULE_SIG_PROTECT=y   → sig_enforce=false (unsigned 許可)
CONFIG_KPROBES=y              → kprobe フック可能
CONFIG_FTRACE=y               → KASLR slide 検出に使用
CONFIG_MODULES=y              → モジュールサポート有効
CONFIG_MODULE_UNLOAD=y        → モジュールアンロード可能
panic_on_warn=1               → exploit で事前にゼロ化必須
```
