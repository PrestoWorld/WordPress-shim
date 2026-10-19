# ShimLoader — Lazy Loader

`ShimLoader` là cơ chế nạp **lazily** các file shim WordPress (global functions + class aliases) của package `prestoworld/wordpress-shim`. Thay vì nạp hết qua `composer autoload.files` trên mỗi request, loader chỉ `require` đúng nhóm file cần tại đúng thời điểm.

- Class: `Prestoworld\WordpressShim\ShimLoader` (`src/ShimLoader.php`)
- Class alias map: `Prestoworld\WordpressShim\ClassShimMap` (`src/ClassShimMap.php`)

---

## 1. Vì sao dùng lazy loading?

Mục tiêu gốc (spec 10 §10.7.1) là **tối ưu OpCache**:

- **`autoload.files` (không dùng)**: Composer nạp toàn bộ file function ngay khi autoload khởi động — mọi request, kể cả route không cần WP API. Đơn giản nhưng luôn tốn chi phí parse/execute.
- **Lazy loader (đang dùng)**: chỉ `require_once` đúng nhóm file khi có lời gọi `loadGroup()`. Request nào cần hook/escaping thì nạp, request nào không thì không.

Vì các file function được bảo vệ bằng `if (!function_exists(...))` và nạp bằng `require_once`, việc gọi `loadGroup()` nhiều lần là **idempotent** — không định nghĩa lại, không conflict.

> Lưu ý thực tế: hai callsite chính (xem §4) đều nạp đủ nhóm `hooks` + `compat` (tức cả hai file function), nên về vận hành hiện tại chúng gần như luôn được nạp. Lazy loader giữ lại tính linh hoạt + điểm giao API sạch nếu sau này muốn tách nhóm chặt hơn.

---

## 2. Cấu trúc file

```
wordpress-shim/
  src/
    ShimLoader.php          # loader (class này)
    ClassShimMap.php        # map WP_* → PrestoWorld\Core\*
    functions/
      wp-compatibility.php  # nhóm 'hooks'   — hook API global (add_action, do_action, ...)
      wp-shims.php          # nhóm 'compat' + escaping/options/... — WP API global còn lại
      wp-class-shims.php    # class_alias WP_* → backend (nạp qua autoload, không qua nhóm)
```

---

## 3. Nhóm (group) → file

`ShimLoader` ánh xạ **tên nhóm** → **file** qua hằng `GROUPS`:

| Nhóm | File | Nội dung |
|------|------|----------|
| `hooks` | `wp-compatibility.php` | `add_action`, `add_filter`, `do_action`, `apply_filters`, `remove_*`, `has_*`, `did_*`, `wp_die`, `register_*_hook`, `plugin_*`... |
| `compat` | `wp-shims.php` | `home_url`, `get_option`, `esc_*`, `sanitize_*`, `wp_kses`, nonce, redirect, json, theme support, enqueue... |
| `escaping` | `wp-shims.php` | (nhánh của `compat` — cùng file) |
| `options` | `wp-shims.php` | (nhánh của `compat` — cùng file) |
| `formatting` | `wp-shims.php` | (nhánh của `compat` — cùng file) |
| `url` | `wp-shims.php` | (nhánh của `compat` — cùng file) |
| `users` | `wp-shims.php` | (nhánh của `compat` — cùng file) |
| `all` | — | nạp mọi file duy nhất trong `GROUPS` |

Vì nhiều nhóm trỏ cùng một file, `loadGroup('compat')` hay `loadGroup('escaping')` đều nạp `wp-shims.php` (một lần, nhờ `require_once`).

---

## 4. Callsites (ai gọi loader)

Có **2 chỗ** trong host app kích hoạt loader:

### a) `PrestoWorldServiceProvider::boot()` (app)

```php
use Prestoworld\WordpressShim\ShimLoader;

public function boot(): void
{
    if (function_exists('storage_path')) {
        $shims = new ShimLoader();
        $shims->loadGroup('hooks');    // wp-compatibility.php
        $shims->loadGroup('compat');   // wp-shims.php
        $shims->registerAutoload();    // spl_autoload cho WP_* aliases
    }
    // ...
}
```

Đảm bảo WP API có sẵn cho code app/plugins chạy về sau.

### b) `ThemeBooter::loadCompileLayer()` (gutenberg-core)

```php
// vendor/prestoworld/gutenberg-core/src/Theme/ThemeBooter.php
private static function loadCompileLayer(): void
{
    if (!class_exists(\Prestoworld\WordpressShim\ShimLoader::class)) {
        return;
    }
    $shims = new \Prestoworld\WordpressShim\ShimLoader();
    $shims->loadGroup('hooks');
    $shims->loadGroup('compat');
    $shims->registerAutoload();
}
```

Đây là điểm nạp **trước** khi `require theme/functions.php` và trước khi `after_setup_theme` fire. Theme `functions.php` (ví dụ timeless) gọi `add_theme_support()` trong callback `after_setup_theme` — nên shim **phải** sẵn sàng từ trước, nếu không sẽ lỗi `Call to undefined function add_theme_support()`.

> **Thứ tự quan trọng**: `ThemeBooter` tự nạp shim vì module `context-builder` có thể boot trước khi `PrestoWorldServiceProvider::boot()` chạy. Không dựa solely vào provider.

---

## 5. API

### `new ShimLoader(string $directory = '')`

- `$directory = ''` (mặc định): tìm file trong `src/functions/` của package (`__DIR__ . '/functions'`).
- `$directory = '<path>'`: chỉ tìm trong thư mục chỉ định (ghi đè).

### `loadGroup(string $group): bool`

Nạp một nhóm. Trả `false` nếu tên nhóm không tồn tại hoặc file không tìm thấy.

```php
$shims = new ShimLoader();
$shims->loadGroup('hooks');    // true
$shims->loadGroup('nope');     // false (tên nhóm không có)
```

### `loadGroups(array $groups): void`

Nạp nhiều nhóm liên tiếp.

```php
$shims->loadGroups(['hooks', 'compat']);
```

### `loadedGroups(): array`

Trả về danh sách nhóm đã nạp (debug/introspection).

```php
$shims->loadGroup('hooks');
$shims->loadedGroups(); // ['hooks']
```

### `registerAutoload(): void`

Đăng ký một `spl_autoload` resolve tên class `WP_*` → backend PrestoWorld qua `ClassShimMap`:

- `WP_Query`  → `PrestoWorld\Core\Post\PostQuery`
- `WP_Post`   → `PrestoWorld\Core\Post\PostEntity`
- `WP_User`   → `PrestoWorld\Core\User\UserEntity`
- `WP_Error`  → `PrestoWorld\Core\Error\PrestoError`

Autoload chỉ chạy khi lần đầu tiên một tên `WP_*` được reference (lazy). Idempotent — gọi nhiều lần chỉ đăng ký một spl handler.

```php
$shims->registerAutoload();
// ...
$q = new WP_Query(); // lần đầu → spl_autoload → class_alias WP_Query → PostQuery
```

---

## 6. Ví dụ dùng đầy đủ

```php
use Prestoworld\WordpressShim\ShimLoader;

// Nạp shim sớm (ví dụ trong bootstrap/provider)
$shims = new ShimLoader();
$shims->loadGroups(['hooks', 'compat']); // hook API + WP API global
$shims->registerAutoload();              // WP_* class aliases

// Sau đó mọi WP API đều dùng được:
add_action('init', function () { /* ... */ });
do_action('init');
$html = esc_html($untrusted);
$opt  = get_option('blogname', '');
```

---

## 7. Thêm nhóm / file mới

1. **Thêm file** vào `src/functions/` (ví dụ `wp-media.php`), mỗi hàm bọc trong `if (!function_exists('ten_ham')) { ... }`.
2. **Đăng ký nhóm** trong `ShimLoader::GROUPS`:

   ```php
   private const GROUPS = [
       // ...
       'media' => 'wp-media.php',
   ];
   ```

3. **Nạp nhóm** tại callsite cần (hoặc thêm vào `loadGroups([...])` hiện có).

Nếu file mới trỏ cùng một file đã có (như các nhóm con của `wp-shims.php`), không cần thêm gì đặc biệt — `require_once` tự dedupe.

---

## 8. Lazy loader vs `composer autoload.files`

| | Lazy loader (`ShimLoader`) | `autoload.files` |
|---|---|---|
| Thời điểm nạp | Khi `loadGroup()` được gọi | Ngay khi Composer autoload chạy |
| Chi phí/route | Chỉ route cần WP API | Mọi request |
| Kiểm soát | Tách nhóm, idempotent | Tất cả-or-nothing |
| OpCache | Ít file parse hơn khi route không dùng WP API | Luôn parse đủ |
| Robustness | Phụ thuộc callsite chạy đúng thứ tự | Luôn sẵn sàng |

Hiện package **không** khai báo `autoload.files`; global functions nạp qua `ShimLoader` tại 2 callsite (§4). Nếu sau này muốn chắc chắn luôn sẵn sàng bất kể thứ tự boot, có thể thêm `autoload.files` vào `composer.json` — nhưng sẽ mất tính lazy/OpCache tối ưu.

---

## 9. Ranh giới package

`ShimLoader` thuộc **WP API bridge layer** của package:

- **Package chứa**: global WP functions (`wp-shims.php`, `wp-compatibility.php`), WP_* class aliases (`wp-class-shims.php` + `ClassShimMap`), và loader (`ShimLoader`).
- **Package KHÔNG chứa**: backend implementation (`PrestoWorld\Core\*` — Escape, Sanitize, OptionRepository...) và Legacy engine (`PrestoWorld\Core\Legacy\*` — LegacyHook, LegacyRegistry, LegacyState, WPQuery...) — tất cả nằm ở **app** (`app/src/Core/`).

Các global function trong package **delegate** sang `PrestoWorld\Core\*` (app backend). Vì vậy package không standalone-reusable nếu thiếu app backend — đây là ranh giới đã chọn để giữ package gọn (bridge) thay vì kéo DB/container vào package.
