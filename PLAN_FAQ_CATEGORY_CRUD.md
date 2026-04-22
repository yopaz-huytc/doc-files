# Plan: FAQ Category CRUD cho Admin & Center

## Context

Hệ thống FAQ hiện tại sử dụng static categories từ `config/qa_categories.php` (10 categories cố định). Cả admin và center đều cần khả năng tạo/sửa/xóa categories riêng trực tiếp trong giao diện quản lý FAQ. Category table hiện tại đã có sẵn và sẽ được mở rộng thành đa hình (polymorphic) để hỗ trợ cả admin và center.

## Yêu cầu

- Admin và center có categories riêng biệt, đầy đủ CRUD
- Dùng chung 1 table `categories` với polymorphic design (owner_type)
- Category management UI dạng Dialog/Modal riêng biệt
- Bỏ static config, chuyển hoàn toàn sang database

---

## Giai đoạn 1: Database & Backend

### 1.1 Tạo Migration: Thêm polymorphic fields vào `categories` table

**File mới**: `database/migrations/2025_04_22_add_polymorphic_fields_to_categories_table.php`

**Schema changes**:
```php
Schema::table('categories', function (Blueprint $table) {
    $table->string('owner_type')->nullable()->after('level');     // 'admin' hoặc 'center'
    $table->unsignedBigInteger('owner_id')->nullable()->after('owner_type');
    $table->unsignedBigInteger('center_id')->nullable()->after('owner_id');

    $table->index('owner_type');
    $table->index('owner_id');
    $table->index('center_id');
    $table->foreign('center_id')->references('id')->on('centers')->onDelete('cascade');
});
```

- Update existing records: `owner_type = 'admin'`

---

### 1.2 Update Category Model

**File sửa**: `app/Models/Category.php`

**Thay đổi**:
- Thêm constants: `OWNER_TYPE_ADMIN = 'admin'`, `OWNER_TYPE_CENTER = 'center'`
- Update `$fillable`: thêm `owner_type`, `owner_id`, `center_id`
- Thêm relationship `owner()` (morphTo)
- Thêm relationship `center()` (belongsTo Center)
- Thêm scope `forAdmin()`: `where('owner_type', self::OWNER_TYPE_ADMIN)`
- Thêm scope `forCenter($centerId)`: `where('owner_type', self::OWNER_TYPE_CENTER)->where('center_id', $centerId)`

---

### 1.3 Tạo Query Builders cho Category

**File mới**: `app/Models/Category/Queries/WhereOwnerTypeQuery.php`
```php
class WhereOwnerTypeQuery implements QueryInterface {
    public function __construct(private readonly string $ownerType) {}
    public function getQuery(Builder $builder): void {
        $builder->where('owner_type', $this->ownerType);
    }
}
```

**File mới**: `app/Models/Category/Queries/WhereCenterIdQuery.php`
```php
class WhereCenterIdQuery implements QueryInterface {
    public function __construct(private readonly int $centerId) {}
    public function getQuery(Builder $builder): void {
        $builder->where('center_id', $this->centerId);
    }
}
```

**File sửa**: `app/Models/Category/Queries/CategoryQueryBuilder.php`
- Thêm static methods: `whereOwnerTypeQuery($ownerType)`, `whereCenterIdQuery($centerId)`

---

### 1.4 Update CategoryService

**File sửa**: `app/Services/CategoryService.php`

**Thêm methods**:
- `getAdminCategories()` - Lấy tất cả categories của admin
- `getCenterCategories(int $centerId)` - Lấy categories của center cụ thể
- `getAdminCategoryById(int $id)` - Lấy category admin theo ID
- `getCenterCategoryById(int $id, int $centerId)` - Lấy category center theo ID (verify ownership)

---

### 1.5 Thêm categoryService vào AppService

**File sửa**: `app/Services/AppService.php`

```php
public static function categoryService(): CategoryService
{
    return self::make(CategoryService::class);
}
```

---

### 1.6 Tạo Request Validators

**File mới**: `app/Http/Requests/Admin/FAQCategory/FAQCategoryCreateRequest.php`
```
Rules: name (required|string|max:255), order (nullable|integer|min:0)
```

**File mới**: `app/Http/Requests/Admin/FAQCategory/FAQCategoryUpdateRequest.php`

**File mới**: `app/Http/Requests/Center/FAQCategory/FAQCategoryCreateRequest.php`

**File mới**: `app/Http/Requests/Center/FAQCategory/FAQCategoryUpdateRequest.php`

---

### 1.7 Tạo Controllers

**File mới**: `app/Http/Controllers/Admin/FAQCategoryController.php`

| Method | Route | Chức năng |
|--------|-------|-----------|
| `index()` | GET `/admin/faq-categories` | List admin categories |
| `store()` | POST `/admin/faq-categories` | Tạo category mới |
| `update()` | PUT `/admin/faq-categories/{id}` | Cập nhật category |
| `destroy()` | DELETE `/admin/faq-categories/{id}` | Xóa category (check FAQs liên quan) |
| `options()` | GET `/admin/faq-categories/options` | Lấy categories cho select dropdown |

**File mới**: `app/Http/Controllers/Center/FAQCategoryController.php`

| Method | Route | Chức năng |
|--------|-------|-----------|
| `index()` | GET `/center/faq-categories` | List center categories |
| `store()` | POST `/center/faq-categories` | Tạo category mới |
| `update()` | PUT `/center/faq-categories/{id}` | Cập nhật category |
| `destroy()` | DELETE `/center/faq-categories/{id}` | Xóa category (check FAQs liên quan) |
| `options()` | GET `/center/faq-categories/options` | Lấy categories cho select dropdown |

---

### 1.8 Thêm Routes

**File sửa**: `routes/admin.php` - Thêm vào trong group `middleware: auth_admin`:
```php
Route::resource('faq-categories', FAQCategoryController::class)->only(['index', 'store', 'update', 'destroy']);
Route::get('faq-categories/options', [FAQCategoryController::class, 'options'])->name('faq-categories.options');
```

**File sửa**: `routes/center.php` - Thêm vào trong group `middleware: auth_center`:
```php
Route::resource('faq-categories', FAQCategoryController::class)->only(['index', 'store', 'update', 'destroy']);
Route::get('faq-categories/options', [FAQCategoryController::class, 'options'])->name('faq-categories.options');
```

---

### 1.9 Update QuestionAnswer Controllers

**File sửa**: `app/Http/Controllers/Admin/QuestionAnswerController.php`

Thay thế tất cả `config('qa_categories')` bằng:
```php
$categories = AppService::categoryService()->get(
    queryBuilder: [
        CategoryQueryBuilder::whereOwnerTypeQuery(Category::OWNER_TYPE_ADMIN),
        BaseQueryBuilder::ordersQuery(['order' => 'asc']),
    ],
    output: BaseOutputBuilder::getCollection()
);
```

**File sửa**: `app/Http/Controllers/Center/QuestionAnswerController.php`

Tương tự, thay thế `config('qa_categories')` bằng dynamic query với `OWNER_TYPE_CENTER` + `center_id`.

---

### 1.10 Update QuestionAnswer Request Validators

**File sửa**: `app/Http/Requests/Admin/QuestionAnswer/QuestionAnswerCreateRequest.php`
```php
// Thay thế:
'category_id' => ['required', 'integer', Rule::in(range(1, 10))],
// Bằng:
'category_id' => ['required', 'integer', Rule::exists('categories', 'id')],
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
    owner_type?: 'admin' | 'center';
    owner_id?: number | null;
    center_id?: number | null;
}
```

---

### 2.2 Tạo FAQ Category Module

**File mới**: `resources/js/modules/faq_category/index.ts`

```typescript
export interface FAQCategoryFormData {
    name: string;
    order?: number;
}

export const faqCategorySchema = z.object({
    name: z.string().min(1, 'カテゴリー名を入力してください').max(255),
    order: z.number().min(0).optional(),
});
```

---

### 2.3 Tạo Category Management Modal (Admin)

**File mới**: `resources/js/admin/pages/questionAnswer/faq-category-modal.tsx`

**UI Structure**:
- Dialog/Modal wrapper (shadcn/ui Dialog)
- Header: "カテゴリー管理"
- Category list: table hiển thị name, order, số FAQs liên quan, nút edit/delete
- Form thêm/sửa category: input name, input order, buttons save/cancel
- Xử lý CRUD qua Inertia.js router calls
- Callback `onCategoryChange` khi categories thay đổi để refresh dropdown

**Props**:
```typescript
interface FAQCategoryModalProps {
    open: boolean;
    onOpenChange: (open: boolean) => void;
    onCategoryChange: () => void;
}
```

---

### 2.4 Tạo Category Management Modal (Center)

**File mới**: `resources/js/center/pages/questionAnswer/faq-category-modal.tsx`

Tương tự admin version nhưng gọi center routes.

---

### 2.5 Update FAQ Forms

**File sửa**: `resources/js/admin/pages/questionAnswer/question-answer-form.tsx`

**Thay đổi**:
- Thêm state `categories` và fetch từ API (thay vì chỉ nhận qua props)
- Thêm button "カテゴリー管理" bên cạnh category dropdown
- Tích hợp `FAQCategoryModal`
- Khi category CRUD xong → refresh categories list → update dropdown

**UI thay đổi ở phần category dropdown**:
```tsx
<div className="flex items-end gap-2">
    <FormField name="category_id" ... /> {/* existing dropdown, w-6/12 */}
    <Button type="button" variant="outline" onClick={() => setCategoryModalOpen(true)}>
        <Settings className="h-4 w-4" />
    </Button>
</div>
```

**File sửa**: `resources/js/center/pages/questionAnswer/question-answer-form.tsx`

Tương tự.

---

### 2.6 Update FAQ Pages

**File sửa**: `resources/js/admin/pages/questionAnswer/question-answer-page.tsx`

**Thay đổi**:
- Thêm button "カテゴリー管理" ở header (bên cạnh "FAQを追加")
- Tích hợp `FAQCategoryModal`
- Khi categories thay đổi → refresh page data

**File sửa**: `resources/js/center/pages/questionAnswer/question-answer-page.tsx`

Tương tự.

---

### 2.7 Update FAQ Tables

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
| `database/migrations/2025_04_22_add_polymorphic_fields_to_categories_table.php` | Migration thêm polymorphic fields |
| `app/Models/Category/Queries/WhereOwnerTypeQuery.php` | Query builder theo owner_type |
| `app/Models/Category/Queries/WhereCenterIdQuery.php` | Query builder theo center_id |
| `app/Http/Requests/Admin/FAQCategory/FAQCategoryCreateRequest.php` | Validation tạo category admin |
| `app/Http/Requests/Admin/FAQCategory/FAQCategoryUpdateRequest.php` | Validation cập nhật category admin |
| `app/Http/Requests/Center/FAQCategory/FAQCategoryCreateRequest.php` | Validation tạo category center |
| `app/Http/Requests/Center/FAQCategory/FAQCategoryUpdateRequest.php` | Validation cập nhật category center |
| `app/Http/Controllers/Admin/FAQCategoryController.php` | Controller CRUD categories admin |
| `app/Http/Controllers/Center/FAQCategoryController.php` | Controller CRUD categories center |
| `resources/js/modules/faq_category/index.ts` | TypeScript types & Zod schemas |
| `resources/js/admin/pages/questionAnswer/faq-category-modal.tsx` | Modal quản lý categories admin |
| `resources/js/center/pages/questionAnswer/faq-category-modal.tsx` | Modal quản lý categories center |

### Files cần sửa:

| File | Thay đổi |
|------|----------|
| `app/Models/Category.php` | Thêm polymorphic support, scopes |
| `app/Models/Category/Queries/CategoryQueryBuilder.php` | Thêm static methods mới |
| `app/Services/CategoryService.php` | Thêm role-specific methods |
| `app/Services/AppService.php` | Thêm `categoryService()` accessor |
| `routes/admin.php` | Thêm FAQ category routes |
| `routes/center.php` | Thêm FAQ category routes |
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
