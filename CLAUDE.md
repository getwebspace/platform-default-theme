# CLAUDE.md — reference for theme development

This file is a map of **what a front-end theme of WebSpace Platform works
with**: which public actions render which templates, what variables they
receive, which models/services stand behind them, and which custom Twig
functions are available in templates. Read it together with the root
`../CLAUDE.md` (overall architecture) and `../docs/notes/` (history of
specific changes).

> Themes (`theme/*`, except `theme/default`) are not in this git repo — they
> are gitignored and copied in locally. Everything below describes the
> **engine contract**, not a specific theme. When changing a theme, check the
> working tree.

---

## 1. How a request reaches a template

`public/index.php` → `src/bootstrap.php` → Slim `App` (`src/routes.php`,
`src/middleware.php`). The public site is served by actions from
`src/Application/Actions/Common/**`. Each action:

1. calls a domain **service** (never a model directly);
2. calls `respond($template, $data)` — a method of `App\Domain\AbstractAction`.

### Template resolution and the theme

`HasRenderer::render()` (`src/Domain/Traits/HasRenderer.php`), on every request:

- adds to the Twig loader the path
  `THEME_DIR . '/' . parameter('common_theme', 'default')` (i.e.
  `theme/<active-theme>/`);
- adds `src/Template/errors/` (error pages `p4xx/p5xx.twig`);
- the base loader already contains `src/Template/` — that is where
  `layout.twig` and `mixin/*` come from (see §9).

The active theme is set by the **`common_theme`** parameter (admin panel →
parameters). Lookup order for a template by name: first `src/Template/`, then
`theme/<theme>/`, then `src/Template/errors/` (the theme and the error pages
are added with `addPath()`, i.e. *after* the engine's own folder).

⚠️ So **the engine wins a name clash**: a theme's `layout.twig` or
`mixin/<name>.twig` is never loaded while `src/Template/` has a file of that
name — `{% extends 'layout.twig' %}` always gets the engine's skeleton, and
a theme `layout.twig` is dead code (this was documented the other way round
before; check with `{{ _self }}` / the DEBUG error message if in doubt).
Name your own files differently (`main.twig`, `partials/…`). What *can* be
replaced by name are the error pages (`p404.twig`…), because they exist only
in `src/Template/errors/`, which comes last.

### Response format (`respond()`)

`respond()` looks at `?format=` and the `Accept` header:

| Condition | Response |
|---|---|
| `?format=json` or `Accept: application/json` / `*/*` | JSON `{errors, params, data}` |
| `?format=text` or `Accept: text/plain` | text (JSON dump of `data`) |
| otherwise (`?format=html`, `Accept: text/html`) | **Twig template render** |

⚠️ Therefore an AJAX request with `Accept: */*` to a page returns JSON, not
HTML. For partial HTML loading pass `Accept: text/html`.

If a template render fails — `p400.twig` is returned with status 400 and an
`exception` variable. `p400.twig` must render without errors itself.

### Localization

`{% locale %}...{% endlocale %}` (tag), the `|locale` filter and the `_()`
function — via `App\Application\i18n`. A missing key is displayed as-is, with
no error. The UI language is in `_language`, the list of locales in
`_locales`.

---

## 2. Variables available in EVERY template

`HasRenderer::render()` merges these into `$data` for any `respond()`:

| Variable | Type | What it is |
|---|---|---|
| `SHA` | string | First 7 chars of `COMMIT_SHA` or `'specific'` — for asset cache-busting |
| `_request` | array | Escaped copy of `$_REQUEST` |
| `_error` | array | Validation errors of the current request: `{ field: error_code }`. Also stored in `\Alksily\Support\Form::$globalError`, so `form()` highlights invalid fields automatically |
| `_language` | string | Current locale code, e.g. `ru-RU` (default `en-US`) |
| `_locales` | array | List of accepted locales |
| `plugins` | Collection | All registered plugins (`container['plugin']->get()`) |
| `user` | `User`\|`false` | The currently authenticated user, or `false` |

Further, in each section of §3 only the **additional** variables that a
specific action provides are listed.

---

## 3. Public (Common) Actions

All paths are relative to the site root. `template param` is the name of an
admin parameter that overrides the twig file name (the default value is the
second argument). Catalog/publication categories can set their own templates
via the JSON `template` field (see the `CatalogCategory`,
`PublicationCategory` models).

### 3.1 Home page — `MainPageAction`

- **Route:** `GET /` (`common:main`), cached 1 hour
- **Template:** `parameter('common_template', 'main.twig')`
- **Extra variables:** none (only the globals from §2)

### 3.2 Static page — `PageAction`

- **Route:** `GET /{args:.*}` (`common:page`) — the lowest-priority route
- **Service:** `PageService->read(['address' => ...])`
- **Template:** the value of the `page.template` field (each page has its own)
- **Extra variables:**
  - `page` — the `Page` model (see §4). With `type = html` the content is in `page.content`.
- **Errors:** `p400.twig` (400), `p404.twig` (404)

### 3.3 Publications (news/blog) — `Common\Publication\ListAction`

- **Route:** `GET /{category}[/{args:.*}]` (`common:publication:list`), where
  `{category}` is one of the `address` values of existing publication
  categories (the route is built dynamically in `routes.php`). A trailing
  number in the path is the pagination page number (offset).
- **Services:** `PublicationCategoryService`, `PublicationService`
- **Two modes:**

**Category publication list** (`/{category}` or `/{category}/N`):
- **Template:** `category.template['list']` ?? `publication.list.twig`
- **Variables:**
  - `categories` — Collection of all **public** categories (`is_public = true`)
  - `category` — the current `PublicationCategory`
  - `publications` — Collection of `Publication` (paginated), only with `date <= now`
  - `pagination` — `{ count, page, offset }` (`page` = page size)

**Single publication** (`/{category}/.../{address}`):
- **Template:** `category.template['full']` ?? `publication.full.twig`
- **Variables:**
  - `categories` — public categories
  - `category` — the publication's category
  - `publication` — the `Publication` model
- **Errors:** `p404.twig`, `p400.twig`

### 3.4 Catalog — `Common\Catalog\ListAction`

- **Route:** `GET /catalog[/{args:.*}]` (`common:catalog:list`). A trailing
  number is the pagination offset — but the whole path is tried as an address
  first (`/catalog/shoes/42` is a product with address `shoes/42` if there is
  one, else page 42 of `shoes`). Three modes determined by the path:

**Catalog storefront** (`/catalog`):
- **Template:** `parameter('catalog_category_template', 'catalog.category.twig')`
- **Variables:**
  - `categories` — Collection of `CatalogCategory` (`status=work`, `is_hidden=false`)
  - `products` — `{ all, filtered, params, count }`:
    - `all` — all storefront products (Collection of `CatalogProduct`)
    - `filtered` — products after filters/sorting, paginated
    - `params` — `{ address, offset }`
    - `count` — total number
  - `pagination` — `{ count, page, offset }`

**Category** (`/catalog/<address>`):
- **Template:** `category.template['category']`
- **Variables:** `categories`, `category` (`CatalogCategory`),
  `products` (same structure), `pagination`

**Product** (`/catalog/<category>/<product-address>`):
- **Template:** `product.category.template['product']`
- **Variables:**
  - `categories`, `category` (the product's category)
  - `products` — Collection (result of `read` by address)
  - `product` — the `CatalogProduct` model
  - `params` — `{ address, offset }`
- **Errors:** `p404.twig`

**Storefront/category filtering query params** (GET):
`price[min]`, `price[max]`, `country`, `manufacturer`, `order`
(`title`|`price`), `direction` (`asc`|`desc`), plus any attribute `address`
with `is_filter = true` → `?<attr_address>=<value>`. Several values of one
attribute are allowed and match any of them: `?color[]=red&color[]=blue`
(different attributes are combined with AND). The page is cut in SQL
(`forPage`), but `products.all` still loads the whole category, so avoid
iterating it in a template on a big catalog.

### 3.5 Cart / checkout — `Common\Catalog\CartAction`

- **Route:** `GET|POST /cart` (`common:catalog:cart`), `X-Robots-Tag: noindex`
- **Template:** `parameter('catalog_cart_template', 'catalog.cart.twig')`
- **GET:** just a render (the cart contents live on the client, see
  `mixin/catalog.twig` + localStorage)
- **POST (checkout):** fields `delivery` (incl. `delivery[address]` which may
  be an array of strings), `phone`, `email`, `comment`, `shipping`, `system`,
  `payment_uuid` (or `payment`), `products` (array). With
  `catalog_order_fields=on` any extra fields are concatenated into `comment`.
- **POST response:** JSON `{ redirect: '/cart/done/<uuid>' }` (or a payment
  plugin URL). Errors — in `_error` (`email`, `phone`, `shipping`,
  `grecaptcha`).
- **What the server keeps of `products`** (`OrderService::sanitizeCartProducts`):
  only existing products with `status=work` and `count > 0`. The price is the
  catalog one; `price_wholesale` is applied only when `count >=
  priceWholesaleFrom` (whatever the client asked for). `price_type=price_self`
  is a feature: the client's own `price` (and `discount`, `tax`,
  `tax_included`) is kept as posted.
- **Expired access token.** An AJAX POST that meets an expired `access_token`
  cookie is sent through `/auth/refresh-token` and comes back to `/cart` with
  its body (308, see §10); if the session cannot be renewed the cookies are
  cleared and the order goes on as a guest's.

### 3.6 Order placed — `Common\Catalog\CartDoneAction`

- **Route:** `GET /cart/done/{order}` (`common:catalog:cart:done`)
- **Template:** `parameter('catalog_cart_complete_template', 'catalog.cart.complete.twig')`
- **Variables:** `order` — the `CatalogOrder` model (see §4)
- Invalid uuid → redirect to `/cart`; not found → `p404.twig`

### 3.7 Search — `Common\SearchAction`

- **Route:** `GET|POST /search` (`common:search`), `X-Robots-Tag: noindex`
- **Engine:** TNTSearch over the `page`, `publication`, `catalog_product` indexes
- **Template:** `parameter('search_template', 'search.twig')`
- **Query:** `query` (or `q`), `type` (or `t`) — restrict to a type
  (`page`|`publication`|`catalog_product`), `limit`
- **Variables:**
  - `query` — the (cleaned) query string
  - `index_type` — the type filter
  - `result` — Collection of models (products already have `address` prefixed with `catalog/`)
  - `count` — number of results
  - `limit`

### 3.8 Contact form — `Common\FormAction`

- **Route:** `POST /form/{unique}` (`common:form`), `{unique}` = the form's `address`
- **Services:** `FormService`, `FormDataService`
- **Behavior:** saves a `FormData`, sends an email (`SendMailTask`),
  optionally POST-forwards to `form.duplicate`. The email template is
  `form.templateFile` ?? `form.template`; or a ready `body` from the request.
- **Response:** JSON `{ status: 'ok' }`. Form not found → `p404.twig`.
  reCAPTCHA failed → `_error['grecaptcha']`, status 500.
- Files are accepted in the `files` field (if `file_is_enabled=yes`).

### 3.9 Guestbook — `Common\GuestBookAction`

- **Route:** `GET|POST /guestbook[/{page}]` (`common:guestbook`)
- **Service:** `GuestBookService`
- **Template:** `parameter('guestbook_template', 'guestbook.twig')`
- **POST:** fields `name`, `email`, `message`, optional `files`. A new entry
  is created with status `moderate`. Errors → `_error` (`name`/`email`/`message`).
- **Variables (GET and after POST):**
  - `messages` — Collection of `GuestBook` with status `work`, email is
    masked (`str_mask_email`)
  - `pagination` — `{ count, page, offset }` (`page` = `guestbook_pagination`, default 10)

### 3.10 Reviews and questions — `Common\ReviewAction`

- **Routes (POST only, authenticated users only):**
  - `/publication/{uuid}/review` (`common:publication:review`)
  - `/catalog/{uuid}/review` (`common:catalog:review`)
  - `/catalog/{uuid}/question` (`common:catalog:question`)
- **Feature flags** (off by default, value `yes` enables):
  `review_publication_is_enabled`, `review_product_is_enabled`,
  `review_question_product_is_enabled`
- **Fields:** `message` (required), `rating` (0–5, for reviews), `parent_uuid`
  (a reply to an existing entry of the same entity), `files`
- **Response:** JSON. Success — `201 { description, data }`. The entry is
  created with status `moderate` (in templates show only `work`).
- No dedicated template — this is an API endpoint for the theme.

### 3.11 User — `Common\User\*`

Common base class `Common\User\UserAction` (services `Auth`, `UserService`,
`UserSubscriberService`). All pages are behind `IsRouteEnabledMiddleware`.

| Action | Route | Template (param / default) | Extra variables |
|---|---|---|---|
| `UserLoginAction` | `GET|POST /user/login` | `user_login_template` / `user.login.twig` | `identifier` (login field: `username`\|`email`\|`phone`, from `user_login_type`), `oauth` (Collection of OAuth plugins) |
| `UserRegisterAction` | `GET|POST /user/register` | `user_register_template` / `user.register.twig` | — (with e-mail confirmation enabled it renders `user_confirmation_template` / `user.confirmation.twig` with `stage='sent'`) |
| `UserProfileAction` | `GET|POST /user/profile` | `user_profile_template` / `user.profile.twig` | `user` — the `User` model. Without auth — redirect to `/user/login`. POST updates the profile (many fields, see the action), `files` for the avatar |
| `UserRecoveryAction` | `GET|POST /user/recovery` | `user_lostpassword_template` / `user.lostpassword.twig` | `stage` (`request`\|`sent`\|`reset`\|`expired`), `sent` (bool), `token` (on the `reset` step) |
| `UserConfirmationAction` | `GET|POST /user/confirm` | `user_confirmation_template` / `user.confirmation.twig` | `stage` (`resend`\|`sent`\|`expired`) |
| `UserSubscribeAction` | `GET|POST /user/subscriber` | — (JSON or redirect to the referer) | field `email`, `action` (`subscribe`\|`unsubscribe`) |
| `UserLogoutAction` | `GET|POST /user/logout` | — redirect to `/auth/logout` | — |
| `UserRevokeTokenAction` | `GET|POST /user/revoke` | — redirect to `/auth/revoke` | fields `uuid`, `redirect` |

Login/logout/refresh — through the `/auth/*` group (`RegisterAction`,
`LoginAction`, `RefreshTokenAction`, `RevokeTokenAction`, `LogoutAction`).
Tokens are set in httpOnly cookies `access_token` (`/`) and `refresh_token`
(`/auth`), `SameSite=Lax`.

### 3.12 Files — `Common\File\*`

| Action | Route | Purpose |
|---|---|---|
| `FileGetAction` | `GET /file/get/{salt}/{hash}` (`common:file:get`) | Serve a file as a download (`Content-Disposition: attachment`) |
| `FileViewAction` | `GET /file/view/{salt}/{hash}` (`common:file:view`) | Serve a file inline (images, etc.) |
| `FileUploadAction` | `POST /file/upload` (`common:file:upload`) | Upload (`file_is_enabled` flag), returns JSON |

In templates, direct links to uploaded files are easier to build via
`file.public_path('small'|'middle'|'big')` (see §4, the `File` model) — this
is a path like `/uploads/<salt>/<size>/<name>.<ext>`.

### 3.13 XML feeds — `Common\XMLFileAction`

- **Route:** `GET /xml/{name}` (`common:xml`) — serves `var/xml/{name}.xml`
  (for example `sitemap`, `yandex`, `google`). Generated by background tasks.

### 3.14 403 page — `Common\ForbiddenPageAction`

- **Route:** `GET|POST /forbidden` (`common:forbidden`) → `p403.twig`, status 403

### 3.15 REST API (for the theme's JS) — brief

`/api/v1/{entity}` (`Api\v1\EntityAction`) — CRUD over the entities from
`App\Domain\References\ApiEntity::MAP` (see §7), requires an API key
(`Authorization`). `GET /api/v1/search` — global search. `GET /api/v1/signature`,
`GET /api/v1/telemetry` — utility. The `parameter` entity is reachable only
through `/cup/api/v1/*` (admin panel), not the public API.

⚠️ `EntityAction` strips `with` from the query params before passing them to
`read()` — the theme cannot eager-load an arbitrary relation via the API.

---

## 4. Models (`src/Domain/Models`)

General: Eloquent outside Laravel, PK is `uuid` (`HasUuids`), JSON/enum
columns via `App\Domain\Casts\*`. In Twig both fields and public model
methods are available (`{{ product.price('price') }}`), as well as relations
(`{{ publication.category.title }}`). `toArray()` (used in JSON responses)
adds computed keys for many models — they are listed below as "in toArray".

> In `toArray()` the `files` key (and `relations`, `products`, …) holds
> **model collections**, not nested arrays — they turn into arrays only when
> the whole thing is JSON-encoded. Do not index them as arrays in PHP code.

> `$hidden` is set per model and is NOT inherited. Below it is noted what is
> hidden from serialization.

### User (`user`)
- **Fields:** `uuid`, `email`, `phone`, `username`, `firstname`, `lastname`,
  `patronymic`, `birthdate` (date), `gender`, `country`, `city`, `address`,
  `postcode`, `company` (array), `legal` (array), `messenger` (array),
  `website`, `source`, `loyalty` (array), `register`/`change` (datetime,
  = created/updated), `language`, `additional`, `group_uuid`,
  `is_allow_mail` (bool), `status` (`work`|`delete`|`block`|`confirmation`),
  `external_id`
- **Hidden:** `password`
- **Relations:** `group` (`UserGroup`), `tokens`, `integrations`, `orders`
  (last 10, by `serial desc`), `images`/`files` (`HasFiles` trait)
- **Methods for the theme:**
  - `name('full'|'name'|'initials'|'short')` — builds a full name, falling
    back to `username`/the local part of the email
  - `avatar(size=64, background, color)` — path to the uploaded avatar
    (`small`) or a ui-avatars.com link
- **in toArray:** `name.{full,short}`, `group`, `avatar`, `files`

### UserGroup (`user_group`)
- **Fields:** `uuid`, `title`, `description`, `access` (array)
- **Hidden:** `access`
- **Relations:** `users`

### UserToken (`user_token`) / UserIntegration (`user_integration`)
- Session tokens / OAuth bindings. Fields: `uuid`, `user_uuid`, `unique`,
  `comment`, `ip`, `agent`, `date` (+ `provider` for integration). Relation `user`.

### UserSubscriber (`user_subscriber`)
- `uuid`, `email`, `date` — newsletter subscribers.

### Page (`page`)
- **Fields:** `uuid`, `title`, `address` (AddressUrl), `date`, `content`
  (string), `type` (`html`|`text`), `meta` (array: `title`,`description`,`keywords`),
  `template` (theme twig file name)
- **Relations:** `files` (`HasFiles`)
- **in toArray:** `files`

### Publication (`publication`)
- **Fields:** `uuid`, `title`, `address`, `category_uuid`, `user_uuid`,
  `content` (array — blocks/HTML), `meta` (array), `date` (datetime),
  `external_id`
- **Relations:** `category` (`PublicationCategory`), `user` (`User`),
  `reviews` (top-level reviews, `entity_type=publication`, `type=review`, no
  replies), `files`
- **in toArray:** trimmed `category` (`uuid,title,address`), `user`
  (`uuid,name,avatar,external_id`), `files`

### PublicationCategory (`publication_category`)
- **Fields:** `uuid`, `title`, `address`, `parent_uuid`, `description`,
  `pagination` (int, default 10), `is_allow_nested` (bool), `is_public` (bool),
  `sort` (array: `by`, `direction`), `template` (array: `list`, `full`),
  `meta` (array)
- **Relations:** `parent`, `publications`, `files`
- **Methods:** `parents()` — Collection from the current one up to the root;
  `nested($force=false)` — the current one + nested (if `is_allow_nested`)
- **in toArray:** `parent` (`title,address`), `files`

### CatalogCategory (`catalog_category`)
- **Fields:** `uuid`, `title`, `description`, `address`, `parent_uuid`,
  `pagination` (int), `is_allow_nested` (bool), `is_hidden` (bool),
  `order` (int), `status` (`work`|`delete`), `sort` (array: `by`,`direction`),
  `meta` (array), `template` (array: `category`, `product`), `specifics` (array),
  `external_id`, `export`, `system`
- **Relations:** `parent`, `attributes` (BelongsToMany), `filtres()`
  (attributes with `is_filter=true`), `products` (only `status=work`), `files`
- **Methods:** `parents()`, `nested($force=false)` (respects `status=work`)
- **in toArray:** `parent`, `attributes` (keyBy `address`), `files`
- ⚠️ The relation is named `attributes` — it collides with an internal
  Eloquent property. In PHP read it via `getRelationValue('attributes')`; in
  Twig `category.attributes` works.

### CatalogProduct (`catalog_product`)
- **Fields:** `uuid`, `title`, `description`, `extra`, `address`, `type`
  (`product`|`service`), `category_uuid`, `vendorcode`, `barcode`,
  `tax` (float), `tax_included` (bool), `priceFirst`, `price`,
  `priceWholesale`, `priceWholesaleFrom`, `discount`, `special` (bool),
  `dimension` (array: `length`,`width`,`height`,`weight`,`length_class`,
  `weight_class`), `quantity`, `quantityMin`, `stock`, `country`,
  `manufacturer`, `tags` (array), `order` (int), `status` (`work`|`delete`),
  `date`, `meta` (array), `external_id`, `export`
- **Hidden:** `priceFirst`, `relations`
- **Relations:** `category`, `attributes` (BelongsToMany, pivot `value`),
  `relations()` (related products, pivot `count`, only `status=work`),
  `reviews()`, `questions()` (top-level, no replies), `files`
- **Methods for the theme:**
  - `price(type='price'|'price_first'|'price_wholesale', precision=0)` —
    price accounting for `discount` and `tax`
  - `tax(type, precision=0)` — tax amount
  - `price_lowest(days=30, precision=0)` — min sale price over a period (or `null`)
  - `attribute(address, default=null)` — the product's attribute value (typed)
  - `specification()` / `specificationWithClass()` — `L×W×H` (+ length class)
  - `weight()` / `weightWithClass()`
- **Stock status:** `stock_status_uuid` and the relation `stockStatus`
  (a `Reference` of type `stock_status`, `null` if none). In `toArray` the
  key `stock_status` (`uuid`, `title`) is present only when the relation was
  loaded.
- **in toArray:** `calculated.{price,price_wholesale}`, trimmed `category`,
  `attributes` (keyBy `address`), `relations` (depth 1), `files`
- In an order-line context `totalPrice/totalCount/totalSum/totalDiscount/
  totalTax` are available (they work via `pivot`).

### CatalogAttribute (`catalog_attribute`)
- **Fields:** `uuid`, `title`, `address`, `type` (`string`|`integer`|`float`|
  `boolean`), `group`, `is_filter` (bool)
- **Relations:** `categories`, `products`
- **Methods:** `values(?CatalogCategory)` — `Collection<value => count>` for a
  faceted filter; `value()` — typed value from the pivot
- **toArray:** `uuid,title,address,type,group,is_filter,value`

### CatalogOrder (`catalog_order`)
- **Fields:** `uuid`, `serial`, `user_uuid`, `status_uuid`, `payment_uuid`,
  `delivery` (array), `shipping` (datetime), `comment`, `phone`, `email`,
  `date`, `system`, `external_id`, `export`
- **Relations:** `products` (BelongsToMany, pivot `price, price_type, count,
  discount, tax, tax_included`), `status` (`Reference`), `payment` (`Reference`)
- **Methods:** `totalSum()`, `totalDiscount()`, `totalTax(precision=0)`
- **in toArray:** `status`, `payment`, `products` (keyBy uuid, with
  `price/discount/tax/count/amount/raw/files`), `calculated.{discount,tax,total}`

### File (`file`)
- **Fields:** `uuid`, `name`, `ext`, `type` (MIME), `size` (int), `salt`,
  `hash`, `date`. In the link pivot — `order`, `comment`.
- **Methods for the theme:**
  - `filename()` → `name.ext`
  - `public_path($size='')` → `/uploads/<salt>[/<size>]/<name>.<ext>`,
    `$size` ∈ `big|middle|small` (falls back to the original if the size is missing)
  - `size()` → human-readable size
  - `order()` / `comment()` — from the pivot
- **toArray:** `uuid,name,ext,type,size,salt,hash,date,order,comment,link,
  path.{big,middle,small}`

### Form (`form`) / FormData (`form_data`)
- **Form:** `uuid`, `title`, `address`, `template`/`templateFile` (email
  template), `recaptcha` (bool), `authorSend` (bool), `origin` (array, from
  strings split by `\n`), `mailto` (array), `duplicate` (URL). Relation
  `entries`.
- **FormData:** `uuid`, `form_uuid`, `data` (array — all submission fields),
  `message`, `date`. Relations `form`, `files`.

### GuestBook (`guestbook`)
- `uuid`, `name`, `email`, `message`, `response` (admin's reply), `status`
  (`work`|`moderate`), `date`.

### Review (`review`)
- **Fields:** `uuid`, `parent_uuid`, `user_uuid`, `type` (`review`|`question`),
  `entity_type` (`publication`|`catalog_product`), `entity_uuid`, `rating`
  (int 0–5), `message`, `status` (`work`|`moderate`), `date`
- **Hidden:** `user_uuid`
- **Relations:** `user`, `parent`, `children`
- **in toArray:** `user` (`uuid,name,avatar,external_id`), `files`, `children`
  (only `status=work`)

### Reference (`reference`) — reference books
- **Fields:** `uuid`, `type` (see the list of types in §7), `title`,
  `description` (text), `kind` (subtype within the type), `value` (array),
  `order` (int), `status` (bool)
- `kind` is validated per type: for `delivery` it is `pickup`, `courier` or
  `carrier` (`App\Domain\References\Delivery::KIND`), anything else is saved
  as `''`; other types have no kinds. `description` is free text for any type
  (the panel edits it for deliveries and payments).
- The `value` of a **delivery** is a list of `{uuid, condition}` pairs (service
  product + minimal cart total) — do not read `kind`/`description` from it.
- The `value` of a **requisite** record: `legal_name`, `tax_id`,
  `tax_reason`, `reg_id`, `address`, `director`, `bank`, `bank_id`, `account`,
  `corr_account`, `email`, `phone` (all optional strings).
- **Method:** `value($key=null, $default=null)` — the whole JSON or one key

### Parameter (`params`)
- `name` (PK), `value`. In templates — only via `parameter()` (§6), not the model.

### ApiKey (`api_key`), Task (`task`)
- Utility, not related to the theme. `ApiKey`: `title`, `scopes` (array
  `{read:[],write:[]}`), `is_full_access` (bool), `status` (`work`|`revoke`).
  `Task`: `title`, `action`, `progress` (float), `status`
  (`queue`|`work`|`done`|`fail`|`cancel`|`delete`), `params`, `output`.

---

## 5. Services (`src/Domain/Service`)

The only place for business logic and queries. All implement
`create/read/update/delete` (`AbstractService`). In templates they are called
not directly but via the Twig functions of §6, but it helps to know the
`read()` contract.

### `read(array $criteria)` — general rules (`AbstractService::buildQuery`)

- **Strict criteria** (the keys below for each service): scalar → `where =`,
  array → `whereIn`.
- **Scalar on a unique field** (`uuid`, `address`, `title`, ...) → fetches a
  single record + throws a `*NotFoundException` on a miss.
  **Wrapped in an array** (`['uuid' => [...]]`) → `whereIn` and always a
  Collection (possibly empty).
- **`search`** — non-strict, case-insensitive LIKE over the service's
  `static::$search_columns`.
- **`with`** — overrides relation eager-loading (`static::$eager`). This is
  an internal PHP knob, not a caller-facing filter.
- **`order`** (`{col: 'asc'|'desc'}`), **`limit`**, **`offset`**.

### `read()` criteria and eager per service

| Service | Strict `read()` criteria | `$search_columns` | `$eager` (loaded by default) |
|---|---|---|---|
| `Page\PageService` | `uuid, title, address, template, type` | `title, address` | `files` |
| `Publication\PublicationService` | `uuid, address, title, category_uuid, user_uuid, external_id` | `title, address` | `category, user, user.images, files` |
| `Publication\CategoryService` | `parent_uuid, uuid, title, address, parent, is_public` | `title, address` | `parent, files` |
| `Catalog\ProductService` | `uuid, category_uuid, title, type, address, vendorcode, barcode, special, status, external_id, export` | `title` | `category, attributes, files, relations(+category,attributes,files)` |
| `Catalog\CategoryService` | `parent_uuid, uuid, is_allow_nested, is_hidden, title, address, status, external_id, export` | `title, address` | `parent, attributes, files` |
| `Catalog\OrderService` | `uuid, user_uuid, serial, phone, email, status_uuid, payment_uuid, external_id, export` | `serial, phone, email` | `status, payment, products, products.files` |
| `Catalog\AttributeService` | `uuid, title, address, type, group, is_filter` | `title, address` | — |
| `GuestBook\GuestBookService` | `uuid, email, status` | `name, email, message` | — |
| `Review\ReviewService` | `uuid, parent_uuid, user_uuid, type, entity_type, entity_uuid, status` | `message` | `user` |
| `Reference\ReferenceService` | `uuid, title, type, status` | `title` | — |
| `User\UserService` | `identifier` (username/email/phone), `uuid, group_uuid, username, firstname, lastname, email, phone, country, city, postcode, additional, is_allow_mail, status, external_id, provider, unique` | `username, email, phone, firstname, lastname` | `group, files, images` |
| `User\GroupService` | `uuid, title` (see the service) | `title` | — |
| `User\SubscriberService` | `uuid, email` | `email` | — |
| `File\FileService` | `uuid, hash, name, ext, type, size` | `name` | — |
| `Parameter\ParameterService` | `name` | `name` | — (`/cup` only) |
| `Task\TaskService` | `uuid, ...` | `title` | — |
| `ApiKey\ApiKeyService` | `uuid, ...` | — | — |

---

## 6. Custom Twig functions, filters, tags

Everything is registered in `App\Application\TwigExtension`
(`src/Application/TwigExtension.php`); the extension name is `wse`.

### 6.1 Domain data functions (call services)

| Function | Signature | Returns |
|---|---|---|
| `page(criteria={}, order={'date':'desc'}, limit=10, offset=null)` | → `PageService->read` | Collection\|Page |
| `publication(criteria={}, order={}, limit=10, offset=null)` | → `PublicationService->read` | Collection\|Publication |
| `publication_category(public=true)` | publication categories (`is_public`) | Collection |
| `catalog_category(criteria={}, order={'order':'asc'})` | always `status=work`, `is_hidden=false` by default | Collection\|CatalogCategory |
| `catalog_product(criteria={}, order={'order':'asc'}, limit=50, offset=null)` | always `status=work` | Collection\|CatalogProduct |
| `catalog_product_count(criteria={})` | always `status=work` | int |
| `catalog_attribute(criteria={}, order={})` | catalog attributes | Collection |
| `catalog_order(criteria={}, order={'date':'desc'}, limit=10, offset=null)` | orders | Collection\|CatalogOrder |
| `catalog_category_tree(root=null)` | the shown categories as a tree; `root` is a category uuid or address (top level by default). Node: `category`, `uuid`, `title`, `address`, `image` (first image `File` or `null`), `own_count`, `count` (what the category page lists: nested products only where `is_allow_nested`), `children`. A hidden category takes its branch with it. Two queries for the whole tree | array |
| `catalog_product_bought_together(product, limit=4)` | products found in the same orders as `product` (model or uuid), the most frequent first, each with `together` (shared orders). Only `status=work`, `type=product`, shown categories; never the product itself. Empty for `null` | Collection |
| `catalog_product_popular(limit=10, type='product')` | top by number of orders | Collection |
| `catalog_product_view(uuid=null, limit=10)` | writes uuid into the "viewed" session list and/or returns the list | array |
| `catalog_product_price_type()` | `needs_context` — price type for the current user (`price` or from `catalog_price_type`) | string |
| `catalog_product_dimensional_weight(product=null)` | `needs_context` — volumetric weight (by `catalog_dimensional_weight`, default 5000) | float |
| `reference(type=null, pluck=false)` | reference book by type (`status=true`, `order asc`); without a type — the list of types; `pluck` → `value` keyBy `title` | Collection\|array |
| `guestbook(order={'date':'desc'}, limit=10, offset=null)` | entries with `status=work`, email masked | Collection |
| `user(criteria={}, order={'email':'asc'})` | users with `status=work` | Collection |
| `user_group(criteria={}, order={'title':'asc'})` | groups | Collection |
| `parameter(name=null, default=null)` | parameter value; an array of names → an array; without a name → all (keys `group[key]`) | mixed |

### 6.2 Routes and URLs

| Function | Purpose |
|---|---|
| `path_for(name, data={}, query={})` | path by Slim route name |
| `full_url_for(name, data={}, query={})` | absolute URL (host from `X-Forwarded-*` / `common_homepage` / the request) |
| `base_url()` | `common_homepage` without a trailing `/` |
| `is_current_path(name, data={})` | does the current path match the route? |
| `current_path(withQuery=false)` | the current path |
| `current_url()` | the full current URL |
| `parse_url()` | `parse_url(current_url())` |

Public route names are in §3 (in parentheses, e.g. `common:catalog:list`).

### 6.3 Pagination / query string

| Function | Purpose |
|---|---|
| `non_page_path()` | the current path without the trailing numeric segment (the page number) |
| `current_page_number()` | the page number from the tail of the path (0 if none) |
| `is_current_page_number(n)` | comparison with the current one |
| `current_query(key=null, value=null)` | the current query string (one key can be replaced), with a leading `?` |
| `build_query(url='', params={})` | build `url?a=b&...` (urldecoded) |

### 6.4 Utilities

| Function/filter | Purpose |
|---|---|
| `df(value='now', format=null)` (function and filter) | date formatting; default — `common_date_format` (`j-m-Y, H:i`) |
| `dfm(format=null)` | the same format, converted to moment.js format (for JS) |
| `trans(obj)` / `|trans` filter | from a multilingual array, take the value for the current locale |
| `_(key)` / `|locale` filter / `{% locale %}` tag | localization (i18n) |
| `count(obj)` (function and filter) | `count()` for a countable, otherwise `false` |
| `collect(array\|Collection)` | wrap in a Laravel Collection |
| `constant('App/Domain/.../Class::CONST'[, key])` | constant value (slashes → backslashes); array → `combine`; with a key → the element |
| `json_encode(v, flags=JSON_UNESCAPED_UNICODE, depth=512)` / `json_decode(json, assoc=true, ...)` | encode is_safe html |
| `base64_encode(s)` / `base64_decode(s, strict=false)` | |
| `convert_size(bytes)` | "12.3 mb" |
| `qr_code(value, size=256, margin=0)` | `<img>` with an inline SVG QR (is_safe html) |
| `preg_replace(subject, pattern, replacement)` (filter) | wrapper over `preg_replace` (arguments reordered!) |
| `cache_put(key, value, seconds, type='memory'|'file')` / `cache_get(key, default=null, type=...)` | cache for the render duration (`memory` = ArrayStore, lives one request; `file` = FileStore) |
| `pre(...args)` | dump for debugging |
| `form(type, name, args={})` (function and helper) | HTML5 field generator (`\Alksily\Support\Form`); picks up `_error` for highlighting |
| `file(criteria={}, order={'date':'desc'}, limit=10, offset=null)` | fetch files via `FileService` |
| `_cup_editor_locale()` | for the admin editor only |

### 6.5 Test

- `x is instanceof('App\\Domain\\Models\\CatalogProduct')` — type check.

---

## 7. Reference books and enum values

**Statuses** (cast constants, strings):
- Catalog (`CatalogCategory`, `CatalogProduct`): `work`, `delete`
- User: `work`, `delete`, `block`, `confirmation`
- Review / GuestBook: `work`, `moderate`
- ApiKey: `work`, `revoke`
- Task: `queue`, `work`, `done`, `fail`, `cancel`, `delete`

**Types:**
- Page: `html`, `text`
- CatalogProduct: `product`, `service`
- CatalogAttribute: `string`, `integer`, `float`, `boolean`
- Review type: `review`, `question`; entity_type: `publication`, `catalog_product`

**`Reference` types** (`reference(type)`): `text`, `store_location`,
`currency`, `order_status`, `stock_status`, `payment`, `delivery`, `country`,
`tax_rate`, `length_class`, `weight_class`, `address_format`,
`social_network`, `manufacturer`, `document`, `requisite`.

**Sorting:**
- Catalog: by ∈ `title|price|stock|date|order`, direction ∈ `ASC|DESC`
  (`App\Domain\References\Catalog`)
- Publication: by ∈ `title|date`, direction ∈ `ASC|DESC`
- Price types: `price`, `price_wholesale`, `price_self`

**Time constants** (`App\Domain\References\Date`): `MINUTE`, `HOUR`, `DAY`,
`WEEK`, `MONTH`, `YEAR`; formats `DATE`='Y-m-d', `DATETIME`='Y-m-d H:i:s',
`DATE_RUS`='d.m.Y', `DATETIME_RUS`='d.m.Y H:i'.

**API entities** (`ApiEntity::MAP`): `catalog/attributes`, `catalog/category`,
`catalog/product`, `catalog/order`, `file`, `guestbook`, `page`,
`publication`, `publication/category`, `reference`, `review`, `task`, `user`,
`user/group`. `/cup` only: `parameter`.

---

## 8. Parameters affecting templates (summary)

Set in the admin panel (`/cup/parameters`), read via
`parameter('name', default)`.

| Parameter | Default | Where used |
|---|---|---|
| `common_theme` | `default` | choice of the theme folder |
| `common_template` | `main.twig` | home page template |
| `common_title`, `common_description`, `common_keywords`, `common_homepage` | — | meta / canonical URL (used in theme templates) |
| `common_date_format` | `j-m-Y, H:i` | `df` / `dfm` |
| `common_auto_generate_address` | `no` | auto-generation of `address` |
| `search_template` | `search.twig` | search |
| `search_limit` | `10` | search |
| `guestbook_template` | `guestbook.twig` | guestbook |
| `guestbook_pagination` | `10` | guestbook |
| `catalog_category_template` | `catalog.category.twig` | catalog storefront |
| `catalog_category_pagination` | `10` | storefront |
| `catalog_cart_template` | `catalog.cart.twig` | cart |
| `catalog_cart_complete_template` | `catalog.cart.complete.twig` | order placed |
| `catalog_sort_by` / `catalog_sort_direction` | `title` / `ASC` | storefront sorting |
| `catalog_price_type` | `price` | price type for authenticated users |
| `catalog_dimensional_weight` | `5000` | volumetric weight |
| `catalog_order_fields` | `off` | concatenate extra order fields into `comment` |
| `catalog_order_address` | `off` | sort address parts |
| `catalog_order_limit` | `0` | order limit per delivery day |
| `user_login_template` | `user.login.twig` | login |
| `user_login_type` | `username` | identifier field (`username`/`email`/`phone`) |
| `user_register_template` | `user.register.twig` | registration |
| `user_profile_template` | `user.profile.twig` | profile |
| `user_lostpassword_template` | `user.lostpassword.twig` | recovery |
| `user_confirmation_template` | `user.confirmation.twig` | e-mail confirmation |
| `recaptcha_is_enabled` / `recaptcha_private` / `recaptcha_public` | `off` | reCAPTCHA check on all POST forms |
| `file_is_enabled` | `yes` | accept uploads |
| `review_publication_is_enabled`, `review_product_is_enabled`, `review_question_product_is_enabled` | `no` | reviews/questions |

Categories (`CatalogCategory.template`, `PublicationCategory.template`) —
JSON, override the file names for their modes: `{category, product}` and
`{list, full}` respectively.

---

## 9. Shared layout and mixins (`src/Template`)

A theme's `main.twig` usually does `{% extends 'layout.twig' %}`, and page
templates do `{% extends 'main.twig' %}`.

### `layout.twig` — the root skeleton
Blocks to override: `head`, `style`, `title` (default:
`title ~ ' | ' ~ parameter('common_title')`), `body`, `bodyinner`, `script`.
Includes `/assets/js/wse.js?v={{ SHA }}`. The `title` variable is available
(if the theme `{% set %}`s it).

### `src/Template/mixin/*` — reusable pieces (available to the theme via `{% include 'mixin/...' %}`)
- `catalog.twig` — client-side cart (localStorage), **no dependencies** (it
  used to need jQuery). `window.catalog` keeps the same options, selectors
  (`data-catalog-*`) and `cart*` methods. Events: the handlers
  `options.events['on:cart:add']` … and DOM events `event:catalog:<name>` on
  `window` (`cart:add`, `cart:update`, `cart:remove`, `cart:remove:all`,
  `cart:checkout:before|after`, `ready`) — data in `e.detail`, not as a second
  argument as with jQuery. `cartAddItemFromElement(element)` replaced
  `cartAddItemFromJQuery`; `cartCheckout()` marks bad required fields with
  `check-error` and returns `false` instead of throwing. `options.cart.url`
  is where the order is posted (the current path by default)
- `recaptcha.twig` — the reCAPTCHA widget include (if enabled)
- `img.twig` / `img-script.twig` / `picture.twig` — lazy/responsive images
- `gallery.twig` — an entity's image gallery (`files`/`images`)
- `form.twig` — form field markup
- `datatable.twig`, `quiz.twig`, `cookies.twig`, `adbd.twig` — extra widgets

A theme **cannot** override a `mixin/*` by name (the engine's folder is
searched first, §1) — copy the file under another name and include that.

### Error pages
`p400 p403 p404 p405 p500 p503` — the engine looks for them first in the
theme, then in `src/Template/errors/`. A theme should ship at least `p404`
and `p500` (in `theme/default` they are minimal).

---

## 10. Gotchas for themes

- **AJAX and `Accept`.** A request with `Accept: */*` gets JSON, not HTML
  (see §1). For HTML fragments send `Accept: text/html` explicitly.
- **A template error = 400 + `p400.twig`.** No partial output; the whole
  page is replaced. Debug with `pre()` and `?format=json`.
- **`status`/`is_hidden` are already filtered** in most Twig functions
  (`catalog_*`, `guestbook`, `user`), but **not** in the raw
  `Model::query()` calls inside the catalog actions — there the conditions
  are set manually.
- **Moderation.** `Review`/`GuestBook` are created with `status=moderate` —
  in lists show only `work` (in `Review.toArray` `children` is already
  filtered).
- **Each model's `$hidden` is its own.** Before outputting a field in the
  markup, check whether it is in `$hidden` (`password`, `priceFirst`,
  `relations`, `user_uuid` in Review, `access` in UserGroup).
- **Relation name collisions** (`attributes`, `relations`): in Twig
  `product.attributes` works, in PHP helpers use `getRelationValue(...)`.
- **The parameter cache** lives for the whole request (ArrayStore). A
  parameter change in the admin panel is visible on the next request.
- **Each request is a fresh PHP-FPM process.** There is no cross-request
  state; `cache_put(..., 'memory')` lives only for the current render.
- **Build file paths** via `file.public_path('small'|'middle'|'big')`, do not
  assemble `/uploads/...` by hand — a size may be missing (it falls back to
  the original).
- **reCAPTCHA.** With `recaptcha_is_enabled=on` every POST (form, guestbook,
  cart, login, review) requires a valid `recaptcha` in the body, otherwise
  `_error['grecaptcha']`.
- **Twig errors in DEBUG** (`DEBUG=1`, development only; production behaves
  as before). A PHP warning/notice raised while a template renders is turned
  into an error that names the template and the line (`... in "x.twig" at
  line 12`) instead of being printed into the HTML — typically inside an
  attribute, with status 200. And a template that extends itself, directly
  or through others, is reported at once with the whole chain and the files
  (`Template "a.twig" extends itself: a.twig (…) -> b.twig (…) -> a.twig (…)`)
  instead of a timeout that names nothing. Without DEBUG neither check runs.
- **Auth cookies and AJAX.** The `access_token` cookie (path `/`) lives a
  month but its JWT ten minutes; the `refresh_token` cookie is sent to
  `/auth/*` only. An expired token makes any request 308-redirect to
  `/auth/refresh-token?redirect=<path?query>`; the refresh answers with a 308
  back to that address (method and body kept, JSON or not). A refresh that
  fails clears both cookies and lets the request through as a guest.
  Two parallel requests with one expired token are fine: the replaced refresh
  token is accepted for 30 seconds more.
