# Plan: QA Category CRUD cho Admin & Center

## Context

Hệ thống QA hiện tại sử dụng static categories từ `config/qa_categories.php` (10 categories cố định). Cả admin và center đều cần khả năng tạo/sửa/xóa categories riêng trực tiếp trong giao diện quản lý QA. Sẽ tạo table mới `qa_categories` riêng cho QA, không dùng table `categories` cũ. Phân biệt admin/center qua `center_id`: `center_id = 0` là của admin, `center_id > 0` là của center tương ứng.

## Yêu cầu

- Admin và center có categories riêng biệt, đầy đủ CRUD
- Tạo table mới `qa_categories` riêng cho QA (không dùng table `categories` cũ)
- Phân biệt qua `center_id`: `0` = admin, `> 0` = center cụ thể
- Category management UI dạng Dialog/Modal riêng biệt
- Bỏ static config, chuyển hoàn toàn sang database

---

## Giai đoạn 1: Database & Backend

### 1.1 Tạo Migration: Tạo table mới `qa_categories`

**File mới**: `database/migrations/2025_04_22_create_qa_categories_table.php`

**Schema**:
```php
Schema::create('qa_categories', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->unsignedBigInteger('center_id')->default(0);  // 0 = admin, > 0 = center
    $table->unsignedSmallInteger('order')->default(0);
    $table->timestamps();

    $table->index('center_id');
});
```

**Chú thích**: `center_id = 0` là categories của admin, `center_id > 0` là categories của center tương ứng. Không dùng foreign key cho center_id vì giá trị 0 không tồn tại trong centers table.

---

### 1.2 Tạo QACategory Model

**File mới**: `app/Models/QACategory.php`

```php
class QACategory extends Model
{
    public const ADMIN_CENTER_ID = 0;

    public $fillable = [
        'name',
        'center_id',
        'order',
    ];
}
```

Không dùng scope trong model. Phân biệt admin/center bằng Query Builders (theo kiến trúc hiện tại của dự án).

---

### 1.3 Tạo Query Builders cho QACategory

**File mới**: `app/Models/QACategory/Queries/QACategoryQueryBuilder.php`
```php
class QACategoryQueryBuilder
{
    public static function whereCenterIdQuery(int $centerId): QueryInterface
    {
        return new WhereCenterIdQuery($centerId);
    }
}
```

**File mới**: `app/Models/QACategory/Queries/WhereCenterIdQuery.php`
```php
class WhereCenterIdQuery implements QueryInterface {
    public function __construct(private readonly int $centerId) {}
    public function getQuery(Builder $builder): void {
        $builder->where('center_id', $this->centerId);
    }
}
```

---

### 1.4 Tạo QACategoryService

**File mới**: `app/Services/QACategoryService.php`

Extends `BaseCURDService`, chứa:
- `getAdminCategories()` - Lấy categories có `center_id = 0`
- `getCenterCategories(int $centerId)` - Lấy categories có `center_id = $centerId`
- `getAdminCategoryById(int $id)` - Lấy category admin theo ID
- `getCenterCategoryById(int $id, int $centerId)` - Lấy category center theo ID (verify ownership)

---

### 1.5 Thêm qaCategoryService vào AppService

**File sửa**: `app/Services/AppService.php`

```php
public static function qaCategoryService(): QACategoryService
{
    return self::make(QACategoryService::class);
}
```

---

### 1.6 Tạo Request Validators

**File mới**: `app/Http/Requests/Admin/QACategory/QACategoryCreateRequest.php`
```
Rules: name (required|string|max:255), order (nullable|integer|min:0)
```

**File mới**: `app/Http/Requests/Admin/QACategory/QACategoryUpdateRequest.php`

**File mới**: `app/Http/Requests/Center/QACategory/QACategoryCreateRequest.php`

**File mới**: `app/Http/Requests/Center/QACategory/QACategoryUpdateRequest.php`

---

### 1.7 Tạo Controllers (AJAX, trả về JSON)

**File mới**: `app/Http/Controllers/Admin/QACategoryController.php`

| Method | Route | Chức năng |
|--------|-------|-----------|
| `index()` | GET `/api/admin/qa-categories` | List admin categories (`center_id = 0`) |
| `store()` | POST `/api/admin/qa-categories` | Tạo category mới (`center_id = 0`) |
| `update()` | PUT `/api/admin/qa-categories/{id}` | Cập nhật category |
| `destroy()` | DELETE `/api/admin/qa-categories/{id}` | Xóa category (check QAs liên quan) |
| `options()` | GET `/api/admin/qa-categories/options` | Lấy categories cho select dropdown |

**File mới**: `app/Http/Controllers/Center/QACategoryController.php`

| Method | Route | Chức năng |
|--------|-------|-----------|
| `index()` | GET `/api/center/qa-categories` | List center categories |
| `store()` | POST `/api/center/qa-categories` | Tạo category mới với `center_id` của center |
| `update()` | PUT `/api/center/qa-categories/{id}` | Cập nhật category (verify ownership) |
| `destroy()` | DELETE `/api/center/qa-categories/{id}` | Xóa category (check QAs liên quan) |
| `options()` | GET `/api/center/qa-categories/options` | Lấy categories cho select dropdown |

Tất cả methods trả về JSON response vì gọi qua AJAX.

---

### 1.8 Thêm AJAX Routes

**File sửa**: `routes/admin_api.php` - Thêm vào:
```php
Route::group(['middleware' => ['auth_admin', 'set_admin_guard']], function () {
    Route::resource('qa-categories', QACategoryController::class)->only(['index', 'store', 'update', 'destroy']);
    Route::get('qa-categories/options', [QACategoryController::class, 'options'])->name('qa-categories.options');
});
```
→ Routes sẽ là `/api/admin/qa-categories/...`

**File sửa**: `routes/center_api.php` - Thêm vào:
```php
Route::group(['middleware' => ['auth_center', 'set_center_guard']], function () {
    Route::resource('qa-categories', QACategoryController::class)->only(['index', 'store', 'update', 'destroy']);
    Route::get('qa-categories/options', [QACategoryController::class, 'options'])->name('qa-categories.options');
});
```
→ Routes sẽ là `/api/center/qa-categories/...`

---

### 1.9 Update QuestionAnswer Controllers

**File sửa**: `app/Http/Controllers/Admin/QuestionAnswerController.php`

Thay thế tất cả `config('qa_categories')` bằng:
```php
$categories = AppService::qaCategoryService()->get(
    queryBuilder: [
        QACategoryQueryBuilder::whereCenterIdQuery(QACategory::ADMIN_CENTER_ID),
        BaseQueryBuilder::ordersQuery(['order' => 'asc']),
    ],
    output: BaseOutputBuilder::getCollection()
);
```

**File sửa**: `app/Http/Controllers/Center/QuestionAnswerController.php`

Tương tự, thay thế `config('qa_categories')` bằng dynamic query với `center_id` của center hiện tại.

---

### 1.10 Update QuestionAnswer Request Validators

**File sửa**: `app/Http/Requests/Admin/QuestionAnswer/QuestionAnswerCreateRequest.php`
```php
// Thay thế:
'category_id' => ['required', 'integer', Rule::in(range(1, 10))],
// Bằng:
'category_id' => ['required', 'integer', Rule::exists('qa_categories', 'id')],
```

**Files sửa tương tự**:
- `app/Http/Requests/Admin/QuestionAnswer/QuestionAnswerEditRequest.php`
- Center equivalents (nếu có)

---

## Giai đoạn 2: Frontend

### 2.1 Update Type Definitions

**File sửa**: `resources/js/modules/question_answer/index.ts`

Update `CategoryInterface`:
```typescript
export interface CategoryInterface {
    id: number;
    name: string;
    order?: number;
    center_id?: number;
}
```

---

### 2.2 Tạo QA Category Module

**File mới**: `resources/js/modules/qa_category/index.ts`

```typescript
export interface QACategoryFormData {
    name: string;
    order?: number;
}

export const qaCategorySchema = z.object({
    name: z.string().min(1, 'カテゴリー名を入力してください').max(255),
    order: z.number().min(0).optional(),
});

// API functions (gọi AJAX qua axios)
export const getAdminQACategories = async () => { ... };
export const createAdminQACategory = async (data: QACategoryFormData) => { ... };
export const updateAdminQACategory = async (id: number, data: QACategoryFormData) => { ... };
export const deleteAdminQACategory = async (id: number) => { ... };
export const getAdminQACategoryOptions = async () => { ... };

// Center versions
export const getCenterQACategories = async () => { ... };
export const createCenterQACategory = async (data: QACategoryFormData) => { ... };
export const updateCenterQACategory = async (id: number, data: QACategoryFormData) => { ... };
export const deleteCenterQACategory = async (id: number) => { ... };
export const getCenterQACategoryOptions = async () => { ... };
```

API functions dùng `axios` trực tiếp (theo pattern `resources/js/modules/center/api.ts`), gọi đến `/api/admin/qa-categories/...` hoặc `/api/center/qa-categories/...`.

---

### 2.3 Tạo Category Management Modal (Admin)

**File mới**: `resources/js/admin/pages/questionAnswer/qa-category-modal.tsx`

**UI Structure**:
- Dialog/Modal wrapper (shadcn/ui Dialog)
- Header: "カテゴリー管理"
- Category list: table hiển thị name, order, số QAs liên quan, nút edit/delete
- Form thêm/sửa category: input name, input order, buttons save/cancel
- Xử lý CRUD qua AJAX calls (axios) đến `/api/admin/qa-categories/...`
- Callback `onCategoryChange` khi categories thay đổi để refresh dropdown

**Props**:
```typescript
interface QACategoryModalProps {
    open: boolean;
    onOpenChange: (open: boolean) => void;
    onCategoryChange: () => void;
}
```

---

### 2.4 Tạo Category Management Modal (Center)

**File mới**: `resources/js/center/pages/questionAnswer/qa-category-modal.tsx`

Tương tự admin version nhưng gọi `/api/center/qa-categories/...`.

---

### 2.5 Update QA Forms

**File sửa**: `resources/js/admin/pages/questionAnswer/question-answer-form.tsx`

**Thay đổi**:
- Thêm state `categories` và fetch từ AJAX (thay vì chỉ nhận qua props)
- Thêm button "カテゴリー管理" bên cạnh category dropdown
- Tích hợp `QACategoryModal`
- Khi category CRUD xong → refresh categories list → update dropdown

**UI thay đổi ở phần category dropdown**:
```tsx
<div className="flex items-end gap-2">
    <FormField name="category_id" ... /> {/* existing dropdown */}
    <Button type="button" variant="outline" onClick={() => setCategoryModalOpen(true)}>
        <Settings className="h-4 w-4" />
    </Button>
</div>
```

**File sửa**: `resources/js/center/pages/questionAnswer/question-answer-form.tsx`

Tương tự.

---

### 2.6 Update QA Pages

**File sửa**: `resources/js/admin/pages/questionAnswer/question-answer-page.tsx`

**Thay đổi**:
- Thêm button "カテゴリー管理" ở header (bên cạnh "FAQを追加")
- Tích hợp `QACategoryModal`
- Khi categories thay đổi → refresh page data

**File sửa**: `resources/js/center/pages/questionAnswer/question-answer-page.tsx`

Tương tự.

---

### 2.7 Update QA Tables

**File sửa**: `resources/js/admin/pages/questionAnswer/question-answer-table.tsx`
- Update để handle dynamic categories (có thể categories thay đổi)

**File sửa**: `resources/js/center/pages/questionAnswer/question-answer-table.tsx`

Tương tự.

---

## Giai đoạn 3: Cleanup

### 3.1 Xóa Static Config

- Xóa file `config/qa_categories.php`
- Xóa references đến `config('qa_categories')` trong toàn bộ codebase

---

## Tóm tắt Files

### Files mới cần tạo:

| File | Mô tả |
|------|--------|
| `database/migrations/2025_04_22_create_qa_categories_table.php` | Tạo table mới `qa_categories` |
| `app/Models/QACategory.php` | Model cho qa_categories table |
| `app/Models/QACategory/Queries/QACategoryQueryBuilder.php` | Query builder factory |
| `app/Models/QACategory/Queries/WhereCenterIdQuery.php` | Query builder theo center_id |
| `app/Services/QACategoryService.php` | Service layer cho CRUD |
| `app/Http/Requests/Admin/QACategory/QACategoryCreateRequest.php` | Validation tạo category admin |
| `app/Http/Requests/Admin/QACategory/QACategoryUpdateRequest.php` | Validation cập nhật category admin |
| `app/Http/Requests/Center/QACategory/QACategoryCreateRequest.php` | Validation tạo category center |
| `app/Http/Requests/Center/QACategory/QACategoryUpdateRequest.php` | Validation cập nhật category center |
| `app/Http/Controllers/Admin/QACategoryController.php` | Controller CRUD categories admin (AJAX/JSON) |
| `app/Http/Controllers/Center/QACategoryController.php` | Controller CRUD categories center (AJAX/JSON) |
| `resources/js/modules/qa_category/index.ts` | TypeScript types, Zod schemas & AJAX API functions |
| `resources/js/admin/pages/questionAnswer/qa-category-modal.tsx` | Modal quản lý categories admin |
| `resources/js/center/pages/questionAnswer/qa-category-modal.tsx` | Modal quản lý categories center |

### Files cần sửa:

| File | Thay đổi |
|------|----------|
| `app/Services/AppService.php` | Thêm `qaCategoryService()` accessor |
| `routes/admin_api.php` | Thêm QA category AJAX routes (`/api/admin/qa-categories/...`) |
| `routes/center_api.php` | Thêm QA category AJAX routes (`/api/center/qa-categories/...`) |
| `app/Http/Controllers/Admin/QuestionAnswerController.php` | Thay `config()` bằng dynamic query |
| `app/Http/Controllers/Center/QuestionAnswerController.php` | Thay `config()` bằng dynamic query |
| `app/Http/Requests/Admin/QuestionAnswer/QuestionAnswerCreateRequest.php` | Update category_id validation |
| `app/Http/Requests/Admin/QuestionAnswer/QuestionAnswerEditRequest.php` | Update category_id validation |
| `resources/js/modules/question_answer/index.ts` | Update CategoryInterface |
| `resources/js/admin/pages/questionAnswer/question-answer-form.tsx` | Thêm category modal integration |
| `resources/js/admin/pages/questionAnswer/question-answer-page.tsx` | Thêm category management button |
| `resources/js/center/pages/questionAnswer/question-answer-form.tsx` | Thêm category modal integration |
| `resources/js/center/pages/questionAnswer/question-answer-page.tsx` | Thêm category management button |

### Files cần xóa:

| File | Lý do |
|------|--------|
| `config/qa_categories.php` | Thay bằng database-driven categories |

---

## Thứ tự triển khai

1. Migration → 2. Model → 3. Query Builders → 4. Service → 5. Validators → 6. Controllers → 7. Routes → 8. Update existing controllers/validators → 9. Frontend types → 10. Frontend components → 11. Integration → 12. Cleanup
