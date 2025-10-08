{
  "type": "excalidraw",
  "version": 2,
  "source": "ChatGPT - Warehouse ERD for Rails 8",
  "elements": [
    {
      "type": "rectangle",
      "x": 100,
      "y": 100,
      "width": 220,
      "height": 180,
      "id": "warehouse",
      "backgroundColor": "#F9F5FF",
      "strokeColor": "#7C3AED",
      "label": "Warehouse\n\nid: bigint (PK)\nname: string\nlocation: string\ncreated_at: timestamptz\nupdated_at: timestamptz"
    },
    {
      "type": "rectangle",
      "x": 400,
      "y": 100,
      "width": 220,
      "height": 180,
      "id": "store",
      "backgroundColor": "#F0F9FF",
      "strokeColor": "#0284C7",
      "label": "Store\n\nid: bigint (PK)\nname: string\nlocation: string\ncreated_at: timestamptz\nupdated_at: timestamptz"
    },
    {
      "type": "rectangle",
      "x": 250,
      "y": 350,
      "width": 220,
      "height": 200,
      "id": "product",
      "backgroundColor": "#ECFDF5",
      "strokeColor": "#059669",
      "label": "Product\n\nid: bigint (PK)\nname: string\nsku: string\nunit: string\ncreated_at: timestamptz\nupdated_at: timestamptz"
    },
    {
      "type": "rectangle",
      "x": 250,
      "y": 600,
      "width": 260,
      "height": 220,
      "id": "product_item",
      "backgroundColor": "#FEF9C3",
      "strokeColor": "#CA8A04",
      "label": "ProductItem\n\nid: bigint (PK)\nproduct_id: bigint (FK)\nlocation_type: string\nlocation_id: bigint\ncost_price: decimal(10,2)\nsale_price: decimal(10,2)\nquantity: decimal(10,2)\nentry_date: timestamptz\ncreated_at: timestamptz\nupdated_at: timestamptz"
    },
    {
      "type": "rectangle",
      "x": 50,
      "y": 900,
      "width": 320,
      "height": 200,
      "id": "stock_transfer",
      "backgroundColor": "#FFF1F2",
      "strokeColor": "#DC2626",
      "label": "StockTransfer\n\nid: bigint (PK)\norigin_type: string\norigin_id: bigint\ndestination_type: string\ndestination_id: bigint\ntransfer_date: timestamptz\nstatus: string\ncreated_at: timestamptz\nupdated_at: timestamptz"
    },
    {
      "type": "rectangle",
      "x": 400,
      "y": 900,
      "width": 320,
      "height": 200,
      "id": "stock_movement",
      "backgroundColor": "#F0FDF4",
      "strokeColor": "#16A34A",
      "label": "StockMovement\n\nid: bigint (PK)\nproduct_item_id: bigint (FK)\nmovement_type: string ('in'/'out')\nquantity: decimal(10,2)\nreason: text\nmovement_date: timestamptz\ncreated_at: timestamptz\nupdated_at: timestamptz"
    }
  ],
  "connections": [
    {
      "from": "product",
      "to": "product_item",
      "label": "1:N"
    },
    {
      "from": "warehouse",
      "to": "product_item",
      "label": "1:N"
    },
    {
      "from": "store",
      "to": "product_item",
      "label": "1:N"
    },
    {
      "from": "product_item",
      "to": "stock_movement",
      "label": "1:N"
    },
    {
      "from": "warehouse",
      "to": "stock_transfer",
      "label": "1:N"
    },
    {
      "from": "store",
      "to": "stock_transfer",
      "label": "1:N"
    }
  ],
  "appState": {
    "viewBackgroundColor": "#ffffff",
    "gridSize": 10
  }
}