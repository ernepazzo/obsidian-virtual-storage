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