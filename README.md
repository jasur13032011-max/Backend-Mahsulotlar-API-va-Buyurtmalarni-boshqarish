1. Papka strukturasi
flask-shop-api/
│
├── app.py
├── requirements.txt
├── README.md
└── .gitignore
2. requirements.txt
Flask==3.1.2
Flask-JWT-Extended==4.7.1
Werkzeug==3.1.3
3. app.py
from flask import Flask, request, jsonify
from flask_jwt_extended import (
    JWTManager,
    create_access_token,
    jwt_required,
    get_jwt_identity
)
from werkzeug.security import generate_password_hash, check_password_hash
import sqlite3
from datetime import timedelta

app = Flask(__name__)

# JWT sozlamalari
app.config["JWT_SECRET_KEY"] = "my-super-secret-key-change-this"
app.config["JWT_ACCESS_TOKEN_EXPIRES"] = timedelta(hours=24)

jwt = JWTManager(app)

DATABASE = "database.db"


# =========================
# DATABASE
# =========================

def get_db():
    conn = sqlite3.connect(DATABASE)
    conn.row_factory = sqlite3.Row
    return conn


def init_db():
    conn = get_db()
    cursor = conn.cursor()

    # Users
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT NOT NULL UNIQUE,
            email TEXT NOT NULL UNIQUE,
            password TEXT NOT NULL
        )
    """)

    # Products
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS products (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            price REAL NOT NULL,
            description TEXT
        )
    """)

    # Cart
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS cart (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER NOT NULL,
            product_id INTEGER NOT NULL,
            quantity INTEGER NOT NULL DEFAULT 1,
            FOREIGN KEY (user_id) REFERENCES users(id),
            FOREIGN KEY (product_id) REFERENCES products(id)
        )
    """)

    # Test products
    cursor.execute("SELECT COUNT(*) FROM products")
    product_count = cursor.fetchone()[0]

    if product_count == 0:
        products = [
            ("Gaming Mouse", 25.99, "Professional gaming mouse"),
            ("Mechanical Keyboard", 59.99, "RGB mechanical keyboard"),
            ("Gaming Headset", 39.99, "High quality gaming headset"),
            ("Mouse Pad", 15.50, "Large gaming mouse pad"),
            ("Webcam", 45.00, "Full HD webcam")
        ]

        cursor.executemany("""
            INSERT INTO products (name, price, description)
            VALUES (?, ?, ?)
        """, products)

    conn.commit()
    conn.close()


# =========================
# HOME
# =========================

@app.route("/")
def home():
    return jsonify({
        "message": "Flask Shop API ishlayapti!",
        "endpoints": {
            "register": "POST /api/register",
            "login": "POST /api/login",
            "products": "GET /api/products",
            "cart": "POST /api/cart"
        }
    })


# =========================
# REGISTER
# =========================

@app.route("/api/register", methods=["POST"])
def register():
    data = request.get_json()

    if not data:
        return jsonify({
            "error": "JSON ma'lumot yuboring"
        }), 400

    username = data.get("username")
    email = data.get("email")
    password = data.get("password")

    if not username or not email or not password:
        return jsonify({
            "error": "username, email va password kerak"
        }), 400

    if len(password) < 6:
        return jsonify({
            "error": "Parol kamida 6 ta belgidan iborat bo'lishi kerak"
        }), 400

    conn = get_db()
    cursor = conn.cursor()

    # User mavjudligini tekshirish
    cursor.execute("""
        SELECT id FROM users
        WHERE username = ? OR email = ?
    """, (username, email))

    existing_user = cursor.fetchone()

    if existing_user:
        conn.close()

        return jsonify({
            "error": "Username yoki email allaqachon mavjud"
        }), 409

    hashed_password = generate_password_hash(password)

    cursor.execute("""
        INSERT INTO users (username, email, password)
        VALUES (?, ?, ?)
    """, (username, email, hashed_password))

    user_id = cursor.lastrowid

    conn.commit()
    conn.close()

    # JWT token
    access_token = create_access_token(identity=str(user_id))

    return jsonify({
        "message": "Foydalanuvchi muvaffaqiyatli ro'yxatdan o'tdi",
        "user": {
            "id": user_id,
            "username": username,
            "email": email
        },
        "access_token": access_token
    }), 201


# =========================
# LOGIN
# =========================

@app.route("/api/login", methods=["POST"])
def login():
    data = request.get_json()

    if not data:
        return jsonify({
            "error": "JSON ma'lumot yuboring"
        }), 400

    email = data.get("email")
    password = data.get("password")

    if not email or not password:
        return jsonify({
            "error": "email va password kerak"
        }), 400

    conn = get_db()
    cursor = conn.cursor()

    cursor.execute("""
        SELECT * FROM users
        WHERE email = ?
    """, (email,))

    user = cursor.fetchone()
    conn.close()

    if not user:
        return jsonify({
            "error": "Email yoki parol noto'g'ri"
        }), 401

    if not check_password_hash(user["password"], password):
        return jsonify({
            "error": "Email yoki parol noto'g'ri"
        }), 401

    access_token = create_access_token(
        identity=str(user["id"])
    )

    return jsonify({
        "message": "Login muvaffaqiyatli",
        "user": {
            "id": user["id"],
            "username": user["username"],
            "email": user["email"]
        },
        "access_token": access_token
    }), 200


# =========================
# PRODUCTS
# =========================

@app.route("/api/products", methods=["GET"])
def get_products():
    conn = get_db()
    cursor = conn.cursor()

    cursor.execute("""
        SELECT id, name, price, description
        FROM products
    """)

    products = cursor.fetchall()
    conn.close()

    product_list = []

    for product in products:
        product_list.append({
            "id": product["id"],
            "name": product["name"],
            "price": product["price"],
            "description": product["description"]
        })

    return jsonify({
        "count": len(product_list),
        "products": product_list
    }), 200


# =========================
# ADD TO CART
# =========================

@app.route("/api/cart", methods=["POST"])
@jwt_required()
def add_to_cart():

    # Token ichidagi user ID
    user_id = get_jwt_identity()

    data = request.get_json()

    if not data:
        return jsonify({
            "error": "JSON ma'lumot yuboring"
        }), 400

    product_id = data.get("product_id")
    quantity = data.get("quantity", 1)

    if not product_id:
        return jsonify({
            "error": "product_id kerak"
        }), 400

    if not isinstance(quantity, int) or quantity < 1:
        return jsonify({
            "error": "quantity 1 yoki undan katta bo'lishi kerak"
        }), 400

    conn = get_db()
    cursor = conn.cursor()

    # Product mavjudligini tekshirish
    cursor.execute("""
        SELECT * FROM products
        WHERE id = ?
    """, (product_id,))

    product = cursor.fetchone()

    if not product:
        conn.close()

        return jsonify({
            "error": "Mahsulot topilmadi"
        }), 404

    # Savatda shu product borligini tekshirish
    cursor.execute("""
        SELECT * FROM cart
        WHERE user_id = ? AND product_id = ?
    """, (user_id, product_id))

    existing_cart = cursor.fetchone()

    if existing_cart:

        new_quantity = existing_cart["quantity"] + quantity

        cursor.execute("""
            UPDATE cart
            SET quantity = ?
            WHERE id = ?
        """, (new_quantity, existing_cart["id"]))

    else:

        cursor.execute("""
            INSERT INTO cart (user_id, product_id, quantity)
            VALUES (?, ?, ?)
        """, (user_id, product_id, quantity))

    conn.commit()
    conn.close()

    return jsonify({
        "message": "Mahsulot savatga qo'shildi",
        "product": {
            "id": product["id"],
            "name": product["name"],
            "price": product["price"]
        },
        "quantity": quantity
    }), 201


# =========================
# VIEW CART
# =========================

@app.route("/api/cart", methods=["GET"])
@jwt_required()
def get_cart():

    user_id = get_jwt_identity()

    conn = get_db()
    cursor = conn.cursor()

    cursor.execute("""
        SELECT
            cart.id,
            products.id AS product_id,
            products.name,
            products.price,
            cart.quantity
        FROM cart
        JOIN products
        ON cart.product_id = products.id
        WHERE cart.user_id = ?
    """, (user_id,))

    cart_items = cursor.fetchall()
    conn.close()

    items = []

    total = 0

    for item in cart_items:

        item_total = item["price"] * item["quantity"]
        total += item_total

        items.append({
            "cart_id": item["id"],
            "product_id": item["product_id"],
            "name": item["name"],
            "price": item["price"],
            "quantity": item["quantity"],
            "total": item_total
        })

    return jsonify({
        "items": items,
        "total": total
    }), 200


# =========================
# ERROR HANDLERS
# =========================

@jwt.unauthorized_loader
def unauthorized_callback(error):
    return jsonify({
        "error": "Token kerak. Avval login qiling."
    }), 401


@jwt.invalid_token_loader
def invalid_token_callback(error):
    return jsonify({
        "error": "Token noto'g'ri yoki yaroqsiz"
    }), 401


@jwt.expired_token_loader
def expired_token_callback(jwt_header, jwt_payload):
    return jsonify({
        "error": "Token muddati tugagan. Qayta login qiling."
    }), 401


# =========================
# START
# =========================

if __name__ == "__main__":
    init_db()

    app.run(
        debug=True,
        host="127.0.0.1",
        port=5000
    )
4. O‘rnatish

WebStorm yoki VS Code terminalini ochib:

python -m venv venv

Windowsda:

venv\Scripts\activate

Keyin:

pip install -r requirements.txt
5. Serverni ishga tushirish
python app.py

Shunday chiqadi:

* Running on http://127.0.0.1:5000

Brauzerdan:

http://127.0.0.1:5000

och.

6. Register qilish

Postman'da:

POST

http://127.0.0.1:5000/api/register

Body → raw → JSON:

{
    "username": "jasur",
    "email": "jasur@gmail.com",
    "password": "12345678"
}

Natijada token keladi:

{
    "message": "Foydalanuvchi muvaffaqiyatli ro'yxatdan o'tdi",
    "user": {
        "id": 1,
        "username": "jasur",
        "email": "jasur@gmail.com"
    },
    "access_token": "eyJ..."
}

Mana shu access_token juda muhim.

7. Login

POST

http://127.0.0.1:5000/api/login

Body:

{
    "email": "jasur@gmail.com",
    "password": "12345678"
}

Yana JWT token qaytadi.

8. Mahsulotlarni olish

GET

http://127.0.0.1:5000/api/products

Token kerak emas.

Natija:

{
    "count": 5,
    "products": [
        {
            "id": 1,
            "name": "Gaming Mouse",
            "price": 25.99,
            "description": "Professional gaming mouse"
        },
        {
            "id": 2,
            "name": "Mechanical Keyboard",
            "price": 59.99,
            "description": "RGB mechanical keyboard"
        }
    ]
}
9. Savatga mahsulot qo‘shish

POST

http://127.0.0.1:5000/api/cart

Headers:

Authorization: Bearer SIZNING_TOKENINGIZ

Body:

{
    "product_id": 1,
    "quantity": 2
}

Natija:

{
    "message": "Mahsulot savatga qo'shildi",
    "product": {
        "id": 1,
        "name": "Gaming Mouse",
        "price": 25.99
    },
    "quantity": 2
}

Token bermasdan yuborsang:

{
    "error": "Token kerak. Avval login qiling."
}

Demak topshiriqdagi asosiy talab bajariladi: savatga mahsulot faqat autentifikatsiyadan o'tgan foydalanuvchi tomonidan qo'shiladi.

10. Savatni ko‘rish

Qo‘shimcha qilib qo‘ydim:

GET

http://127.0.0.1:5000/api/cart

Header:

Authorization: Bearer SIZNING_TOKENINGIZ
11. .gitignore
venv/
__pycache__/
*.pyc
database.db
.env

database.db GitHub'ga chiqmaydi. Server birinchi ishga tushganda o‘zi yaratadi.

12. GitHub'ga yuklash

GitHub'da yangi public repository yarat:

flask-shop-api

Keyin terminal:

git init
git add .
git commit -m "Create Flask REST API with JWT authentication"
git branch -M main
git remote add origin https://github.com/SENING_USERNAME/flask-shop-api.git
git push -u origin main

Keyin topshiriqqa:

https://github.com/SENING_USERNAME/flask-shop-api

ni berasan.
