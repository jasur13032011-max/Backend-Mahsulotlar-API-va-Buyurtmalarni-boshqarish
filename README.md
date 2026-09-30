# Backend-Mahsulotlar-API-va-Buyurtmalarni-boshqarish
Ha, bu keyingi loyiha **Flask backend** bo‘ladi. Senga topshiriqni bajaradigan minimal, ishlaydigan variantni beraman.

### Loyiha strukturasi

```text
backend/
├── app.py
├── models.py
├── requirements.txt
└── database.db
```

### 1. `requirements.txt`

```txt
Flask
Flask-SQLAlchemy
```

### 2. `models.py`

```python
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()


class Product(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    price = db.Column(db.Float, nullable=False)
    description = db.Column(db.String(255))


class Order(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    product_id = db.Column(db.Integer, nullable=False)
    quantity = db.Column(db.Integer, nullable=False)
    customer_name = db.Column(db.String(100), nullable=False)
```

### 3. `app.py`

```python
from flask import Flask, jsonify, request
from models import db, Product, Order

app = Flask(__name__)

app.config["SQLALCHEMY_DATABASE_URI"] = "sqlite:///database.db"
app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False

db.init_app(app)


with app.app_context():
    db.create_all()

    # Test mahsulotlar
    if Product.query.count() == 0:
        products = [
            Product(
                name="Keyboard",
                price=25.99,
                description="Mechanical keyboard"
            ),
            Product(
                name="Mouse",
                price=19.99,
                description="Gaming mouse"
            ),
            Product(
                name="Headphones",
                price=39.99,
                description="Gaming headphones"
            )
        ]

        db.session.add_all(products)
        db.session.commit()


# GET /api/products
@app.route("/api/products", methods=["GET"])
def get_products():

    products = Product.query.all()

    result = []

    for product in products:
        result.append({
            "id": product.id,
            "name": product.name,
            "price": product.price,
            "description": product.description
        })

    return jsonify(result)


# POST /api/orders
@app.route("/api/orders", methods=["POST"])
def create_order():

    data = request.get_json()

    if not data:
        return jsonify({
            "success": False,
            "message": "JSON ma'lumot yuborilmadi"
        }), 400

    product_id = data.get("product_id")
    quantity = data.get("quantity")
    customer_name = data.get("customer_name")

    if not product_id or not quantity or not customer_name:
        return jsonify({
            "success": False,
            "message": "Barcha maydonlarni to'ldiring"
        }), 400

    product = Product.query.get(product_id)

    if not product:
        return jsonify({
            "success": False,
            "message": "Mahsulot topilmadi"
        }), 404

    if quantity <= 0:
        return jsonify({
            "success": False,
            "message": "Quantity 0 dan katta bo'lishi kerak"
        }), 400

    order = Order(
        product_id=product_id,
        quantity=quantity,
        customer_name=customer_name
    )

    db.session.add(order)
    db.session.commit()

    return jsonify({
        "success": True,
        "message": "Buyurtma muvaffaqiyatli qabul qilindi",
        "order": {
            "id": order.id,
            "product_id": order.product_id,
            "quantity": order.quantity,
            "customer_name": order.customer_name
        }
    }), 201


if __name__ == "__main__":
    app.run(debug=True)
```

### Ishga tushirish

Terminalda:

```bash
python -m venv venv
```

Windows:

```bash
venv\Scripts\activate
```

Keyin:

```bash
pip install -r requirements.txt
```

Va:

```bash
python app.py
```

Server:

```text
http://127.0.0.1:5000
```

### `/api/products`

Brauzerda:

```text
http://127.0.0.1:5000/api/products
```

JSON chiqadi:

```json
[
  {
    "id": 1,
    "name": "Keyboard",
    "price": 25.99,
    "description": "Mechanical keyboard"
  },
  {
    "id": 2,
    "name": "Mouse",
    "price": 19.99,
    "description": "Gaming mouse"
  }
]
```

### `/api/orders`

POST qilib quyidagi JSON yuboriladi:

```json
{
  "product_id": 1,
  "quantity": 2,
  "customer_name": "Jasur"
}
```

Javob:

```json
{
  "success": true,
  "message": "Buyurtma muvaffaqiyatli qabul qilindi",
  "order": {
    "id": 1,
    "product_id": 1,
    "quantity": 2,
    "customer_name": "Jasur"
  }
}
```

Shu variant bilan topshiriqdagi asosiy **2 ta API endpoint + Product modeli + Order modeli + SQLite saqlash** talablari bajariladi.
