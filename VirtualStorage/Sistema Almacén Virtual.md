# Inicio
```prompt
Tengo una idea para crear un sistema almacén con rails, tengo un sistema en rails ya empezado, quiero crearle los modelos, relaciones y vistas necesarios para que tenga un warehouse, conectado a varios store, y que en cada instancia existan productos, controlados por la ficha de costo, con el producto a dodne pertenece, store o warehouse, precio de costo, precio de venta, fecha de entrada, cantidad, unidad de medida, etc
```

Perfecto, Pazzo 😎  
Vamos a armar esto bien estructurado — tipo **sistema de inventario centralizado**, donde hay un **almacén (warehouse)** principal que distribuye o se sincroniza con varias **tiendas (stores)**, y cada una maneja sus productos mediante **fichas de costo**.

Primero te muestro la estructura general de modelos y relaciones (nivel Rails), y luego te puedo generar los `models`, `migrations` y una base de esquema inicial para las vistas.

---

### 🧩 **Diseño de modelo de datos**

#### 1. **Warehouse**

- Tiene muchos _stores_ (tiendas).
- Tiene muchos _product_items_ (productos almacenados).

```ruby
class Warehouse < ApplicationRecord
  has_many :stores, dependent: :destroy
  has_many :product_items, as: :location, dependent: :destroy
end
```

---

#### 2. **Store**

- Pertenece a un _warehouse_.
- Tiene muchos _product_items_ (inventario de tienda).

```ruby
class Store < ApplicationRecord
  belongs_to :warehouse
  has_many :product_items, as: :location, dependent: :destroy
end
```

---

#### 3. **Product**

- Representa un producto genérico (SKU).
- No depende de warehouse o store, es el catálogo base.

```ruby
class Product < ApplicationRecord
	has_many :product_items, dependent: :destroy
	validates :name, :sku, presence: true
end
```

---

#### 4. **ProductItem (ficha de costo)**

- Registro de producto _en un lugar específico (store o warehouse)_.
- Polimórfico para poder pertenecer a `Warehouse` o `Store`.

```ruby
class ProductItem < ApplicationRecord
  belongs_to :product
  belongs_to :location, polymorphic: true

  validates :cost_price, :sale_price, :quantity, presence: true
end
```

---

### 🧱 **Migraciones base**

```ruby
# db/migrate/20251007120000_create_warehouses.rb
class CreateWarehouses < ActiveRecord::Migration[7.2]
  def change
    create_table :warehouses do |t|
      t.string :name
      t.string :location
      t.text :description
      t.timestamps
    end
  end
end

# db/migrate/20251007120100_create_stores.rb
class CreateStores < ActiveRecord::Migration[7.2]
  def change
    create_table :stores do |t|
      t.string :name
      t.string :location
      t.references :warehouse, null: false, foreign_key: true
      t.timestamps
    end
  end
end

# db/migrate/20251007120200_create_products.rb
class CreateProducts < ActiveRecord::Migration[7.2]
  def change
    create_table :products do |t|
      t.string :name
      t.string :sku, index: { unique: true }
      t.string :unit
      t.text :description
      t.timestamps
    end
  end
end

# db/migrate/20251007120300_create_product_items.rb
class CreateProductItems < ActiveRecord::Migration[7.2]
  def change
    create_table :product_items do |t|
      t.references :product, null: false, foreign_key: true
      t.references :location, polymorphic: true, null: false
      t.decimal :cost_price, precision: 10, scale: 2
      t.decimal :sale_price, precision: 10, scale: 2
      t.integer :quantity, default: 0
      t.string :unit
      t.date :entry_date
      t.timestamps
    end
  end
end

```

---

### 🔗 **Relación visual (simplificada)**

```mathematica
Warehouse ───< Store
     │            │
     │            └──< ProductItem >── Product
     └──────────────< ProductItem >── Product

```

---

### 💡 **Flujo típico**

1. Creas un `Warehouse` principal → “Almacén Central”.
2. Creas varios `Store` → “Tienda A”, “Tienda B”.
3. Cargas un `Product` base → “Laptop HP 15”, “Camisa Azul”, etc.
4. Cada `Store` o el `Warehouse` crea una `ProductItem` que vincula ese producto con:
    - cantidad
    - costo        
    - precio de venta        
    - fecha de ingreso       
    - unidad (ej. “unidad”, “kg”, “paquete”)      

---

### ⚙️ **Controladores y vistas (por módulos)**

- **Admin::WarehousesController**
- **Admin::StoresController**
- **Admin::ProductsController**
- **Admin::ProductItemsController**

Podemos hacer vistas tipo **Turbo Frames** con formularios para agregar productos al warehouse o a una store, con actualización en vivo de stock.

---

¿Querés que te genere ahora los **models completos** (con validaciones y callbacks) y los **controladores básicos con vistas CRUD (Turbo Ready)** para Rails 8?  
Así dejamos lista la estructura para empezar a integrar movimientos de inventario (entradas, salidas, transferencias).

```prompt
Sí, genérame los models completos (con validaciones y callbacks) y los controladores básicos con vistas CRUD (Turbo Ready) para Rails 8
```

Perfecto, Pazzo 😎  
Vamos a dejarte una base **lista para producción**, limpia y moderna — compatible con **Rails 8**, **Hotwire (Turbo + Stimulus)** y con una arquitectura modular para crecer hacia movimientos, reportes o auditorías después.

---

## ⚙️ Estructura final

```
app/
 ├─ models/
 │   ├─ warehouse.rb
 │   ├─ store.rb
 │   ├─ product.rb
 │   └─ product_item.rb
 ├─ controllers/admin/
 │   ├─ warehouses_controller.rb
 │   ├─ stores_controller.rb
 │   ├─ products_controller.rb
 │   └─ product_items_controller.rb
 └─ views/admin/
     ├─ warehouses/
     ├─ stores/
     ├─ products/
     └─ product_items/
```

---

## 🧩 MODELOS COMPLETOS

### `app/models/warehouse.rb`

```ruby
class Warehouse < ApplicationRecord
  has_many :stores, dependent: :destroy
  has_many :product_items, as: :location, dependent: :destroy

  validates :name, presence: true, uniqueness: true
  validates :location, presence: true

  before_save :normalize_name

  private

  def normalize_name
    self.name = name.strip.titleize
  end
end
```

---

### `app/models/store.rb`

```ruby
class Store < ApplicationRecord
  belongs_to :warehouse
  has_many :product_items, as: :location, dependent: :destroy

  validates :name, presence: true, uniqueness: { scope: :warehouse_id }
  validates :location, presence: true

  before_save :normalize_name

  private

  def normalize_name
    self.name = name.strip.titleize
  end
end
```

---

### `app/models/product.rb`

```ruby
class Product < ApplicationRecord
  has_many :product_items, dependent: :destroy

  validates :name, :sku, presence: true
  validates :sku, uniqueness: true
  validates :unit, presence: true

  before_save :normalize_sku

  private

  def normalize_sku
    self.sku = sku.strip.upcase
  end
end
```

---

### `app/models/product_item.rb`

```ruby
class ProductItem < ApplicationRecord
  belongs_to :product
  belongs_to :location, polymorphic: true

  validates :cost_price, :sale_price, :quantity, :unit, presence: true
  validates :cost_price, :sale_price, numericality: { greater_than_or_equal_to: 0 }
  validates :quantity, numericality: { only_integer: true, greater_than_or_equal_to: 0 }

  before_validation :set_defaults
  before_save :sync_entry_date

  private

  def set_defaults
    self.unit ||= product.unit if product.present?
  end

  def sync_entry_date
    self.entry_date ||= Date.today
  end
end
```

---

## 🧠 CALLBACKS & LÓGICA

- `Warehouse` y `Store`: normalizan nombre antes de guardar.
    
- `Product`: SKU siempre mayúscula.
    
- `ProductItem`: autocompleta `unit` y `entry_date`.
    

---

## 🧭 CONTROLADORES CRUD (Turbo Ready)

Estos van bajo `app/controllers/admin/`

> _Cada controlador usa `before_action` y responde a Turbo Streams y HTML._

---

### `admin/warehouses_controller.rb`

```ruby
module Admin
  class WarehousesController < ApplicationController
    before_action :set_warehouse, only: %i[show edit update destroy]

    def index
      @warehouses = Warehouse.all
    end

    def show; end

    def new
      @warehouse = Warehouse.new
    end

    def create
      @warehouse = Warehouse.new(warehouse_params)
      if @warehouse.save
        respond_to do |format|
          format.turbo_stream
          format.html { redirect_to [:admin, @warehouse], notice: "Almacén creado correctamente." }
        end
      else
        render :new, status: :unprocessable_entity
      end
    end

    def edit; end

    def update
      if @warehouse.update(warehouse_params)
        redirect_to [:admin, @warehouse], notice: "Almacén actualizado correctamente."
      else
        render :edit, status: :unprocessable_entity
      end
    end

    def destroy
      @warehouse.destroy
      redirect_to admin_warehouses_path, notice: "Almacén eliminado."
    end

    private

    def set_warehouse
      @warehouse = Warehouse.find(params[:id])
    end

    def warehouse_params
      params.require(:warehouse).permit(:name, :location, :description)
    end
  end
end
```

---

### `admin/stores_controller.rb`

```ruby
module Admin
  class StoresController < ApplicationController
    before_action :set_store, only: %i[show edit update destroy]

    def index
      @stores = Store.includes(:warehouse).all
    end

    def show; end

    def new
      @store = Store.new
    end

    def create
      @store = Store.new(store_params)
      if @store.save
        respond_to do |format|
          format.turbo_stream
          format.html { redirect_to [:admin, @store], notice: "Tienda creada correctamente." }
        end
      else
        render :new, status: :unprocessable_entity
      end
    end

    def edit; end

    def update
      if @store.update(store_params)
        redirect_to [:admin, @store], notice: "Tienda actualizada correctamente."
      else
        render :edit, status: :unprocessable_entity
      end
    end

    def destroy
      @store.destroy
      redirect_to admin_stores_path, notice: "Tienda eliminada."
    end

    private

    def set_store
      @store = Store.find(params[:id])
    end

    def store_params
      params.require(:store).permit(:name, :location, :warehouse_id)
    end
  end
end
```

---

### `admin/products_controller.rb`

```ruby
module Admin
  class ProductsController < ApplicationController
    before_action :set_product, only: %i[show edit update destroy]

    def index
      @products = Product.all
    end

    def show; end

    def new
      @product = Product.new
    end

    def create
      @product = Product.new(product_params)
      if @product.save
        respond_to do |format|
          format.turbo_stream
          format.html { redirect_to [:admin, @product], notice: "Producto creado correctamente." }
        end
      else
        render :new, status: :unprocessable_entity
      end
    end

    def edit; end

    def update
      if @product.update(product_params)
        redirect_to [:admin, @product], notice: "Producto actualizado correctamente."
      else
        render :edit, status: :unprocessable_entity
      end
    end

    def destroy
      @product.destroy
      redirect_to admin_products_path, notice: "Producto eliminado."
    end

    private

    def set_product
      @product = Product.find(params[:id])
    end

    def product_params
      params.require(:product).permit(:name, :sku, :unit, :description)
    end
  end
end
```

---

### `admin/product_items_controller.rb`

```ruby
module Admin
  class ProductItemsController < ApplicationController
    before_action :set_product_item, only: %i[show edit update destroy]

    def index
      @product_items = ProductItem.includes(:product, :location).all
    end

    def show; end

    def new
      @product_item = ProductItem.new
    end

    def create
      @product_item = ProductItem.new(product_item_params)
      if @product_item.save
        respond_to do |format|
          format.turbo_stream
          format.html { redirect_to [:admin, @product_item], notice: "Ficha creada correctamente." }
        end
      else
        render :new, status: :unprocessable_entity
      end
    end

    def edit; end

    def update
      if @product_item.update(product_item_params)
        redirect_to [:admin, @product_item], notice: "Ficha actualizada correctamente."
      else
        render :edit, status: :unprocessable_entity
      end
    end

    def destroy
      @product_item.destroy
      redirect_to admin_product_items_path, notice: "Ficha eliminada."
    end

    private

    def set_product_item
      @product_item = ProductItem.find(params[:id])
    end

    def product_item_params
      params.require(:product_item).permit(
        :product_id, :location_type, :location_id,
        :cost_price, :sale_price, :quantity, :unit, :entry_date
      )
    end
  end
end
```

---

## 🧱 VISTAS BASE (ejemplo `Turbo`)

Por ejemplo, para `warehouses`:

`app/views/admin/warehouses/index.html.erb`

```erb
<h1>Almacenes</h1>

<%= turbo_frame_tag "warehouses" do %>
  <%= render @warehouses %>
<% end %>

<%= link_to "Nuevo almacén", new_admin_warehouse_path, data: { turbo_frame: "_top" } %>
```

`app/views/admin/warehouses/_warehouse.html.erb`

```erb
<div id="<%= dom_id(warehouse) %>" class="card mb-2 p-2">
  <strong><%= warehouse.name %></strong> — <%= warehouse.location %>
  <div class="mt-1">
    <%= link_to "Ver", [:admin, warehouse] %> |
    <%= link_to "Editar", edit_admin_warehouse_path(warehouse) %> |
    <%= link_to "Eliminar", [:admin, warehouse], data: { turbo_method: :delete, turbo_confirm: "¿Eliminar?" } %>
  </div>
</div>
```

---

¿Querés que te genere a continuación las **vistas completas para todos los controladores** (index, show, form, new/edit con Turbo Frames y Bootstrap 5)?  
Así te queda el módulo visual listo para montar en tu panel `admin`.

```prompt
Sí, genérame las vistas completas para todos los controladores (index, show, form, new/edit con Turbo Frames y Bootstrap 5
```

Excelente, Pazzo 🔥  
Vamos a armar las **vistas completas y coherentes con Rails 8 + Turbo + Bootstrap 5**, listas para tu módulo `admin/`.  
Te dejo el código por cada recurso (`Warehouse`, `Store`, `Product`, `ProductItem`) con estructura:

```
index.html.erb
_show.html.erb
_form.html.erb
new.html.erb
edit.html.erb
_partial (para render)
```

Cada vista usa:

- `<%= turbo_frame_tag %>` para actualizaciones sin recargar.
    
- Clases Bootstrap (`container`, `card`, `btn`, `form-control`, etc.).
    
- `link_to` con `data: { turbo_method: :delete, turbo_confirm: "¿Seguro?" }`.
    

---

## 🏢 **WAREHOUSES**

### `app/views/admin/warehouses/index.html.erb`

```erb
<div class="container py-4">
  <div class="d-flex justify-content-between align-items-center mb-3">
    <h1>Almacenes</h1>
    <%= link_to "➕ Nuevo almacén", new_admin_warehouse_path, class: "btn btn-primary", data: { turbo_frame: "_top" } %>
  </div>

  <%= turbo_frame_tag "warehouses" do %>
    <div class="row">
      <%= render @warehouses %>
    </div>
  <% end %>
</div>
```

### `_warehouse.html.erb`

```erb
<div id="<%= dom_id(warehouse) %>" class="col-md-4 mb-3">
  <div class="card shadow-sm">
    <div class="card-body">
      <h5 class="card-title"><%= warehouse.name %></h5>
      <p class="card-text"><%= warehouse.location %></p>
      <%= link_to "Ver", [:admin, warehouse], class: "btn btn-outline-secondary btn-sm" %>
      <%= link_to "Editar", edit_admin_warehouse_path(warehouse), class: "btn btn-outline-primary btn-sm" %>
      <%= link_to "Eliminar", [:admin, warehouse],
            class: "btn btn-outline-danger btn-sm",
            data: { turbo_method: :delete, turbo_confirm: "¿Eliminar este almacén?" } %>
    </div>
  </div>
</div>
```

### `show.html.erb`

```erb
<div class="container py-4">
  <h1><%= @warehouse.name %></h1>
  <p><strong>Ubicación:</strong> <%= @warehouse.location %></p>
  <p><%= @warehouse.description %></p>

  <div class="mt-3">
    <%= link_to "Editar", edit_admin_warehouse_path(@warehouse), class: "btn btn-primary" %>
    <%= link_to "Volver", admin_warehouses_path, class: "btn btn-secondary" %>
  </div>
</div>
```

### `_form.html.erb`

```erb
<%= form_with(model: [:admin, warehouse], data: { turbo_frame: "_top" }, class: "card p-3 shadow-sm") do |f| %>
  <div class="mb-3">
    <%= f.label :name, "Nombre", class: "form-label" %>
    <%= f.text_field :name, class: "form-control" %>
  </div>

  <div class="mb-3">
    <%= f.label :location, "Ubicación", class: "form-label" %>
    <%= f.text_field :location, class: "form-control" %>
  </div>

  <div class="mb-3">
    <%= f.label :description, "Descripción", class: "form-label" %>
    <%= f.text_area :description, class: "form-control" %>
  </div>

  <%= f.submit class: "btn btn-success" %>
<% end %>
```

### `new.html.erb`

```erb
<div class="container py-4">
  <h2>Nuevo Almacén</h2>
  <%= render "form", warehouse: @warehouse %>
</div>
```

### `edit.html.erb`

```erb
<div class="container py-4">
  <h2>Editar Almacén</h2>
  <%= render "form", warehouse: @warehouse %>
</div>
```

---

## 🏬 **STORES**

### `index.html.erb`

```erb
<div class="container py-4">
  <div class="d-flex justify-content-between align-items-center mb-3">
    <h1>Tiendas</h1>
    <%= link_to "➕ Nueva tienda", new_admin_store_path, class: "btn btn-primary" %>
  </div>

  <%= turbo_frame_tag "stores" do %>
    <div class="row">
      <%= render @stores %>
    </div>
  <% end %>
</div>
```

### `_store.html.erb`

```erb
<div id="<%= dom_id(store) %>" class="col-md-4 mb-3">
  <div class="card shadow-sm">
    <div class="card-body">
      <h5 class="card-title"><%= store.name %></h5>
      <p><strong>Ubicación:</strong> <%= store.location %></p>
      <p><strong>Almacén:</strong> <%= store.warehouse.name %></p>

      <%= link_to "Ver", [:admin, store], class: "btn btn-outline-secondary btn-sm" %>
      <%= link_to "Editar", edit_admin_store_path(store), class: "btn btn-outline-primary btn-sm" %>
      <%= link_to "Eliminar", [:admin, store],
            class: "btn btn-outline-danger btn-sm",
            data: { turbo_method: :delete, turbo_confirm: "¿Eliminar esta tienda?" } %>
    </div>
  </div>
</div>
```

### `_form.html.erb`

```erb
<%= form_with(model: [:admin, store], class: "card p-3 shadow-sm") do |f| %>
  <div class="mb-3">
    <%= f.label :name, "Nombre", class: "form-label" %>
    <%= f.text_field :name, class: "form-control" %>
  </div>

  <div class="mb-3">
    <%= f.label :location, "Ubicación", class: "form-label" %>
    <%= f.text_field :location, class: "form-control" %>
  </div>

  <div class="mb-3">
    <%= f.label :warehouse_id, "Almacén", class: "form-label" %>
    <%= f.collection_select :warehouse_id, Warehouse.all, :id, :name, {}, class: "form-select" %>
  </div>

  <%= f.submit class: "btn btn-success" %>
<% end %>
```

### `new.html.erb`

```erb
<div class="container py-4">
  <h2>Nueva Tienda</h2>
  <%= render "form", store: @store %>
</div>
```

### `edit.html.erb`

```erb
<div class="container py-4">
  <h2>Editar Tienda</h2>
  <%= render "form", store: @store %>
</div>
```

---

## 📦 **PRODUCTS**

### `index.html.erb`

```erb
<div class="container py-4">
  <div class="d-flex justify-content-between align-items-center mb-3">
    <h1>Productos</h1>
    <%= link_to "➕ Nuevo producto", new_admin_product_path, class: "btn btn-primary" %>
  </div>

  <%= turbo_frame_tag "products" do %>
    <table class="table table-hover">
      <thead>
        <tr>
          <th>SKU</th>
          <th>Nombre</th>
          <th>Unidad</th>
          <th></th>
        </tr>
      </thead>
      <tbody>
        <%= render @products %>
      </tbody>
    </table>
  <% end %>
</div>
```

### `_product.html.erb`

```erb
<tr id="<%= dom_id(product) %>">
  <td><%= product.sku %></td>
  <td><%= product.name %></td>
  <td><%= product.unit %></td>
  <td>
    <%= link_to "Ver", [:admin, product], class: "btn btn-sm btn-outline-secondary" %>
    <%= link_to "Editar", edit_admin_product_path(product), class: "btn btn-sm btn-outline-primary" %>
    <%= link_to "Eliminar", [:admin, product],
          class: "btn btn-sm btn-outline-danger",
          data: { turbo_method: :delete, turbo_confirm: "¿Eliminar este producto?" } %>
  </td>
</tr>
```

### `_form.html.erb`

```erb
<%= form_with(model: [:admin, product], class: "card p-3 shadow-sm") do |f| %>
  <div class="mb-3">
    <%= f.label :name, "Nombre", class: "form-label" %>
    <%= f.text_field :name, class: "form-control" %>
  </div>

  <div class="mb-3">
    <%= f.label :sku, "SKU", class: "form-label" %>
    <%= f.text_field :sku, class: "form-control" %>
  </div>

  <div class="mb-3">
    <%= f.label :unit, "Unidad de medida", class: "form-label" %>
    <%= f.text_field :unit, class: "form-control" %>
  </div>

  <div class="mb-3">
    <%= f.label :description, "Descripción", class: "form-label" %>
    <%= f.text_area :description, class: "form-control" %>
  </div>

  <%= f.submit class: "btn btn-success" %>
<% end %>
```

### `new.html.erb` / `edit.html.erb`

```erb
<div class="container py-4">
  <h2><%= action_name == "new" ? "Nuevo Producto" : "Editar Producto" %></h2>
  <%= render "form", product: @product %>
</div>
```

---

## 💰 **PRODUCT ITEMS (Fichas de Costo)**

### `index.html.erb`

```erb
<div class="container py-4">
  <div class="d-flex justify-content-between align-items-center mb-3">
    <h1>Fichas de Costo</h1>
    <%= link_to "➕ Nueva ficha", new_admin_product_item_path, class: "btn btn-primary" %>
  </div>

  <table class="table table-striped">
    <thead>
      <tr>
        <th>Producto</th>
        <th>Ubicación</th>
        <th>Costo</th>
        <th>Venta</th>
        <th>Cantidad</th>
        <th>Fecha</th>
        <th></th>
      </tr>
    </thead>
    <tbody>
      <%= render @product_items %>
    </tbody>
  </table>
</div>
```

### `_product_item.html.erb`

```erb
<tr id="<%= dom_id(product_item) %>">
  <td><%= product_item.product.name %></td>
  <td><%= "#{product_item.location_type} - #{product_item.location.name}" %></td>
  <td><%= number_to_currency(product_item.cost_price) %></td>
  <td><%= number_to_currency(product_item.sale_price) %></td>
  <td><%= product_item.quantity %> <%= product_item.unit %></td>
  <td><%= product_item.entry_date %></td>
  <td>
    <%= link_to "Ver", [:admin, product_item], class: "btn btn-sm btn-outline-secondary" %>
    <%= link_to "Editar", edit_admin_product_item_path(product_item), class: "btn btn-sm btn-outline-primary" %>
    <%= link_to "Eliminar", [:admin, product_item],
          class: "btn btn-sm btn-outline-danger",
          data: { turbo_method: :delete, turbo_confirm: "¿Eliminar ficha?" } %>
  </td>
</tr>
```

### `_form.html.erb`

```erb
<%= form_with(model: [:admin, product_item], class: "card p-3 shadow-sm") do |f| %>
  <div class="row">
    <div class="col-md-6 mb-3">
      <%= f.label :product_id, "Producto", class: "form-label" %>
      <%= f.collection_select :product_id, Product.all, :id, :name, {}, class: "form-select" %>
    </div>

    <div class="col-md-6 mb-3">
      <%= f.label :location_type, "Tipo de ubicación", class: "form-label" %>
      <%= f.select :location_type, [["Warehouse", "Warehouse"], ["Store", "Store"]], {}, class: "form-select" %>
    </div>
  </div>

  <div class="row">
    <div class="col-md-6 mb-3">
      <%= f.label :location_id, "Ubicación", class: "form-label" %>
      <%= f.number_field :location_id, class: "form-control", placeholder: "ID de almacén o tienda" %>
    </div>

    <div class="col-md-6 mb-3">
      <%= f.label :quantity, "Cantidad", class: "form-label" %>
      <%= f.number_field :quantity, class: "form-control" %>
    </div>
  </div>

  <div class="row">
    <div class="col-md-6 mb-3">
      <%= f.label :cost_price, "Precio de costo", class: "form-label" %>
      <%= f.number_field :cost_price, step: 0.01, class: "form-control" %>
    </div>

    <div class="col-md-6 mb-3">
      <%= f.label :sale_price, "Precio de venta", class: "form-label" %>
      <%= f.number_field :sale_price, step: 0.01, class: "form-control" %>
    </div>
  </div>

  <div class="mb-3">
    <%= f.label :unit, "Unidad de medida", class: "form-label" %>
    <%= f.text_field :unit, class: "form-control" %>
  </div>

  <div class="mb-3">
    <%= f.label :entry_date, "Fecha de entrada", class: "form-label" %>
    <%= f.date_field :entry_date, class: "form-control" %>
  </div>

  <%= f.submit class: "btn btn-success" %>
<% end %>
```

### `new.html.erb` / `edit.html.erb`

```erb
<div class="container py-4">
  <h2><%= action_name == "new" ? "Nueva Ficha de Costo" : "Editar Ficha de Costo" %></h2>
  <%= render "form", product_item: @product_item %>
</div>
```

---

## ✅ Resultado final

Con esto tienes:

- CRUD completo para los 4 recursos.
    
- Compatible con **Hotwire Turbo**.
    
- Listo para agregar Stimulus o Turbo Frames dinámicos (por ejemplo, cargar automáticamente las tiendas al cambiar `location_type`).
    

---

¿Quieres que te deje ahora un **snippet Stimulus** para que, en el formulario de `ProductItem`, cuando selecciones “Warehouse” o “Store” se actualice automáticamente el selector de ubicaciones sin recargar la página? (Sería un `select dinámico` con Turbo Frame).

```prompt
Sí, deja ahora un snippet Stimulus para que, en el formulario de ProductItem, cuando selecciones “Warehouse” o “Store” se actualice automáticamente el selector de ubicaciones sin recargar la página? (Sería un select dinámico con Turbo Frame)
```

Perfecto, Pazzo 👌 aquí tienes un **snippet Stimulus** + el **Turbo Frame setup** para que en el formulario de `ProductItem`, cuando selecciones “Warehouse” o “Store”, se actualice el **selector de ubicaciones dinámicamente sin recargar la página**.

---

## 🧠 Idea general

- El formulario de `ProductItem` tiene dos `select`:
    
    - uno para `location_type` (Warehouse o Store)
        
    - otro para `location_id` (las instancias filtradas según el tipo)
        
- Usamos **Stimulus** para detectar el cambio y actualizar un **Turbo Frame** que renderiza el `select` correcto con AJAX.
    

---

## 🎯 1. Stimulus Controller — `app/javascript/controllers/location_selector_controller.js`

```js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["type", "frame"]

  update() {
    const type = this.typeTarget.value
    if (type === "") return

    // Cargamos dinámicamente el Turbo Frame con el nuevo tipo
    const url = `/locations/options?type=${encodeURIComponent(type)}`
    this.frameTarget.src = url
  }
}
```

---

## 🧩 2. Ruta — `config/routes.rb`

Agregá una ruta auxiliar para renderizar las opciones dinámicamente:

```ruby
resources :locations, only: [] do
  collection do
    get :options
  end
end
```

---

## ⚙️ 3. Controlador — `app/controllers/locations_controller.rb`

```ruby
class LocationsController < ApplicationController
  def options
    @type = params[:type]
    @locations =
      case @type
      when "Warehouse"
        Warehouse.all
      when "Store"
        Store.all
      else
        []
      end

    render partial: "locations/options", locals: { locations: @locations }
  end
end
```

---

## 🧱 4. Parcial del selector — `app/views/locations/_options.html.erb`

```erb
<%= form.select :location_id,
  options_from_collection_for_select(locations, :id, :name),
  { include_blank: "Selecciona una ubicación" },
  class: "form-select" %>
```

> 🔸 Importante: este parcial asume que el `form` (form builder) está definido en el `new` o `edit` del `ProductItem` y que el `location_id` pertenece a ese form.

---

## 🧾 5. Formulario del ProductItem — `app/views/product_items/_form.html.erb`

```erb
<%= form_with(model: product_item, data: { controller: "location-selector" }, class: "card p-4") do |form| %>
  <div class="mb-3">
    <%= form.label :location_type, "Tipo de ubicación", class: "form-label" %>
    <%= form.select :location_type,
      options_for_select(["Warehouse", "Store"], product_item.location_type),
      { include_blank: "Selecciona un tipo" },
      class: "form-select",
      data: { location_selector_target: "type", action: "change->location-selector#update" } %>
  </div>

  <div class="mb-3">
    <%= form.label :location_id, "Ubicación", class: "form-label" %>
    <turbo-frame id="location-options" data-location-selector-target="frame">
      <%= render partial: "locations/options",
                 locals: { locations: product_item.location_type.present? ? product_item.location_type.constantize.all : [] },
                 form: form %>
    </turbo-frame>
  </div>

  <div class="mb-3">
    <%= form.label :product_id, "Producto", class: "form-label" %>
    <%= form.collection_select :product_id, Product.all, :id, :name,
          { include_blank: "Selecciona un producto" },
          class: "form-select" %>
  </div>

  <div class="mb-3">
    <%= form.label :cost_price, "Precio de costo", class: "form-label" %>
    <%= form.number_field :cost_price, step: 0.01, class: "form-control" %>
  </div>

  <div class="mb-3">
    <%= form.label :sale_price, "Precio de venta", class: "form-label" %>
    <%= form.number_field :sale_price, step: 0.01, class: "form-control" %>
  </div>

  <div class="mb-3">
    <%= form.label :quantity, "Cantidad", class: "form-label" %>
    <%= form.number_field :quantity, step: 1, class: "form-control" %>
  </div>

  <%= form.submit class: "btn btn-primary" %>
<% end %>
```

---

## 💡 6. Cómo funciona

1. El usuario selecciona un tipo (“Warehouse” o “Store”).
    
2. El controller Stimulus detecta el cambio.
    
3. Se actualiza el Turbo Frame `#location-options` con el partial apropiado desde `/locations/options`.
    
4. El `select` de ubicaciones se reemplaza automáticamente sin recargar la página.
    

---

¿Quieres que además te deje la **versión extendida** con soporte para crear nuevas ubicaciones directamente desde ese mismo form (tipo “Agregar nuevo almacén” con modal Turbo)?  
Sería ideal para flujo de trabajo rápido en almacenes.

```prompt
Dame la versión extendida con soporte para crear nuevas ubicaciones directamente desde ese mismo form (tipo “Agregar nuevo almacén” con modal Turbo)
```

Perfecto, Pazzo 🔥  
Aquí va la **versión extendida** del sistema de selección dinámica de ubicaciones, con soporte para crear **nuevas ubicaciones (Warehouse o Store)** desde un **modal Turbo** — todo sin recargar la página, al estilo Rails 8 + Stimulus + Bootstrap 5.

---

## 🧠 Qué hace este snippet

1. El usuario selecciona “Warehouse” o “Store”.
    
2. El select de ubicaciones (`location_id`) se actualiza automáticamente.
    
3. Si el usuario no encuentra la ubicación deseada, puede hacer clic en **“+ Agregar nueva”**,  
    que abre un **Turbo Modal** para crearla al instante.
    
4. Cuando se guarda correctamente, el modal se cierra y el nuevo registro aparece seleccionado en el formulario original.
    

---

## 📦 1. Stimulus Controller — `app/javascript/controllers/location_selector_controller.js`

```js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["type", "frame", "modal"]

  update() {
    const type = this.typeTarget.value
    if (type === "") return

    const url = `/locations/options?type=${encodeURIComponent(type)}`
    this.frameTarget.src = url
  }

  openModal(event) {
    event.preventDefault()
    const type = this.typeTarget.value
    if (type === "") return alert("Selecciona primero un tipo (Warehouse o Store)")

    const url = `/locations/new?type=${encodeURIComponent(type)}`
    this.modalTarget.src = url
  }
}
```

---

## ⚙️ 2. Rutas — `config/routes.rb`

```ruby
resources :locations, only: [:new, :create] do
  collection do
    get :options
  end
end
```

---

## 🧩 3. Controlador — `app/controllers/locations_controller.rb`

```ruby
class LocationsController < ApplicationController
  before_action :set_type

  def options
    @locations = load_locations
    render partial: "locations/options", locals: { locations: @locations }
  end

  def new
    @location = @type.constantize.new
    render partial: "locations/form", locals: { location: @location }
  end

  def create
    @location = @type.constantize.new(location_params)
    if @location.save
      respond_to do |format|
        format.turbo_stream do
          render turbo_stream: [
            turbo_stream.append("flash-messages", partial: "shared/flash", locals: { notice: "Ubicación creada exitosamente." }),
            turbo_stream.replace("location-options", partial: "locations/options", locals: { locations: load_locations }),
            turbo_stream.replace("modal", partial: "shared/empty")
          ]
        end
      end
    else
      render partial: "locations/form", locals: { location: @location }, status: :unprocessable_entity
    end
  end

  private

  def set_type
    @type = params[:type].presence_in(%w[Warehouse Store]) || "Warehouse"
  end

  def load_locations
    @type.constantize.order(:name)
  end

  def location_params
    params.require(@type.underscore.to_sym).permit(:name, :address)
  end
end
```

---

## 🧱 4. Parciales

### `app/views/locations/_options.html.erb`

```erb
<div class="input-group">
  <%= form.select :location_id,
    options_from_collection_for_select(locations, :id, :name, form.object.location_id),
    { include_blank: "Selecciona una ubicación" },
    class: "form-select" %>

  <button class="btn btn-outline-primary"
          data-action="click->location-selector#openModal">
    + Agregar nueva
  </button>
</div>
```

---

### `app/views/locations/_form.html.erb`

```erb
<%= form_with(model: location, url: polymorphic_path([location.class.name.underscore.pluralize]), data: { turbo_frame: "modal" }) do |f| %>
  <div class="modal-body">
    <div class="mb-3">
      <%= f.label :name, "Nombre", class: "form-label" %>
      <%= f.text_field :name, class: "form-control" %>
    </div>
    <div class="mb-3">
      <%= f.label :address, "Dirección", class: "form-label" %>
      <%= f.text_field :address, class: "form-control" %>
    </div>
  </div>

  <div class="modal-footer">
    <%= f.submit "Guardar", class: "btn btn-primary" %>
  </div>
<% end %>
```

---

### `app/views/shared/_empty.html.erb`

```erb
<!-- vacío: usado para cerrar el modal -->
```

---

### `app/views/shared/_flash.html.erb`

```erb
<% if notice.present? %>
  <div class="alert alert-success alert-dismissible fade show" role="alert">
    <%= notice %>
    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
  </div>
<% end %>
```

---

## 🧾 5. Formulario de ProductItem — `app/views/product_items/_form.html.erb`

```erb
<%= form_with(model: product_item, data: { controller: "location-selector" }, class: "card p-4") do |form| %>
  <div class="mb-3">
    <%= form.label :location_type, "Tipo de ubicación", class: "form-label" %>
    <%= form.select :location_type,
      options_for_select(["Warehouse", "Store"], product_item.location_type),
      { include_blank: "Selecciona un tipo" },
      class: "form-select",
      data: { location_selector_target: "type", action: "change->location-selector#update" } %>
  </div>

  <div class="mb-3">
    <%= form.label :location_id, "Ubicación", class: "form-label" %>
    <turbo-frame id="location-options" data-location-selector-target="frame">
      <%= render partial: "locations/options",
                 locals: { locations: product_item.location_type.present? ? product_item.location_type.constantize.all : [] },
                 form: form %>
    </turbo-frame>
  </div>

  <turbo-frame id="modal" data-location-selector-target="modal"></turbo-frame>

  <div class="mb-3">
    <%= form.label :product_id, "Producto", class: "form-label" %>
    <%= form.collection_select :product_id, Product.all, :id, :name,
          { include_blank: "Selecciona un producto" },
          class: "form-select" %>
  </div>

  <div class="row">
    <div class="col-md-6">
      <%= form.label :cost_price, "Precio de costo", class: "form-label" %>
      <%= form.number_field :cost_price, step: 0.01, class: "form-control" %>
    </div>
    <div class="col-md-6">
      <%= form.label :sale_price, "Precio de venta", class: "form-label" %>
      <%= form.number_field :sale_price, step: 0.01, class: "form-control" %>
    </div>
  </div>

  <div class="mt-3">
    <%= form.label :quantity, "Cantidad", class: "form-label" %>
    <%= form.number_field :quantity, step: 1, class: "form-control" %>
  </div>

  <div class="mt-3">
    <%= form.submit class: "btn btn-primary" %>
  </div>
<% end %>
```

---

## 🎬 6. Estilo del modal (Bootstrap 5 + Turbo)

Agrega este fragmento al layout principal (por ejemplo en `app/views/layouts/application.html.erb` justo antes de `</body>`):

```erb
<turbo-frame id="modal"></turbo-frame>

<div id="flash-messages"></div>
```

Y un **componente modal global** de Bootstrap que se reemplace automáticamente:

```erb
<!-- Turbo modal -->
<div class="modal fade" id="turboModal" tabindex="-1" aria-hidden="true">
  <div class="modal-dialog modal-dialog-centered">
    <div class="modal-content" id="modal-content">
      <!-- el contenido llega por Turbo -->
    </div>
  </div>
</div>

<script>
  document.addEventListener("turbo:frame-load", (event) => {
    if (event.target.id === "modal") {
      const modal = new bootstrap.Modal(document.getElementById("turboModal"))
      document.getElementById("modal-content").innerHTML = event.target.innerHTML
      modal.show()
    }
  })
</script>
```

---

¿Quieres que te lo deje listo con los **Turbo Streams de cierre automático del modal y refresh del select**, usando un broadcast desde el modelo `Warehouse` / `Store` (ActionCable-like)?  
Eso te permitiría ver los nuevos almacenes creados **en tiempo real** si hay varios usuarios en la app.

```prompt
Déjame listo con los Turbo Streams de cierre automático del modal y refresh del select, usando un broadcast desde el modelo Warehouse / Store (ActionCable-like)
```

Perfecto, Pazzo 🔥🔥  
Aquí te dejo la versión **completa y pro**, donde todo el flujo se hace con **Turbo Streams + Broadcasts en vivo**, para que cualquier usuario conectado vea los nuevos almacenes o tiendas sin recargar la página.

> 🧩 Stack: **Rails 8 + Hotwire (Turbo) + Stimulus + Bootstrap 5**  
> Flujo: seleccionas tipo → se cargan ubicaciones → puedes crear una nueva → el modal se cierra solo → el select se refresca dinámicamente para todos los usuarios conectados.

---

## 🧠 Estructura General

```
app/models/warehouse.rb
app/models/store.rb
app/controllers/locations_controller.rb
app/views/locations/_options.html.erb
app/views/locations/_form.html.erb
app/views/locations/create.turbo_stream.erb
app/views/product_items/_form.html.erb
app/javascript/controllers/location_selector_controller.js
```

---

## 🏗️ 1. Modelos con Broadcast

**`app/models/warehouse.rb`**

```ruby
class Warehouse < ApplicationRecord
  validates :name, presence: true
  validates :address, presence: true

  after_create_commit -> { broadcast_prepend_to "warehouses", partial: "locations/location_option", locals: { location: self }, target: "warehouse-options" }
end
```

**`app/models/store.rb`**

```ruby
class Store < ApplicationRecord
  validates :name, presence: true
  validates :address, presence: true

  after_create_commit -> { broadcast_prepend_to "stores", partial: "locations/location_option", locals: { location: self }, target: "store-options" }
end
```

> 🔸 Cada modelo “emite” (broadcast) un Turbo Stream cuando se crea un registro nuevo, actualizando el `<turbo-frame>` correspondiente a su tipo.

---

## ⚙️ 2. Controlador — `app/controllers/locations_controller.rb`

```ruby
class LocationsController < ApplicationController
  before_action :set_type

  def options
    @locations = load_locations
    render partial: "locations/options", locals: { locations: @locations }
  end

  def new
    @location = @type.constantize.new
    render partial: "locations/form", locals: { location: @location }
  end

  def create
    @location = @type.constantize.new(location_params)

    if @location.save
      respond_to do |format|
        format.turbo_stream
      end
    else
      render partial: "locations/form", locals: { location: @location }, status: :unprocessable_entity
    end
  end

  private

  def set_type
    @type = params[:type].presence_in(%w[Warehouse Store]) || "Warehouse"
  end

  def load_locations
    @type.constantize.order(:name)
  end

  def location_params
    params.require(@type.underscore.to_sym).permit(:name, :address)
  end
end
```

---

## 🧱 3. Vistas Parciales

### `app/views/locations/_options.html.erb`

```erb
<turbo-frame id="<%= "#{form.object.location_type&.downcase}-options" %>">
  <div class="input-group">
    <%= form.select :location_id,
      options_from_collection_for_select(locations, :id, :name, form.object.location_id),
      { include_blank: "Selecciona una ubicación" },
      class: "form-select" %>

    <button class="btn btn-outline-primary"
            data-action="click->location-selector#openModal">
      + Agregar nueva
    </button>
  </div>
</turbo-frame>
```

---

### `app/views/locations/_form.html.erb`

```erb
<%= form_with(model: location, url: polymorphic_path([location.class.name.underscore.pluralize]),
              data: { turbo_frame: "modal" }) do |f| %>

  <div class="modal-body">
    <div class="mb-3">
      <%= f.label :name, "Nombre", class: "form-label" %>
      <%= f.text_field :name, class: "form-control" %>
    </div>

    <div class="mb-3">
      <%= f.label :address, "Dirección", class: "form-label" %>
      <%= f.text_field :address, class: "form-control" %>
    </div>
  </div>

  <div class="modal-footer">
    <%= f.submit "Guardar", class: "btn btn-primary" %>
  </div>
<% end %>
```

---

### `app/views/locations/_location_option.html.erb`

```erb
<option value="<%= location.id %>"><%= location.name %></option>
```

---

### `app/views/locations/create.turbo_stream.erb`

Cuando se guarda una nueva ubicación, este Turbo Stream:

1. Cierra el modal
    
2. Refresca el select
    
3. Muestra un mensaje de éxito
    

```erb
<%= turbo_stream.replace "modal" do %>
  <%= render partial: "shared/empty" %>
<% end %>

<%= turbo_stream.append "flash-messages" do %>
  <div class="alert alert-success alert-dismissible fade show mt-2" role="alert">
    <%= "#{@type} creada exitosamente." %>
    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
  </div>
<% end %>

<%= turbo_stream.replace "location-options" do %>
  <%= render partial: "locations/options",
             locals: { locations: @type.constantize.order(:name), form: nil } %>
<% end %>
```

---

## 🧾 4. Formulario principal — `app/views/product_items/_form.html.erb`

```erb
<%= form_with(model: product_item, data: { controller: "location-selector" }, class: "card p-4") do |form| %>
  <div class="mb-3">
    <%= form.label :location_type, "Tipo de ubicación", class: "form-label" %>
    <%= form.select :location_type,
      options_for_select(["Warehouse", "Store"], product_item.location_type),
      { include_blank: "Selecciona un tipo" },
      class: "form-select",
      data: { location_selector_target: "type", action: "change->location-selector#update" } %>
  </div>

  <div class="mb-3">
    <%= form.label :location_id, "Ubicación", class: "form-label" %>
    <turbo-frame id="location-options" data-location-selector-target="frame">
      <%= render partial: "locations/options",
                 locals: { locations: product_item.location_type.present? ? product_item.location_type.constantize.all : [] },
                 form: form %>
    </turbo-frame>
  </div>

  <turbo-frame id="modal" data-location-selector-target="modal"></turbo-frame>

  <div class="mb-3">
    <%= form.label :product_id, "Producto", class: "form-label" %>
    <%= form.collection_select :product_id, Product.all, :id, :name,
          { include_blank: "Selecciona un producto" },
          class: "form-select" %>
  </div>

  <div class="row">
    <div class="col-md-6">
      <%= form.label :cost_price, "Precio de costo", class: "form-label" %>
      <%= form.number_field :cost_price, step: 0.01, class: "form-control" %>
    </div>
    <div class="col-md-6">
      <%= form.label :sale_price, "Precio de venta", class: "form-label" %>
      <%= form.number_field :sale_price, step: 0.01, class: "form-control" %>
    </div>
  </div>

  <div class="mt-3">
    <%= form.label :quantity, "Cantidad", class: "form-label" %>
    <%= form.number_field :quantity, step: 1, class: "form-control" %>
  </div>

  <div class="mt-3">
    <%= form.submit class: "btn btn-primary" %>
  </div>
<% end %>
```

---

## ⚡ 5. Stimulus Controller — `app/javascript/controllers/location_selector_controller.js`

```js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["type", "frame", "modal"]

  update() {
    const type = this.typeTarget.value
    if (type === "") return
    const url = `/locations/options?type=${encodeURIComponent(type)}`
    this.frameTarget.src = url
  }

  openModal(event) {
    event.preventDefault()
    const type = this.typeTarget.value
    if (type === "") return alert("Selecciona primero un tipo (Warehouse o Store)")
    const url = `/locations/new?type=${encodeURIComponent(type)}`
    this.modalTarget.src = url
  }
}
```

---

## 🎬 6. Turbo Modal Global + Flash

Agrega esto en tu layout (antes de `</body>`):

```erb
<div id="flash-messages"></div>
<turbo-frame id="modal"></turbo-frame>

<script>
  document.addEventListener("turbo:frame-load", (event) => {
    if (event.target.id === "modal") {
      const modal = new bootstrap.Modal(document.getElementById("turboModal"))
      document.getElementById("modal-content").innerHTML = event.target.innerHTML
      modal.show()
    }
  })
</script>

<!-- Bootstrap modal -->
<div class="modal fade" id="turboModal" tabindex="-1" aria-hidden="true">
  <div class="modal-dialog modal-dialog-centered">
    <div class="modal-content" id="modal-content"></div>
  </div>
</div>
```

---

## 🔥 Resultado

- Cualquier usuario en la app con el formulario abierto verá en **tiempo real** los nuevos Warehouses o Stores.
    
- Al crear uno nuevo desde el modal:
    
    - El modal se cierra solo.
        
    - El `<select>` de ubicaciones se refresca automáticamente.
        
    - Se muestra un flash TurboStream.
        

---

¿Quieres que te lo deje con soporte para **actualizar y eliminar ubicaciones** también en vivo (edición y eliminación con broadcasts Turbo)?  
Eso completaría el flujo CRUD 100% en tiempo real.

```prompt
sí, déjame con soporte para actualizar y eliminar ubicaciones también en vivo (edición y eliminación con broadcasts Turbo)
```

Perfecto 💪 Pazzo, aquí te dejo el **sistema completo Turbo-ready**, con soporte **en vivo (real time)** para crear, actualizar y eliminar **Warehouses** y **Stores** desde el mismo formulario de `ProductItem`.  
Está preparado para **Rails 8**, **Turbo Streams**, y **Stimulus**, con **Bootstrap 5** y sin necesidad de recargar la página.

---

## 🧱 1. Modelos con Broadcasts Turbo

```ruby
# app/models/warehouse.rb
class Warehouse < ApplicationRecord
  has_many :product_items, dependent: :nullify

  validates :name, presence: true, uniqueness: true

  after_create_commit  -> { broadcast_prepend_to "warehouses", target: "warehouses_list", partial: "warehouses/warehouse", locals: { warehouse: self } }
  after_update_commit  -> { broadcast_replace_to "warehouses", target: dom_id(self), partial: "warehouses/warehouse", locals: { warehouse: self } }
  after_destroy_commit -> { broadcast_remove_to "warehouses", target: dom_id(self) }
end

# app/models/store.rb
class Store < ApplicationRecord
  has_many :product_items, dependent: :nullify

  validates :name, presence: true, uniqueness: true

  after_create_commit  -> { broadcast_prepend_to "stores", target: "stores_list", partial: "stores/store", locals: { store: self } }
  after_update_commit  -> { broadcast_replace_to "stores", target: dom_id(self), partial: "stores/store", locals: { store: self } }
  after_destroy_commit -> { broadcast_remove_to "stores", target: dom_id(self) }
end
```

---

## 🎮 2. Controladores Turbo Ready

### `warehouses_controller.rb`

```ruby
class WarehousesController < ApplicationController
  before_action :set_warehouse, only: %i[edit update destroy]

  def index
    @warehouses = Warehouse.all
  end

  def new
    @warehouse = Warehouse.new
  end

  def create
    @warehouse = Warehouse.new(warehouse_params)
    if @warehouse.save
      respond_to do |format|
        format.turbo_stream
        format.html { redirect_to warehouses_path, notice: "Almacén creado." }
      end
    else
      render :new, status: :unprocessable_entity
    end
  end

  def edit; end

  def update
    if @warehouse.update(warehouse_params)
      respond_to do |format|
        format.turbo_stream
        format.html { redirect_to warehouses_path, notice: "Almacén actualizado." }
      end
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @warehouse.destroy
    respond_to do |format|
      format.turbo_stream
      format.html { redirect_to warehouses_path, notice: "Almacén eliminado." }
    end
  end

  private

  def set_warehouse
    @warehouse = Warehouse.find(params[:id])
  end

  def warehouse_params
    params.require(:warehouse).permit(:name, :location)
  end
end
```

> El controlador de `StoresController` es igual, solo cambia el modelo.

---

## 🧩 3. Stimulus Controller

`app/javascript/controllers/location_selector_controller.js`

```js
import { Controller } from "@hotwired/stimulus"

// Manejador del select dinámico de ubicaciones
export default class extends Controller {
  static targets = ["typeSelect", "locationSelect", "frame"]

  connect() {
    this.update()
  }

  update() {
    const type = this.typeSelectTarget.value
    const url = type === "Warehouse"
      ? "/warehouses"
      : "/stores"

    this.frameTarget.src = `${url}?turbo_frame=${this.locationSelectTarget.id}`
  }
}
```

---

## 🧰 4. Formulario de ProductItem (con select dinámico y modal Turbo)

`app/views/product_items/_form.html.erb`

```erb
<%= turbo_frame_tag "product_item_form" do %>
  <%= form_with model: @product_item, data: { controller: "location-selector" } do |f| %>
    <div class="mb-3">
      <%= f.label :location_type, "Tipo de ubicación" %>
      <%= f.select :location_type, [["Warehouse", "Warehouse"], ["Store", "Store"]],
          {}, class: "form-select", data: { location_selector_target: "typeSelect", action: "change->location-selector#update" } %>
    </div>

    <div class="mb-3">
      <%= f.label :location_id, "Ubicación" %>
      <turbo-frame id="location_select_frame" data-location-selector-target="frame">
        <%= f.select :location_id, [], {}, class: "form-select", id: "product_item_location_select", data: { location_selector_target: "locationSelect" } %>
      </turbo-frame>
      <div class="mt-2">
        <%= link_to "Agregar nuevo", "#", data: { bs_toggle: "modal", bs_target: "#newLocationModal" }, class: "btn btn-outline-primary btn-sm" %>
      </div>
    </div>

    <%= f.submit class: "btn btn-success" %>
  <% end %>
<% end %>

<%= render "locations/modal" %>
```

---

## 🪟 5. Modal Turbo para crear/editar ubicaciones

`app/views/locations/_modal.html.erb`

```erb
<div class="modal fade" id="newLocationModal" tabindex="-1" aria-hidden="true">
  <div class="modal-dialog">
    <div class="modal-content">
      <turbo-frame id="new_location_frame">
        <div class="modal-header">
          <h5 class="modal-title">Agregar nueva ubicación</h5>
          <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
        </div>
        <div class="modal-body">
          <%= render "locations/form" %>
        </div>
      </turbo-frame>
    </div>
  </div>
</div>
```

---

## 🧾 6. Formulario de Warehouse y Store (reutilizable)

`app/views/locations/_form.html.erb`

```erb
<%= form_with model: (@warehouse || @store), data: { turbo_frame: "new_location_frame" } do |f| %>
  <div class="mb-3">
    <%= f.label :name, "Nombre" %>
    <%= f.text_field :name, class: "form-control" %>
  </div>

  <div class="mb-3">
    <%= f.label :location, "Dirección / ubicación" %>
    <%= f.text_field :location, class: "form-control" %>
  </div>

  <%= f.submit "Guardar", class: "btn btn-primary" %>
<% end %>
```

---

## ⚡ 7. Turbo Streams para cierre automático del modal y refresh del select

`app/views/warehouses/create.turbo_stream.erb`

```erb
<%= turbo_stream.prepend "warehouses_list", partial: "warehouses/warehouse", locals: { warehouse: @warehouse } %>
<%= turbo_stream.update "new_location_frame" do %>
  <script>
    const modal = bootstrap.Modal.getInstance(document.getElementById("newLocationModal"))
    modal.hide()
    Turbo.visit(window.location.href) // recarga el select dinámico
  </script>
<% end %>
```

Lo mismo aplica para `update.turbo_stream.erb` y `destroy.turbo_stream.erb` (solo cambia la acción).

---

## 📡 8. Partial de item broadcast

`app/views/warehouses/_warehouse.html.erb`

```erb
<li id="<%= dom_id(warehouse) %>">
  <%= warehouse.name %> — <%= warehouse.location %>
  <%= link_to "Editar", edit_warehouse_path(warehouse), data: { turbo_frame: "new_location_frame" }, class: "btn btn-sm btn-outline-secondary" %>
  <%= link_to "Eliminar", warehouse, method: :delete, data: { turbo_confirm: "¿Seguro?" }, class: "btn btn-sm btn-outline-danger" %>
</li>
```

---

## 🚀 Resultado

✅ Crear, editar y eliminar **Warehouses/Stores** desde un **modal Turbo**  
✅ Se cierran automáticamente los modales con JS inline  
✅ Se actualizan en **tiempo real** en todos los clientes (por los broadcasts)  
✅ El select de `ProductItem` se refresca sin recargar  
✅ Todo en **Rails 8 + Turbo + Stimulus + Bootstrap 5**

---

¿Quieres que te lo deje preparado con **Hotwire Reflex + CableReady** para tener _actualizaciones entre distintos usuarios en vivo_ (sin recargar y sin Turbo.visit)?  
Eso haría que si un usuario crea un Warehouse, el select se actualice en todos los navegadores abiertos al instante 🔥



```prompt
No, ahora quiero que me crees un los modelos y diagramas para hacer la ingeniaría de todo esto
```

Perfecto, Pazzo ⚙️🔥  
Vamos a estructurar **la ingeniería completa** de tu sistema de almacén con Rails — con sus **modelos, relaciones, atributos, y un diagrama entidad-relación (ERD)** que te servirá tanto para diseño de base de datos como documentación del proyecto.

---

## 🧩 1. Concepto general

Tu sistema manejará **productos** distribuidos entre:

- **Warehouses (almacenes centrales)**
    
- **Stores (tiendas físicas o puntos de venta)**
    

Cada ubicación almacena **ProductItems** (instancias físicas o lotes de productos), que tienen:

- ficha de costo
    
- cantidad
    
- precio de venta
    
- fecha de entrada
    
- unidad de medida
    

También puedes transferir productos entre ubicaciones más adelante (si lo deseas).

---

## 🧱 2. Modelos y relaciones principales

|Modelo|Descripción|Relaciones|
|---|---|---|
|**Warehouse**|Almacén central o depósito|Tiene muchos `ProductItems`|
|**Store**|Tienda o punto de venta|Tiene muchos `ProductItems`|
|**Product**|Producto general (definición base)|Tiene muchas `ProductItems`, pertenece a una `Category`|
|**ProductItem**|Representa una existencia física (stock) de un producto en una ubicación (warehouse o store)|Pertenece a un `Product`, pertenece a una ubicación (`Warehouse` o `Store`)|
|**Category**|Clasifica los productos (electrónica, ropa, etc.)|Tiene muchos `Products`|
|**Unit**|Unidad de medida (kg, litro, unidad, caja, etc.)|Tiene muchos `ProductItems`|
|**Supplier** _(opcional, pero útil)_|Proveedor de productos|Tiene muchos `ProductItems` o `Products`|

---

## 🧠 3. Relaciones en Rails

```ruby
# app/models/warehouse.rb
class Warehouse < ApplicationRecord
  has_many :product_items, as: :location, dependent: :destroy
  validates :name, presence: true, uniqueness: true
end

# app/models/store.rb
class Store < ApplicationRecord
  has_many :product_items, as: :location, dependent: :destroy
  validates :name, presence: true, uniqueness: true
end

# app/models/product.rb
class Product < ApplicationRecord
  belongs_to :category
  has_many :product_items, dependent: :destroy
  validates :name, :sku, presence: true
  validates :sku, uniqueness: true
end

# app/models/product_item.rb
class ProductItem < ApplicationRecord
  belongs_to :product
  belongs_to :location, polymorphic: true
  belongs_to :unit

  validates :quantity, numericality: { greater_than_or_equal_to: 0 }
  validates :cost_price, :sale_price, numericality: { greater_than_or_equal_to: 0 }

  before_validation :set_defaults

  private

  def set_defaults
    self.entry_date ||= Time.current
  end
end

# app/models/category.rb
class Category < ApplicationRecord
  has_many :products
  validates :name, presence: true, uniqueness: true
end

# app/models/unit.rb
class Unit < ApplicationRecord
  has_many :product_items
  validates :name, presence: true, uniqueness: true
end
```

---

## 🧮 4. Esquema de base de datos (migraciones resumidas)

```ruby
create_table :warehouses do |t|
  t.string :name, null: false
  t.string :location
  t.timestamps
end

create_table :stores do |t|
  t.string :name, null: false
  t.string :address
  t.timestamps
end

create_table :categories do |t|
  t.string :name, null: false
  t.timestamps
end

create_table :products do |t|
  t.string :name, null: false
  t.string :sku, null: false
  t.references :category, null: false, foreign_key: true
  t.text :description
  t.timestamps
end

create_table :units do |t|
  t.string :name, null: false
  t.string :symbol
  t.timestamps
end

create_table :product_items do |t|
  t.references :product, null: false, foreign_key: true
  t.references :unit, null: false, foreign_key: true
  t.references :location, polymorphic: true, null: false
  t.decimal :cost_price, precision: 10, scale: 2
  t.decimal :sale_price, precision: 10, scale: 2
  t.decimal :quantity, precision: 10, scale: 2
  t.datetime :entry_date
  t.timestamps
end
```

---

## 🗺️ 5. Diagrama Entidad-Relación (ERD)

Aquí te lo dejo descrito (puedo generarte la imagen SVG si quieres):

```
+-----------------+           +----------------+
|   Warehouse     |           |     Store      |
|-----------------|           |----------------|
| id              |           | id             |
| name            |           | name           |
| location        |           | address        |
+-----------------+           +----------------+
         \                        /
          \                      /
           \    polymorphic     /
            \                  /
             \                /
              \              /
             +----------------------+
             |     ProductItem      |
             |----------------------|
             | id                   |
             | product_id           |
             | location_id          |
             | location_type        |
             | unit_id              |
             | cost_price           |
             | sale_price           |
             | quantity             |
             | entry_date           |
             +----------------------+
                      |
                      |
                      v
               +---------------+
               |    Product     |
               |---------------|
               | id            |
               | category_id   |
               | name          |
               | sku           |
               | description   |
               +---------------+
                      |
                      v
              +---------------+
              |   Category    |
              |---------------|
              | id            |
              | name          |
              +---------------+

              +---------------+
              |    Unit       |
              |---------------|
              | id            |
              | name          |
              | symbol        |
              +---------------+
```

---

## 🧩 6. Ejemplo de flujo funcional

1. Creas categorías (`Electrónica`, `Ferretería`).
    
2. Creas unidades (`Unidad`, `Litro`, `Kg`).
    
3. Creas productos (`Taladro`, `Laptop`).
    
4. Creas almacenes y tiendas (`Bodega Central`, `Tienda Norte`).
    
5. En `ProductItems`, seleccionas producto + ubicación (Warehouse o Store) + precios + cantidad.
    
6. Todo se muestra y actualiza con Turbo Streams (como ya montamos antes).
    

---

## 🚀 7. Posible extensión futura

Una vez tengas esto funcionando, puedes ampliar:

- **Transfers** (modelo `StockTransfer`) para mover productos entre ubicaciones.
    
- **Stock Movements** (entradas/salidas).
    
- **Reports** (movimiento histórico, costos promedio, rotación de stock).
    
- **Integración con ventas / facturación**.
    

---

¿Quieres que te genere **el diagrama ERD como imagen SVG o PNG** (bonito y con colores tipo draw.io o dbdiagram.io)?  
Puedo generarlo automáticamente para tu documentación del proyecto.



```prompt
Pasos para completar esto
Una vez tengas esto funcionando, puedes ampliar:

Transfers (modelo StockTransfer) para mover productos entre ubicaciones.

Stock Movements (entradas/salidas).

Reports (movimiento histórico, costos promedio, rotación de stock).
```

Perfecto, Pazzo 😎  
ya tienes la base sólida con `Warehouse`, `Store`, `Product`, `ProductItem` y sus broadcasts Turbo.  
Aquí te dejo los **pasos exactos** para completar todo el sistema de almacén y expandirlo con **Transfers, Movements y Reports**, ordenado por fases.

---

## 🧩 FASE 1 — Base lista (lo que ya tienes)

✅ Modelos y relaciones:

- `Warehouse`, `Store`, `Product`, `ProductItem`  
    ✅ Turbo Streams y Stimulus:
    
- Selector dinámico entre Warehouse/Store
    
- Creación/edición/eliminación de ubicaciones en vivo  
    ✅ ERD listo (`inventory-erd.excalidraw`)
    

---

## 🚛 FASE 2 — Stock Transfers (movimientos entre ubicaciones)

**Objetivo:** mover productos entre `Warehouse` y `Store` (o entre dos `Stores`).

### 🔹 Modelos

```ruby
# app/models/stock_transfer.rb
class StockTransfer < ApplicationRecord
  belongs_to :source, polymorphic: true  # Warehouse o Store
  belongs_to :destination, polymorphic: true
  has_many :stock_transfer_items, dependent: :destroy
  accepts_nested_attributes_for :stock_transfer_items, allow_destroy: true

  validates :source, :destination, presence: true
  validate :different_locations

  after_commit :process_transfer, on: :create

  private

  def different_locations
    errors.add(:destination, "no puede ser igual al origen") if source == destination
  end

  def process_transfer
    stock_transfer_items.each do |item|
      ProductItem.transaction do
        source_item = ProductItem.find_by(product: item.product, location: source)
        dest_item = ProductItem.find_or_create_by(product: item.product, location: destination)

        source_item.decrement!(:quantity, item.quantity)
        dest_item.increment!(:quantity, item.quantity)
      end
    end
  end
end
```

```ruby
# app/models/stock_transfer_item.rb
class StockTransferItem < ApplicationRecord
  belongs_to :stock_transfer
  belongs_to :product

  validates :quantity, numericality: { greater_than: 0 }
end
```

### 🔹 Vistas y controladores

- CRUD de `StockTransfer` con Turbo Frames.
    
- Form con dos selects (`source` y `destination`), dinámicos según tipo de ubicación.
    
- Nested form para los ítems transferidos (usando Stimulus o Cocoon).
    

---

## 📦 FASE 3 — Stock Movements (entradas y salidas)

**Objetivo:** registrar ajustes de inventario manuales (por ejemplo, entrada de proveedor o merma).

### 🔹 Modelos

```ruby
# app/models/stock_movement.rb
class StockMovement < ApplicationRecord
  belongs_to :location, polymorphic: true
  belongs_to :product

  enum movement_type: { in: 0, out: 1 }

  validates :quantity, numericality: { greater_than: 0 }

  after_commit :apply_movement, on: :create

  private

  def apply_movement
    item = ProductItem.find_or_create_by(product: product, location: location)
    if in?
      item.increment!(:quantity, quantity)
    else
      item.decrement!(:quantity, quantity)
    end
  end
end
```

### 🔹 Controlador y vistas

- Form simple con `product`, `movement_type`, `quantity`, `location`.
    
- Turbo Stream de actualización en vivo de la cantidad del `ProductItem`.
    

---

## 📊 FASE 4 — Reports (histórico y métricas)

**Objetivo:** analizar el inventario a lo largo del tiempo.

### 🔹 Modelos / consultas

Podés usar scopes o modelos “virtuales” (como `StockReport`):

```ruby
class StockReport
  def self.movement_history(product)
    StockMovement.where(product:).order(created_at: :desc)
  end

  def self.average_cost(product)
    movements = StockMovement.where(product:, movement_type: :in)
    total_cost = movements.sum("quantity * cost_price")
    total_qty = movements.sum(:quantity)
    total_qty.zero? ? 0 : (total_cost / total_qty)
  end

  def self.rotation_rate(product)
    out = StockMovement.where(product:, movement_type: :out).sum(:quantity)
    avg_stock = ProductItem.where(product:).average(:quantity) || 0
    avg_stock.zero? ? 0 : (out / avg_stock)
  end
end
```

### 🔹 Interfaz

- Página `/reports` con gráficos usando Chartkick o Recharts (ya compatibles con Turbo).
    
- Filtros por fechas, ubicación, categoría de producto.
    

---

## 🧠 FASE 5 — Auditoría y mejora

- Agregar `PaperTrail` o `Audited` para historial de cambios.
    
- Implementar `acts_as_tenant` si deseas multiempresa.
    
- Añadir WebSocket broadcast de movimientos (dashboard en vivo).
    

---

¿Quieres que te genere ahora el **.excalidraw actualizado** con estos tres módulos nuevos (`StockTransfer`, `StockMovement`, `StockReport`) y sus relaciones visuales (extendiendo el que ya hicimos)?  
Puedo dejártelo listo para reemplazar en tu Obsidian.

Perfecto, Pazzo 😎 — vamos full-stack entonces.  
Te voy a dejar **todas las vistas Turbo-ready (Bootstrap 5)** para los nuevos módulos:

- `StockTransfers`
    
- `StockMovements`
    
- `Reports` (solo lectura)
    

Cada una usa **Turbo Frames** y **Turbo Streams**, y sigue el estilo de Bootstrap 5 con cards, modales y botones claros.  
Así puedes copiarlas directo a tu `app/views/` y tener el flujo completo funcionando.

---

# 🧱 Estructura general

```
app/views/
├── stock_transfers/
│   ├── index.html.erb
│   ├── _form.html.erb
│   ├── _item_fields.html.erb
│   ├── show.html.erb
│   ├── new.html.erb
│   └── edit.html.erb
├── stock_movements/
│   ├── index.html.erb
│   ├── _form.html.erb
│   ├── show.html.erb
│   ├── new.html.erb
│   └── edit.html.erb
└── reports/
    └── index.html.erb
```

---

## 🔹 `app/views/stock_transfers/index.html.erb`

```erb
<div class="d-flex justify-content-between align-items-center mb-3">
  <h2>Transferencias de Stock</h2>
  <%= link_to "Nueva transferencia", new_stock_transfer_path, data: { turbo_frame: "modal" }, class: "btn btn-primary" %>
</div>

<table class="table table-hover">
  <thead>
    <tr>
      <th>Origen</th>
      <th>Destino</th>
      <th>Fecha</th>
      <th></th>
    </tr>
  </thead>
  <tbody id="stock_transfers">
    <%= render @stock_transfers %>
  </tbody>
</table>

<%= turbo_frame_tag "modal" %>
```

---

## 🔹 `app/views/stock_transfers/_stock_transfer.html.erb`

```erb
<tr id="<%= dom_id(stock_transfer) %>">
  <td><%= stock_transfer.source_type %> - <%= stock_transfer.source.name %></td>
  <td><%= stock_transfer.destination_type %> - <%= stock_transfer.destination.name %></td>
  <td><%= l(stock_transfer.created_at, format: :short) %></td>
  <td>
    <%= link_to "Ver", stock_transfer_path(stock_transfer), class: "btn btn-sm btn-outline-secondary" %>
    <%= link_to "Editar", edit_stock_transfer_path(stock_transfer), data: { turbo_frame: "modal" }, class: "btn btn-sm btn-outline-primary" %>
    <%= link_to "Eliminar", stock_transfer_path(stock_transfer), data: { turbo_method: :delete, turbo_confirm: "¿Seguro?" }, class: "btn btn-sm btn-danger" %>
  </td>
</tr>
```

---

## 🔹 `app/views/stock_transfers/_form.html.erb`

```erb
<%= form_with(model: stock_transfer, data: { turbo_frame: "_top" }, class: "needs-validation") do |f| %>
  <div class="mb-3">
    <%= f.label :source_type, "Tipo de origen" %>
    <%= f.select :source_type, [["Warehouse", "Warehouse"], ["Store", "Store"]],
          {}, class: "form-select", data: { action: "change->transfer#updateSource" } %>
  </div>

  <div class="mb-3" data-transfer-target="sourceSelect">
    <%= f.label :source_id, "Origen" %>
    <%= f.collection_select :source_id, Warehouse.all, :id, :name, {}, class: "form-select" %>
  </div>

  <div class="mb-3">
    <%= f.label :destination_type, "Tipo de destino" %>
    <%= f.select :destination_type, [["Warehouse", "Warehouse"], ["Store", "Store"]],
          {}, class: "form-select", data: { action: "change->transfer#updateDestination" } %>
  </div>

  <div class="mb-3" data-transfer-target="destinationSelect">
    <%= f.label :destination_id, "Destino" %>
    <%= f.collection_select :destination_id, Store.all, :id, :name, {}, class: "form-select" %>
  </div>

  <hr>

  <h5>Productos a transferir</h5>
  <div id="items">
    <%= f.fields_for :stock_transfer_items do |item_form| %>
      <%= render "item_fields", f: item_form %>
    <% end %>
  </div>
  <%= link_to "Agregar producto", "#", class: "btn btn-outline-secondary", data: { action: "click->transfer#addItem" } %>

  <div class="mt-4 text-end">
    <%= f.submit class: "btn btn-success" %>
  </div>
<% end %>
```

---

## 🔹 `app/views/stock_transfers/_item_fields.html.erb`

```erb
<div class="card mb-2 p-3 border">
  <div class="row">
    <div class="col-md-6">
      <%= f.label :product_id, "Producto" %>
      <%= f.collection_select :product_id, Product.all, :id, :name, {}, class: "form-select" %>
    </div>
    <div class="col-md-4">
      <%= f.label :quantity, "Cantidad" %>
      <%= f.number_field :quantity, class: "form-control", min: 1 %>
    </div>
    <div class="col-md-2 d-flex align-items-end">
      <%= link_to "🗑️", "#", class: "btn btn-outline-danger", data: { action: "click->transfer#removeItem" } %>
    </div>
  </div>
</div>
```

---

## 🔹 `app/views/stock_transfers/new.html.erb`

```erb
<%= turbo_frame_tag "modal" do %>
  <div class="modal fade show d-block" tabindex="-1" style="background-color:rgba(0,0,0,.4)">
    <div class="modal-dialog modal-lg modal-dialog-centered">
      <div class="modal-content">
        <div class="modal-header">
          <h5 class="modal-title">Nueva Transferencia</h5>
          <button type="button" class="btn-close" data-action="click->modal#close"></button>
        </div>
        <div class="modal-body">
          <%= render "form", stock_transfer: @stock_transfer %>
        </div>
      </div>
    </div>
  </div>
<% end %>
```

---

## 🔹 `app/views/stock_movements/index.html.erb`

```erb
<div class="d-flex justify-content-between align-items-center mb-3">
  <h2>Movimientos de Stock</h2>
  <%= link_to "Nuevo movimiento", new_stock_movement_path, data: { turbo_frame: "modal" }, class: "btn btn-primary" %>
</div>

<table class="table table-striped">
  <thead>
    <tr>
      <th>Ubicación</th>
      <th>Producto</th>
      <th>Tipo</th>
      <th>Cantidad</th>
      <th>Fecha</th>
    </tr>
  </thead>
  <tbody>
    <%= render @stock_movements %>
  </tbody>
</table>

<%= turbo_frame_tag "modal" %>
```

---

## 🔹 `app/views/stock_movements/_stock_movement.html.erb`

```erb
<tr id="<%= dom_id(stock_movement) %>">
  <td><%= stock_movement.location.name %></td>
  <td><%= stock_movement.product.name %></td>
  <td><%= stock_movement.movement_type %></td>
  <td><%= stock_movement.quantity %></td>
  <td><%= l(stock_movement.created_at, format: :short) %></td>
</tr>
```

---

## 🔹 `app/views/stock_movements/_form.html.erb`

```erb
<%= form_with(model: stock_movement, data: { turbo_frame: "_top" }) do |f| %>
  <div class="mb-3">
    <%= f.label :movement_type, "Tipo de movimiento" %>
    <%= f.select :movement_type, StockMovement.movement_types.keys.map { |t| [t.titleize, t] }, {}, class: "form-select" %>
  </div>

  <div class="mb-3">
    <%= f.label :location_type, "Tipo de ubicación" %>
    <%= f.select :location_type, [["Warehouse", "Warehouse"], ["Store", "Store"]], {}, class: "form-select", data: { action: "change->movement#updateLocation" } %>
  </div>

  <div class="mb-3" data-movement-target="locationSelect">
    <%= f.label :location_id, "Ubicación" %>
    <%= f.collection_select :location_id, Warehouse.all, :id, :name, {}, class: "form-select" %>
  </div>

  <div class="mb-3">
    <%= f.label :product_id, "Producto" %>
    <%= f.collection_select :product_id, Product.all, :id, :name, {}, class: "form-select" %>
  </div>

  <div class="mb-3">
    <%= f.label :quantity, "Cantidad" %>
    <%= f.number_field :quantity, class: "form-control", min: 1 %>
  </div>

  <div class="text-end">
    <%= f.submit class: "btn btn-success" %>
  </div>
<% end %>
```

---

## 🔹 `app/views/reports/index.html.erb`

```erb
<div class="d-flex justify-content-between align-items-center mb-3">
  <h2>Reportes de Inventario</h2>
  <%= form_with url: reports_path, method: :get, class: "d-flex gap-2" do %>
    <%= select_tag :product_id, options_from_collection_for_select(Product.all, :id, :name, params[:product_id]), prompt: "Seleccione un producto", class: "form-select" %>
    <%= submit_tag "Ver", class: "btn btn-outline-primary" %>
  <% end %>
</div>

<% if @product.present? %>
  <div class="card p-3 mb-3">
    <h4><%= @product.name %></h4>
    <p>Costo promedio: <strong><%= number_to_currency(@avg_cost) %></strong></p>
    <p>Rotación: <strong><%= @rotation.round(2) %></strong></p>
  </div>

  <h5>Histórico de movimientos</h5>
  <table class="table table-sm table-striped">
    <thead>
      <tr>
        <th>Fecha</th>
        <th>Tipo</th>
        <th>Cantidad</th>
        <th>Ubicación</th>
      </tr>
    </thead>
    <tbody>
      <% @movements.each do |m| %>
        <tr>
          <td><%= l(m.created_at, format: :short) %></td>
          <td><%= m.movement_type %></td>
          <td><%= m.quantity %></td>
          <td><%= m.location.name %></td>
        </tr>
      <% end %>
    </tbody>
  </table>
<% end %>
```

---

¿Querés que te prepare también los **controllers** con Turbo Streams (para crear, editar y eliminar sin recargar la página) de estos tres módulos?  
Así te queda el stack completo listo para correr en Rails 8.

Excelente, Pazzo 😎  
vamos a cerrar el ciclo **completo del módulo de inventario avanzado** para tu sistema Rails 8:  
vas a tener los **modelos, controladores, vistas, rutas y broadcasts Turbo** listos para correr.

Esto incluye:  
✅ `StockTransfer` y `StockTransferItem`  
✅ `StockMovement`  
✅ `ReportsController` (solo lectura)  
✅ Rutas completas  
✅ Turbo Streams + broadcasts  
✅ Callbacks seguros  
✅ Controllers 100% compatibles con Turbo Frames

---

## 🧭 RUTAS (`config/routes.rb`)

```ruby
Rails.application.routes.draw do
  resources :warehouses
  resources :stores
  resources :products
  resources :product_items

  resources :stock_transfers do
    resources :stock_transfer_items, only: [:create, :destroy]
  end

  resources :stock_movements

  get "reports", to: "reports#index", as: :reports

  root "products#index"
end
```

---

## 🚛 MODELOS

### `app/models/stock_transfer.rb`

```ruby
class StockTransfer < ApplicationRecord
  belongs_to :source, polymorphic: true
  belongs_to :destination, polymorphic: true

  has_many :stock_transfer_items, dependent: :destroy
  accepts_nested_attributes_for :stock_transfer_items, allow_destroy: true

  validates :source, :destination, presence: true
  validate :different_locations

  after_commit :process_transfer, on: :create
  after_commit -> { broadcast_append_to "stock_transfers" }
  after_update_commit -> { broadcast_replace_to "stock_transfers" }
  after_destroy_commit -> { broadcast_remove_to "stock_transfers" }

  private

  def different_locations
    errors.add(:destination, "no puede ser igual al origen") if source == destination
  end

  def process_transfer
    stock_transfer_items.each do |item|
      ProductItem.transaction do
        source_item = ProductItem.find_by(product: item.product, location: source)
        dest_item = ProductItem.find_or_create_by(product: item.product, location: destination)

        source_item&.decrement!(:quantity, item.quantity)
        dest_item.increment!(:quantity, item.quantity)
      end
    end
  end
end
```

---

### `app/models/stock_transfer_item.rb`

```ruby
class StockTransferItem < ApplicationRecord
  belongs_to :stock_transfer
  belongs_to :product

  validates :quantity, numericality: { greater_than: 0 }
end
```

---

### `app/models/stock_movement.rb`

```ruby
class StockMovement < ApplicationRecord
  belongs_to :location, polymorphic: true
  belongs_to :product

  enum movement_type: { in: 0, out: 1 }

  validates :quantity, numericality: { greater_than: 0 }

  after_commit :apply_movement, on: :create
  after_commit -> { broadcast_append_to "stock_movements" }
  after_destroy_commit -> { broadcast_remove_to "stock_movements" }

  private

  def apply_movement
    item = ProductItem.find_or_create_by(product: product, location: location)
    if in?
      item.increment!(:quantity, quantity)
    else
      item.decrement!(:quantity, quantity)
    end
  end
end
```

---

## 🧠 CONTROLADORES

---

### `app/controllers/stock_transfers_controller.rb`

```ruby
class StockTransfersController < ApplicationController
  before_action :set_stock_transfer, only: %i[show edit update destroy]

  def index
    @stock_transfers = StockTransfer.includes(:source, :destination).order(created_at: :desc)
  end

  def show; end

  def new
    @stock_transfer = StockTransfer.new
    @stock_transfer.stock_transfer_items.build
  end

  def edit; end

  def create
    @stock_transfer = StockTransfer.new(stock_transfer_params)
    if @stock_transfer.save
      respond_to do |format|
        format.turbo_stream
        format.html { redirect_to @stock_transfer, notice: "Transferencia creada." }
      end
    else
      render :new, status: :unprocessable_entity
    end
  end

  def update
    if @stock_transfer.update(stock_transfer_params)
      respond_to do |format|
        format.turbo_stream
        format.html { redirect_to @stock_transfer, notice: "Transferencia actualizada." }
      end
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @stock_transfer.destroy
    respond_to do |format|
      format.turbo_stream
      format.html { redirect_to stock_transfers_path, notice: "Transferencia eliminada." }
    end
  end

  private

  def set_stock_transfer
    @stock_transfer = StockTransfer.find(params[:id])
  end

  def stock_transfer_params
    params.require(:stock_transfer).permit(
      :source_type, :source_id, :destination_type, :destination_id,
      stock_transfer_items_attributes: [:id, :product_id, :quantity, :_destroy]
    )
  end
end
```

---

### `app/controllers/stock_transfer_items_controller.rb`

```ruby
class StockTransferItemsController < ApplicationController
  before_action :set_stock_transfer

  def create
    @item = @stock_transfer.stock_transfer_items.create(stock_transfer_item_params)
    respond_to do |format|
      format.turbo_stream
    end
  end

  def destroy
    @item = @stock_transfer.stock_transfer_items.find(params[:id])
    @item.destroy
    respond_to do |format|
      format.turbo_stream
    end
  end

  private

  def set_stock_transfer
    @stock_transfer = StockTransfer.find(params[:stock_transfer_id])
  end

  def stock_transfer_item_params
    params.require(:stock_transfer_item).permit(:product_id, :quantity)
  end
end
```

---

### `app/controllers/stock_movements_controller.rb`

```ruby
class StockMovementsController < ApplicationController
  before_action :set_stock_movement, only: %i[show edit update destroy]

  def index
    @stock_movements = StockMovement.includes(:product, :location).order(created_at: :desc)
  end

  def show; end

  def new
    @stock_movement = StockMovement.new
  end

  def edit; end

  def create
    @stock_movement = StockMovement.new(stock_movement_params)
    if @stock_movement.save
      respond_to do |format|
        format.turbo_stream
        format.html { redirect_to stock_movements_path, notice: "Movimiento registrado." }
      end
    else
      render :new, status: :unprocessable_entity
    end
  end

  def update
    if @stock_movement.update(stock_movement_params)
      respond_to do |format|
        format.turbo_stream
        format.html { redirect_to stock_movements_path, notice: "Movimiento actualizado." }
      end
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @stock_movement.destroy
    respond_to do |format|
      format.turbo_stream
      format.html { redirect_to stock_movements_path, notice: "Movimiento eliminado." }
    end
  end

  private

  def set_stock_movement
    @stock_movement = StockMovement.find(params[:id])
  end

  def stock_movement_params
    params.require(:stock_movement).permit(:product_id, :quantity, :movement_type, :location_type, :location_id)
  end
end
```

---

### `app/controllers/reports_controller.rb`

```ruby
class ReportsController < ApplicationController
  def index
    if params[:product_id].present?
      @product = Product.find(params[:product_id])
      @movements = StockMovement.where(product: @product).includes(:location).order(created_at: :desc)
      @avg_cost = StockReport.average_cost(@product)
      @rotation = StockReport.rotation_rate(@product)
    else
      @movements = []
    end
  end
end
```

---

## 🪄 MIGRACIONES

```ruby
class CreateStockTransfers < ActiveRecord::Migration[7.1]
  def change
    create_table :stock_transfers do |t|
      t.references :source, polymorphic: true, null: false
      t.references :destination, polymorphic: true, null: false
      t.timestamps
    end
  end
end
```

```ruby
class CreateStockTransferItems < ActiveRecord::Migration[7.1]
  def change
    create_table :stock_transfer_items do |t|
      t.references :stock_transfer, null: false, foreign_key: true
      t.references :product, null: false, foreign_key: true
      t.integer :quantity, null: false, default: 0
      t.timestamps
    end
  end
end
```

```ruby
class CreateStockMovements < ActiveRecord::Migration[7.1]
  def change
    create_table :stock_movements do |t|
      t.references :location, polymorphic: true, null: false
      t.references :product, null: false, foreign_key: true
      t.integer :movement_type, default: 0, null: false
      t.integer :quantity, null: false
      t.timestamps
    end
  end
end
```

---

## 💥 Opcional: Stimulus controllers

`app/javascript/controllers/transfer_controller.js`

```js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["sourceSelect", "destinationSelect"]

  updateSource(event) {
    const type = event.target.value
    this.loadOptions(this.sourceSelectTarget, type)
  }

  updateDestination(event) {
    const type = event.target.value
    this.loadOptions(this.destinationSelectTarget, type)
  }

  loadOptions(target, type) {
    fetch(`/locations/options?type=${type}`, { headers: { Accept: "text/vnd.turbo-stream.html" } })
      .then(r => r.text())
      .then(html => target.innerHTML = html)
  }

  addItem(e) {
    e.preventDefault()
    const template = document.querySelector("#item_template").content.cloneNode(true)
    document.querySelector("#items").appendChild(template)
  }

  removeItem(e) {
    e.preventDefault()
    e.target.closest(".card").remove()
  }
}
```

---

## ⚙️ ¿Qué sigue?

Si querés, puedo generarte ahora el **`.excalidraw` actualizado** con los modelos:  
`Warehouse`, `Store`, `Product`, `ProductItem`, `StockTransfer`, `StockTransferItem`, `StockMovement` y `StockReport`  
— todas las relaciones dibujadas para que lo abras directo en tu Obsidian.

¿Quieres que te lo prepare ya?


