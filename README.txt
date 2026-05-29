Структура проекта:
Папки:
__pycache__
images
imports - demo_items_5000_generated
resources

Файлы:
create_tables.py ( таблички)
database.py ( бдшка питон)
demo_exam.db (создается автоматом после запуска - бд)
import_data.py (для импорта данных)
main.py (основа)
README.md ( руководство пользователя)
schema.sql ( основа бд)
tests_protocol_template.md (шаблон протокола тестирования)
er_diagram.mmd ( диаграмма через dbever)
authorization_flow.mmd ( блок схема авторизации)


КОД:

create_tables.py:

import sqlite3
from database import DB_PATH, SCHEMA_PATH

def create_tables():
    conn = sqlite3.connect(DB_PATH)
    with open(SCHEMA_PATH, "r", encoding="utf-8") as f:
        sql_script = f.read()
    conn.executescript(sql_script)
    conn.commit()
    conn.close()
    print("Таблицы успешно созданы!")

if __name__ == "__main__":
    create_tables()


database.py:

import sqlite3
from pathlib import Path

DB_PATH = Path(__file__).resolve().parent / "demo_exam.db"
SCHEMA_PATH = Path(__file__).resolve().parent / "schema.sql"


def get_connection():
    connection = sqlite3.connect(DB_PATH)
    connection.row_factory = sqlite3.Row
    connection.execute("PRAGMA foreign_keys = ON")
    return connection


def initialize_database():
    with get_connection() as connection:
        connection.executescript(SCHEMA_PATH.read_text(encoding="utf-8"))
        seed_database(connection)


def seed_database(connection):
    roles = [
        ("admin", "Администратор"),
        ("manager", "Менеджер"),
        ("client", "Авторизованный клиент"),
    ]
    connection.executemany(
        "INSERT OR IGNORE INTO Roles(name, title) VALUES (?, ?)",
        roles,
    )

    role_ids = {
        row["name"]: row["id"]
        for row in connection.execute("SELECT id, name FROM Roles")
    }

    users = [
        (role_ids["admin"], "admin", "admin", "Иванов Иван Иванович"),
        (role_ids["manager"], "manager", "manager", "Петрова Мария Сергеевна"),
        (role_ids["client"], "client", "client", "Сидоров Алексей Павлович"),
    ]
    connection.executemany(
        """
        INSERT OR IGNORE INTO Users(role_id, login, password, full_name)
        VALUES (?, ?, ?, ?)
        """,
        users,
    )

    categories = ["Кроссовки", "Ботинки", "Туфли"]
    manufacturers = ["ОбувьПро", "StepLine", "Комфорт"]
    suppliers = ["ООО Поставка", "ИП Складов", "АО Ресурс"]

    connection.executemany(
        "INSERT OR IGNORE INTO Categories(name) VALUES (?)",
        [(item,) for item in categories],
    )
    connection.executemany(
        "INSERT OR IGNORE INTO Manufacturers(name) VALUES (?)",
        [(item,) for item in manufacturers],
    )
    connection.executemany(
        "INSERT OR IGNORE INTO Suppliers(name) VALUES (?)",
        [(item,) for item in suppliers],
    )

    if connection.execute("SELECT COUNT(*) FROM Products").fetchone()[0] == 0:
        products = [
            ("A001", "Кроссовки городские", 1, "Повседневная обувь", 1, 1, 3500.00, "пара", 10, 5, ""),
            ("A002", "Ботинки зимние", 2, "Теплая обувь", 2, 2, 6200.00, "пара", 0, 0, ""),
            ("A003", "Туфли классические", 3, "Офисная обувь", 3, 3, 4800.50, "пара", 7, 20, ""),
        ]
        connection.executemany(
            """
            INSERT INTO Products(
                article, name, category_id, description, manufacturer_id,
                supplier_id, price, unit, quantity, discount, image_path
            )
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            """,
            products,
        )

    if connection.execute("SELECT COUNT(*) FROM Orders").fetchone()[0] == 0:
        connection.execute(
            """
            INSERT INTO Orders(status, pickup_point, order_date, issue_date)
            VALUES (?, ?, ?, ?)
            """,
            ("Новый", "Пункт выдачи №1", "2026-01-10", "2026-01-15"),
        )
        order_id = connection.execute("SELECT last_insert_rowid()").fetchone()[0]
        product_id = connection.execute("SELECT id FROM Products WHERE article = 'A001'").fetchone()[0]
        connection.execute(
            "INSERT INTO OrderProducts(order_id, product_id, quantity) VALUES (?, ?, ?)",
            (order_id, product_id, 1),
        )


def authenticate(login, password):
    with get_connection() as connection:
        return connection.execute(
            """
            SELECT
                Users.id,
                Users.login,
                Users.full_name,
                Roles.name AS role_name,
                Roles.title AS role_title
            FROM Users
            JOIN Roles ON Roles.id = Users.role_id
            WHERE Users.login = ? AND Users.password = ?
            """,
            (login, password),
        ).fetchone()


def fetch_products(search_text="", supplier_name="Все поставщики", sort_mode="Без сортировки"):
    query = """
        SELECT
            Products.id,
            Products.article,
            Products.name,
            Categories.name AS category,
            Products.description,
            Manufacturers.name AS manufacturer,
            Suppliers.name AS supplier,
            Products.price,
            Products.unit,
            Products.quantity,
            Products.discount,
            Products.image_path
        FROM Products
        JOIN Categories ON Categories.id = Products.category_id
        JOIN Manufacturers ON Manufacturers.id = Products.manufacturer_id
        JOIN Suppliers ON Suppliers.id = Products.supplier_id
        WHERE 1 = 1
    """
    params = []

    if search_text:
        like = f"%{search_text.lower()}%"
        query += """
            AND (
                LOWER(Products.article) LIKE ?
                OR LOWER(Products.name) LIKE ?
                OR LOWER(Categories.name) LIKE ?
                OR LOWER(Products.description) LIKE ?
                OR LOWER(Manufacturers.name) LIKE ?
                OR LOWER(Suppliers.name) LIKE ?
                OR LOWER(Products.unit) LIKE ?
            )
        """
        params.extend([like] * 7)

    if supplier_name and supplier_name != "Все поставщики":
        query += " AND Suppliers.name = ?"
        params.append(supplier_name)

    if sort_mode == "Количество ↑":
        query += " ORDER BY Products.quantity ASC, Products.name ASC"
    elif sort_mode == "Количество ↓":
        query += " ORDER BY Products.quantity DESC, Products.name ASC"
    else:
        query += " ORDER BY Products.name ASC"

    with get_connection() as connection:
        return connection.execute(query, params).fetchall()


def fetch_suppliers():
    with get_connection() as connection:
        rows = connection.execute("SELECT name FROM Suppliers ORDER BY name").fetchall()
    return ["Все поставщики"] + [row["name"] for row in rows]


def fetch_lookup(table_name):
    allowed = {"Categories", "Manufacturers", "Suppliers"}
    if table_name not in allowed:
        raise ValueError("Недопустимое имя справочника.")
    with get_connection() as connection:
        return connection.execute(f"SELECT id, name FROM {table_name} ORDER BY name").fetchall()


def fetch_product_by_id(product_id):
    with get_connection() as connection:
        return connection.execute(
            "SELECT * FROM Products WHERE id = ?",
            (product_id,),
        ).fetchone()


def save_product(data, product_id=None):
    with get_connection() as connection:
        if product_id is None:
            connection.execute(
                """
                INSERT INTO Products(
                    article, name, category_id, description, manufacturer_id,
                    supplier_id, price, unit, quantity, discount, image_path
                )
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
                """,
                (
                    data["article"],
                    data["name"],
                    data["category_id"],
                    data["description"],
                    data["manufacturer_id"],
                    data["supplier_id"],
                    data["price"],
                    data["unit"],
                    data["quantity"],
                    data["discount"],
                    data["image_path"],
                ),
            )
        else:
            connection.execute(
                """
                UPDATE Products
                SET
                    article = ?,
                    name = ?,
                    category_id = ?,
                    description = ?,
                    manufacturer_id = ?,
                    supplier_id = ?,
                    price = ?,
                    unit = ?,
                    quantity = ?,
                    discount = ?,
                    image_path = ?
                WHERE id = ?
                """,
                (
                    data["article"],
                    data["name"],
                    data["category_id"],
                    data["description"],
                    data["manufacturer_id"],
                    data["supplier_id"],
                    data["price"],
                    data["unit"],
                    data["quantity"],
                    data["discount"],
                    data["image_path"],
                    product_id,
                ),
            )


def product_used_in_orders(product_id):
    with get_connection() as connection:
        count = connection.execute(
            "SELECT COUNT(*) FROM OrderProducts WHERE product_id = ?",
            (product_id,),
        ).fetchone()[0]
    return count > 0


def delete_product(product_id):
    with get_connection() as connection:
        connection.execute("DELETE FROM Products WHERE id = ?", (product_id,))


def fetch_orders():
    with get_connection() as connection:
        return connection.execute(
            """
            SELECT
                Orders.id,
                Orders.status,
                Orders.pickup_point,
                Orders.order_date,
                Orders.issue_date,
                GROUP_CONCAT(Products.article || ' (' || OrderProducts.quantity || ')', ', ') AS products
            FROM Orders
            LEFT JOIN OrderProducts ON OrderProducts.order_id = Orders.id
            LEFT JOIN Products ON Products.id = OrderProducts.product_id
            GROUP BY Orders.id
            ORDER BY Orders.id DESC
            """
        ).fetchall()


def fetch_order_by_id(order_id):
    with get_connection() as connection:
        order = connection.execute(
            "SELECT * FROM Orders WHERE id = ?",
            (order_id,),
        ).fetchone()
        product = connection.execute(
            "SELECT product_id, quantity FROM OrderProducts WHERE order_id = ? LIMIT 1",
            (order_id,),
        ).fetchone()
    return order, product


def fetch_product_articles():
    with get_connection() as connection:
        return connection.execute("SELECT id, article, name FROM Products ORDER BY article").fetchall()


def save_order(data, order_id=None):
    with get_connection() as connection:
        if order_id is None:
            connection.execute(
                """
                INSERT INTO Orders(status, pickup_point, order_date, issue_date)
                VALUES (?, ?, ?, ?)
                """,
                (
                    data["status"],
                    data["pickup_point"],
                    data["order_date"],
                    data["issue_date"],
                ),
            )
            order_id = connection.execute("SELECT last_insert_rowid()").fetchone()[0]
        else:
            connection.execute(
                """
                UPDATE Orders
                SET status = ?, pickup_point = ?, order_date = ?, issue_date = ?
                WHERE id = ?
                """,
                (
                    data["status"],
                    data["pickup_point"],
                    data["order_date"],
                    data["issue_date"],
                    order_id,
                ),
            )
            connection.execute("DELETE FROM OrderProducts WHERE order_id = ?", (order_id,))

        connection.execute(
            "INSERT INTO OrderProducts(order_id, product_id, quantity) VALUES (?, ?, ?)",
            (order_id, data["product_id"], data["quantity"]),
        )


def delete_order(order_id):
    with get_connection() as connection:
        connection.execute("DELETE FROM Orders WHERE id = ?", (order_id,))



import_data.py:

import csv
from pathlib import Path
import database

IMPORT_DIR = Path(__file__).resolve().parent / "imports"

def read_csv(path):
    raw = path.read_text(encoding="utf-8-sig")
    dialect = csv.Sniffer().sniff(raw[:2048], delimiters=";,")
    return list(csv.DictReader(raw.splitlines(), dialect=dialect))

def get_or_create(connection, table_name, name):
    if not name:
        name = "Не указано"
    row = connection.execute(f"SELECT id FROM {table_name} WHERE name = ?", (name,)).fetchone()
    if row:
        return row["id"]
    connection.execute(f"INSERT INTO {table_name}(name) VALUES (?)", (name,))
    return connection.execute(f"SELECT last_insert_rowid()").fetchone()[0]

def import_products(file_name):
    path = IMPORT_DIR / file_name
    rows = read_csv(path)

    with database.get_connection() as connection:
        for index, row in enumerate(rows):
            # Генерация уникального артикля
            article = f"PRD{index+1:05d}"
            name = row.get("name") or row.get("Наименование") or "Не указано"
            category = row.get("category") or row.get("Категория") or "Не указано"
            manufacturer = row.get("manufacturer") or row.get("Производитель") or "Не указано"
            supplier = row.get("supplier") or row.get("Поставщик") or "Не указано"
            description = row.get("description") or row.get("Описание") or ""
            unit = row.get("unit") or row.get("Единица") or "шт."
            price = float(str(row.get("price") or 0).replace(",", "."))
            quantity = int(row.get("quantity") or 0)
            discount = int(row.get("discount") or 0)
            image_path = row.get("image_path") or "resources/picture.png"

            category_id = get_or_create(connection, "Categories", category)
            manufacturer_id = get_or_create(connection, "Manufacturers", manufacturer)
            supplier_id = get_or_create(connection, "Suppliers", supplier)

            connection.execute(
                """
                INSERT OR REPLACE INTO Products(
                    article, name, category_id, description, manufacturer_id,
                    supplier_id, price, unit, quantity, discount, image_path
                )
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
                """,
                (
                    article,
                    name,
                    category_id,
                    description,
                    manufacturer_id,
                    supplier_id,
                    price,
                    unit,
                    quantity,
                    discount,
                    image_path,
                ),
            )

if __name__ == "__main__":
    database.initialize_database()
    import_products("demo_items_5000_generated.csv")
    print("Импорт завершён успешно!")


maim.py:
import shutil
import tkinter as tk
from pathlib import Path
from tkinter import filedialog, messagebox, ttk

import database


APP_DIR = Path(__file__).resolve().parent
IMAGES_DIR = APP_DIR / "images"
IMAGES_DIR.mkdir(exist_ok=True)


class App(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("Обувь - система учета товаров")
        self.geometry("1180x720")
        self.minsize(980, 620)

        self.current_user = None
        self.product_form_window = None
        self.order_form_window = None

        database.initialize_database()
        self.show_login()

    def clear(self):
        for widget in self.winfo_children():
            widget.destroy()

    def show_login(self):
        self.current_user = None
        self.clear()
        LoginFrame(self).pack(fill="both", expand=True)

    def show_products(self, user):
        self.current_user = user
        self.clear()
        ProductsFrame(self, user).pack(fill="both", expand=True)

    def show_orders(self):
        self.clear()
        OrdersFrame(self, self.current_user).pack(fill="both", expand=True)


class LoginFrame(ttk.Frame):
    def __init__(self, master):
        super().__init__(master, padding=30)

        container = ttk.Frame(self, padding=20)
        container.place(relx=0.5, rely=0.5, anchor="center")

        ttk.Label(container, text="Вход в систему", font=("Arial", 18, "bold")).grid(
            row=0, column=0, columnspan=2, pady=(0, 25)
        )

        ttk.Label(container, text="Логин").grid(row=1, column=0, sticky="w", pady=5)
        self.login_entry = ttk.Entry(container, width=35)
        self.login_entry.grid(row=1, column=1, pady=5)

        ttk.Label(container, text="Пароль").grid(row=2, column=0, sticky="w", pady=5)
        self.password_entry = ttk.Entry(container, width=35, show="*")
        self.password_entry.grid(row=2, column=1, pady=5)

        ttk.Button(container, text="Войти", command=self.login).grid(
            row=3, column=0, columnspan=2, sticky="ew", pady=(18, 5)
        )
        ttk.Button(container, text="Войти как гость", command=self.login_as_guest).grid(
            row=4, column=0, columnspan=2, sticky="ew"
        )

        ttk.Label(
            container,
            text="Тестовые учетные записи: admin/admin, manager/manager, client/client",
            foreground="gray",
        ).grid(row=5, column=0, columnspan=2, pady=(15, 0))

        self.login_entry.focus()

    def login(self):
        login = self.login_entry.get().strip()
        password = self.password_entry.get().strip()

        if not login or not password:
            messagebox.showwarning(
                "Пустые поля",
                "Введите логин и пароль. После заполнения полей повторите вход.",
            )
            return

        user = database.authenticate(login, password)
        if user is None:
            messagebox.showerror(
                "Ошибка авторизации",
                "Пользователь с указанным логином и паролем не найден. Проверьте данные и повторите попытку.",
            )
            return

        self.master.show_products(dict(user))

    def login_as_guest(self):
        guest = {
            "id": None,
            "login": "guest",
            "full_name": "Гость",
            "role_name": "guest",
            "role_title": "Гость",
        }
        self.master.show_products(guest)


class ProductsFrame(ttk.Frame):
    def __init__(self, master, user):
        super().__init__(master, padding=10)
        self.user = user
        self.role_name = user["role_name"]
        self.can_manage_products = self.role_name == "admin"
        self.can_filter = self.role_name in ("admin", "manager")

        self.search_var = tk.StringVar()
        self.supplier_var = tk.StringVar(value="Все поставщики")
        self.sort_var = tk.StringVar(value="Без сортировки")

        self.build_header()
        self.build_controls()
        self.build_table()
        self.load_products()

        self.search_var.trace_add("write", lambda *_: self.load_products())

    def build_header(self):
        header = ttk.Frame(self)
        header.pack(fill="x", pady=(0, 10))

        ttk.Label(
            header,
            text="Список товаров",
            font=("Arial", 16, "bold"),
        ).pack(side="left")

        ttk.Label(
            header,
            text=f"{self.user['role_title']}: {self.user['full_name']}",
        ).pack(side="right", padx=10)

        ttk.Button(header, text="Выйти", command=self.master.show_login).pack(side="right")

    def build_controls(self):
        controls = ttk.Frame(self)
        controls.pack(fill="x", pady=(0, 10))

        if self.can_filter:
            ttk.Label(controls, text="Поиск").pack(side="left")
            ttk.Entry(controls, textvariable=self.search_var, width=35).pack(side="left", padx=5)

            ttk.Label(controls, text="Поставщик").pack(side="left", padx=(15, 0))
            suppliers = database.fetch_suppliers()
            supplier_combo = ttk.Combobox(
                controls,
                textvariable=self.supplier_var,
                values=suppliers,
                state="readonly",
                width=25,
            )
            supplier_combo.pack(side="left", padx=5)
            supplier_combo.bind("<<ComboboxSelected>>", lambda _: self.load_products())

            ttk.Label(controls, text="Сортировка").pack(side="left", padx=(15, 0))
            sort_combo = ttk.Combobox(
                controls,
                textvariable=self.sort_var,
                values=["Без сортировки", "Количество ↑", "Количество ↓"],
                state="readonly",
                width=18,
            )
            sort_combo.pack(side="left", padx=5)
            sort_combo.bind("<<ComboboxSelected>>", lambda _: self.load_products())

        if self.can_manage_products:
            ttk.Button(controls, text="Добавить товар", command=self.open_add_product).pack(side="right", padx=5)

        if self.role_name in ("admin", "manager"):
            ttk.Button(controls, text="Заказы", command=self.master.show_orders).pack(side="right", padx=5)

    def build_table(self):
        columns = (
            "article",
            "name",
            "category",
            "description",
            "manufacturer",
            "supplier",
            "price",
            "unit",
            "quantity",
            "discount",
        )
        self.tree = ttk.Treeview(self, columns=columns, show="headings", height=22)

        headings = {
            "article": "Артикул",
            "name": "Наименование",
            "category": "Категория",
            "description": "Описание",
            "manufacturer": "Производитель",
            "supplier": "Поставщик",
            "price": "Цена",
            "unit": "Ед.",
            "quantity": "Остаток",
            "discount": "Скидка",
        }
        widths = {
            "article": 90,
            "name": 180,
            "category": 120,
            "description": 210,
            "manufacturer": 130,
            "supplier": 130,
            "price": 120,
            "unit": 60,
            "quantity": 80,
            "discount": 80,
        }

        for column in columns:
            self.tree.heading(column, text=headings[column])
            self.tree.column(column, width=widths[column], anchor="w")

        self.tree.tag_configure("discount", background="#2E8B57", foreground="white")
        self.tree.tag_configure("empty_stock", background="#ADD8E6")

        scrollbar = ttk.Scrollbar(self, orient="vertical", command=self.tree.yview)
        self.tree.configure(yscrollcommand=scrollbar.set)

        self.tree.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")

        if self.can_manage_products:
            self.tree.bind("<Double-1>", self.open_edit_product)

    def format_price(self, price, discount):
        price = float(price)
        discount = int(discount)
        if discount > 0:
            final_price = price * (1 - discount / 100)
            return f"{price:.2f} -> {final_price:.2f}"
        return f"{price:.2f}"

    def load_products(self):
        for item in self.tree.get_children():
            self.tree.delete(item)

        products = database.fetch_products(
            search_text=self.search_var.get().strip() if self.can_filter else "",
            supplier_name=self.supplier_var.get() if self.can_filter else "Все поставщики",
            sort_mode=self.sort_var.get() if self.can_filter else "Без сортировки",
        )

        for product in products:
            tags = []
            if product["quantity"] == 0:
                tags.append("empty_stock")
            elif product["discount"] > 15:
                tags.append("discount")

            self.tree.insert(
                "",
                "end",
                iid=str(product["id"]),
                values=(
                    product["article"],
                    product["name"],
                    product["category"],
                    product["description"],
                    product["manufacturer"],
                    product["supplier"],
                    self.format_price(product["price"], product["discount"]),
                    product["unit"],
                    product["quantity"],
                    f"{product['discount']}%",
                ),
                tags=tags,
            )

    def open_add_product(self):
        if self.master.product_form_window is not None and self.master.product_form_window.winfo_exists():
            self.master.product_form_window.focus()
            messagebox.showwarning(
                "Окно уже открыто",
                "Завершите добавление или редактирование текущего товара перед открытием нового окна.",
            )
            return

        self.master.product_form_window = ProductForm(self.master, self, None)

    def open_edit_product(self, _event=None):
        selected = self.tree.selection()
        if not selected:
            return

        if self.master.product_form_window is not None and self.master.product_form_window.winfo_exists():
            self.master.product_form_window.focus()
            messagebox.showwarning(
                "Окно уже открыто",
                "Нельзя одновременно редактировать несколько товаров. Закройте текущее окно редактирования.",
            )
            return

        product_id = int(selected[0])
        self.master.product_form_window = ProductForm(self.master, self, product_id)


class ProductForm(tk.Toplevel):
    def __init__(self, app, products_frame, product_id=None):
        super().__init__(app)
        self.app = app
        self.products_frame = products_frame
        self.product_id = product_id
        self.title("Добавление товара" if product_id is None else "Редактирование товара")
        self.geometry("560x620")
        self.resizable(False, False)
        self.protocol("WM_DELETE_WINDOW", self.close)

        self.image_path_var = tk.StringVar()
        self.entries = {}
        self.lookup_maps = {}

        self.build_form()

        if product_id is not None:
            self.load_product(product_id)

    def build_form(self):
        frame = ttk.Frame(self, padding=15)
        frame.pack(fill="both", expand=True)

        row = 0
        if self.product_id is not None:
            ttk.Label(frame, text="ID").grid(row=row, column=0, sticky="w", pady=4)
            self.id_entry = ttk.Entry(frame, width=42)
            self.id_entry.grid(row=row, column=1, sticky="ew", pady=4)
            self.id_entry.insert(0, str(self.product_id))
            self.id_entry.config(state="readonly")
            row += 1

        fields = [
            ("article", "Артикул"),
            ("name", "Наименование"),
            ("description", "Описание"),
            ("price", "Цена"),
            ("unit", "Единица измерения"),
            ("quantity", "Количество на складе"),
            ("discount", "Действующая скидка"),
        ]

        for key, label in fields:
            ttk.Label(frame, text=label).grid(row=row, column=0, sticky="w", pady=4)
            entry = ttk.Entry(frame, width=42)
            entry.grid(row=row, column=1, sticky="ew", pady=4)
            self.entries[key] = entry
            row += 1

        for key, label, table in [
            ("category_id", "Категория", "Categories"),
            ("manufacturer_id", "Производитель", "Manufacturers"),
            ("supplier_id", "Поставщик", "Suppliers"),
        ]:
            ttk.Label(frame, text=label).grid(row=row, column=0, sticky="w", pady=4)
            values = database.fetch_lookup(table)
            self.lookup_maps[key] = {item["name"]: item["id"] for item in values}
            combo = ttk.Combobox(frame, values=list(self.lookup_maps[key].keys()), state="readonly", width=39)
            combo.grid(row=row, column=1, sticky="ew", pady=4)
            self.entries[key] = combo
            row += 1

        ttk.Label(frame, text="Фото товара").grid(row=row, column=0, sticky="w", pady=4)
        image_frame = ttk.Frame(frame)
        image_frame.grid(row=row, column=1, sticky="ew", pady=4)
        ttk.Entry(image_frame, textvariable=self.image_path_var, width=30).pack(side="left", fill="x", expand=True)
        ttk.Button(image_frame, text="Выбрать", command=self.choose_image).pack(side="left", padx=(5, 0))
        row += 1

        buttons = ttk.Frame(frame)
        buttons.grid(row=row, column=0, columnspan=2, sticky="ew", pady=(20, 0))

        ttk.Button(buttons, text="Сохранить", command=self.save).pack(side="left")
        if self.product_id is not None:
            ttk.Button(buttons, text="Удалить", command=self.delete).pack(side="left", padx=10)
        ttk.Button(buttons, text="Назад", command=self.close).pack(side="right")

    def choose_image(self):
        source_path = filedialog.askopenfilename(
            title="Выберите изображение товара",
            filetypes=[
                ("Изображения", "*.png *.jpg *.jpeg *.gif"),
                ("Все файлы", "*.*"),
            ],
        )
        if not source_path:
            return

        source = Path(source_path)
        destination = IMAGES_DIR / source.name
        counter = 1
        while destination.exists():
            destination = IMAGES_DIR / f"{source.stem}_{counter}{source.suffix}"
            counter += 1

        shutil.copy2(source, destination)
        self.image_path_var.set(str(destination.relative_to(APP_DIR)))

    def load_product(self, product_id):
        product = database.fetch_product_by_id(product_id)
        if product is None:
            messagebox.showerror("Ошибка", "Товар не найден в базе данных.")
            self.close()
            return

        for key in ["article", "name", "description", "price", "unit", "quantity", "discount"]:
            self.entries[key].insert(0, str(product[key]))

        for key in ["category_id", "manufacturer_id", "supplier_id"]:
            reverse_map = {value: name for name, value in self.lookup_maps[key].items()}
            self.entries[key].set(reverse_map.get(product[key], ""))

        self.image_path_var.set(product["image_path"] or "")

    def collect_data(self):
        try:
            article = self.entries["article"].get().strip()
            name = self.entries["name"].get().strip()
            description = self.entries["description"].get().strip()
            unit = self.entries["unit"].get().strip() or "шт."
            price = float(self.entries["price"].get().replace(",", "."))
            quantity = int(self.entries["quantity"].get())
            discount = int(self.entries["discount"].get())

            if not article or not name:
                raise ValueError("Артикул и наименование обязательны для заполнения.")
            if price < 0:
                raise ValueError("Цена не может быть отрицательной.")
            if quantity < 0:
                raise ValueError("Количество на складе не может быть отрицательным.")
            if discount < 0 or discount > 100:
                raise ValueError("Скидка должна быть в диапазоне от 0 до 100.")

            data = {
                "article": article,
                "name": name,
                "description": description,
                "price": price,
                "unit": unit,
                "quantity": quantity,
                "discount": discount,
                "image_path": self.image_path_var.get().strip(),
            }

            for key in ["category_id", "manufacturer_id", "supplier_id"]:
                selected_name = self.entries[key].get().strip()
                if not selected_name:
                    raise ValueError("Выберите категорию, производителя и поставщика.")
                data[key] = self.lookup_maps[key][selected_name]

            return data

        except ValueError as error:
            messagebox.showerror(
                "Ошибка заполнения формы",
                f"{error}\nИсправьте данные в форме и повторите сохранение.",
            )
            return None

    def save(self):
        data = self.collect_data()
        if data is None:
            return

        try:
            database.save_product(data, self.product_id)
        except Exception as error:
            messagebox.showerror(
                "Ошибка сохранения",
                f"Не удалось сохранить товар.\nПричина: {error}",
            )
            return

        messagebox.showinfo("Сохранение", "Данные товара успешно сохранены.")
        self.products_frame.load_products()
        self.close()

    def delete(self):
        if self.product_id is None:
            return

        if database.product_used_in_orders(self.product_id):
            messagebox.showerror(
                "Удаление запрещено",
                "Этот товар присутствует в заказе, поэтому удалить его нельзя.",
            )
            return

        confirmed = messagebox.askyesno(
            "Подтверждение удаления",
            "Удалить выбранный товар? Это действие нельзя отменить.",
        )
        if not confirmed:
            return

        try:
            database.delete_product(self.product_id)
        except Exception as error:
            messagebox.showerror("Ошибка удаления", f"Не удалось удалить товар.\nПричина: {error}")
            return

        self.products_frame.load_products()
        self.close()

    def close(self):
        self.app.product_form_window = None
        self.destroy()


class OrdersFrame(ttk.Frame):
    def __init__(self, master, user):
        super().__init__(master, padding=10)
        self.user = user
        self.can_edit = user["role_name"] == "admin"
        self.build_header()
        self.build_table()
        self.load_orders()

    def build_header(self):
        header = ttk.Frame(self)
        header.pack(fill="x", pady=(0, 10))

        ttk.Label(header, text="Заказы", font=("Arial", 16, "bold")).pack(side="left")
        ttk.Button(header, text="Назад к товарам", command=lambda: self.master.show_products(self.user)).pack(side="right")

        if self.can_edit:
            ttk.Button(header, text="Добавить заказ", command=self.open_add_order).pack(side="right", padx=5)

    def build_table(self):
        columns = ("id", "status", "pickup_point", "order_date", "issue_date", "products")
        self.tree = ttk.Treeview(self, columns=columns, show="headings", height=24)

        headings = {
            "id": "ID",
            "status": "Статус",
            "pickup_point": "Пункт выдачи",
            "order_date": "Дата заказа",
            "issue_date": "Дата выдачи",
            "products": "Товары",
        }

        for column in columns:
            self.tree.heading(column, text=headings[column])
            self.tree.column(column, width=150 if column != "products" else 300)

        self.tree.pack(fill="both", expand=True)

        if self.can_edit:
            self.tree.bind("<Double-1>", self.open_edit_order)

    def load_orders(self):
        for item in self.tree.get_children():
            self.tree.delete(item)

        for order in database.fetch_orders():
            self.tree.insert(
                "",
                "end",
                iid=str(order["id"]),
                values=(
                    order["id"],
                    order["status"],
                    order["pickup_point"],
                    order["order_date"],
                    order["issue_date"],
                    order["products"] or "",
                ),
            )

    def open_add_order(self):
        if self.master.order_form_window is not None and self.master.order_form_window.winfo_exists():
            self.master.order_form_window.focus()
            return
        self.master.order_form_window = OrderForm(self.master, self, None)

    def open_edit_order(self, _event=None):
        selected = self.tree.selection()
        if not selected:
            return

        if self.master.order_form_window is not None and self.master.order_form_window.winfo_exists():
            self.master.order_form_window.focus()
            return

        self.master.order_form_window = OrderForm(self.master, self, int(selected[0]))


class OrderForm(tk.Toplevel):
    def __init__(self, app, orders_frame, order_id=None):
        super().__init__(app)
        self.app = app
        self.orders_frame = orders_frame
        self.order_id = order_id
        self.title("Добавление заказа" if order_id is None else "Редактирование заказа")
        self.geometry("520x380")
        self.resizable(False, False)
        self.protocol("WM_DELETE_WINDOW", self.close)

        self.entries = {}
        self.product_map = {}
        self.build_form()

        if order_id is not None:
            self.load_order(order_id)

    def build_form(self):
        frame = ttk.Frame(self, padding=15)
        frame.pack(fill="both", expand=True)

        fields = [
            ("status", "Статус заказа"),
            ("pickup_point", "Адрес пункта выдачи"),
            ("order_date", "Дата заказа"),
            ("issue_date", "Дата выдачи"),
            ("quantity", "Количество"),
        ]

        row = 0
        ttk.Label(frame, text="Артикул").grid(row=row, column=0, sticky="w", pady=4)
        articles = database.fetch_product_articles()
        self.product_map = {f"{item['article']} - {item['name']}": item["id"] for item in articles}
        product_combo = ttk.Combobox(frame, values=list(self.product_map.keys()), state="readonly", width=42)
        product_combo.grid(row=row, column=1, sticky="ew", pady=4)
        self.entries["product"] = product_combo
        row += 1

        for key, label in fields:
            ttk.Label(frame, text=label).grid(row=row, column=0, sticky="w", pady=4)
            entry = ttk.Entry(frame, width=45)
            entry.grid(row=row, column=1, sticky="ew", pady=4)
            self.entries[key] = entry
            row += 1

        buttons = ttk.Frame(frame)
        buttons.grid(row=row, column=0, columnspan=2, sticky="ew", pady=(20, 0))
        ttk.Button(buttons, text="Сохранить", command=self.save).pack(side="left")
        if self.order_id is not None:
            ttk.Button(buttons, text="Удалить", command=self.delete).pack(side="left", padx=10)
        ttk.Button(buttons, text="Назад", command=self.close).pack(side="right")

        if self.order_id is None:
            self.entries["status"].insert(0, "Новый")
            self.entries["order_date"].insert(0, "2026-01-10")
            self.entries["issue_date"].insert(0, "2026-01-15")
            self.entries["quantity"].insert(0, "1")

    def load_order(self, order_id):
        order, product = database.fetch_order_by_id(order_id)
        if order is None:
            messagebox.showerror("Ошибка", "Заказ не найден в базе данных.")
            self.close()
            return

        for key in ["status", "pickup_point", "order_date", "issue_date"]:
            self.entries[key].insert(0, str(order[key]))

        if product:
            self.entries["quantity"].insert(0, str(product["quantity"]))
            reverse_product_map = {value: name for name, value in self.product_map.items()}
            self.entries["product"].set(reverse_product_map.get(product["product_id"], ""))
        else:
            self.entries["quantity"].insert(0, "1")

    def collect_data(self):
        try:
            product_name = self.entries["product"].get().strip()
            if not product_name:
                raise ValueError("Выберите товар по артикулу.")

            status = self.entries["status"].get().strip()
            pickup_point = self.entries["pickup_point"].get().strip()
            order_date = self.entries["order_date"].get().strip()
            issue_date = self.entries["issue_date"].get().strip()
            quantity = int(self.entries["quantity"].get())

            if not status or not pickup_point or not order_date or not issue_date:
                raise ValueError("Все поля заказа должны быть заполнены.")
            if quantity <= 0:
                raise ValueError("Количество товара в заказе должно быть больше нуля.")

            return {
                "product_id": self.product_map[product_name],
                "status": status,
                "pickup_point": pickup_point,
                "order_date": order_date,
                "issue_date": issue_date,
                "quantity": quantity,
            }
        except ValueError as error:
            messagebox.showerror(
                "Ошибка заполнения формы",
                f"{error}\nИсправьте данные и повторите сохранение.",
            )
            return None

    def save(self):
        data = self.collect_data()
        if data is None:
            return

        try:
            database.save_order(data, self.order_id)
        except Exception as error:
            messagebox.showerror("Ошибка сохранения", f"Не удалось сохранить заказ.\nПричина: {error}")
            return

        self.orders_frame.load_orders()
        self.close()

    def delete(self):
        confirmed = messagebox.askyesno(
            "Подтверждение удаления",
            "Удалить выбранный заказ? Это действие нельзя отменить.",
        )
        if not confirmed:
            return

        try:
            database.delete_order(self.order_id)
        except Exception as error:
            messagebox.showerror("Ошибка удаления", f"Не удалось удалить заказ.\nПричина: {error}")
            return

        self.orders_frame.load_orders()
        self.close()

    def close(self):
        self.app.order_form_window = None
        self.destroy()


if __name__ == "__main__":
    app = App()
    app.mainloop()


schema.sql

PRAGMA foreign_keys = ON;

CREATE TABLE IF NOT EXISTS Roles (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE,
    title TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS Users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    role_id INTEGER NOT NULL,
    login TEXT NOT NULL UNIQUE,
    password TEXT NOT NULL,
    full_name TEXT NOT NULL,
    FOREIGN KEY (role_id) REFERENCES Roles(id)
        ON UPDATE CASCADE
        ON DELETE RESTRICT
);

CREATE TABLE IF NOT EXISTS Categories (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE
);

CREATE TABLE IF NOT EXISTS Manufacturers (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE
);

CREATE TABLE IF NOT EXISTS Suppliers (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE
);

CREATE TABLE IF NOT EXISTS Products (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    article TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    category_id INTEGER NOT NULL,
    description TEXT NOT NULL DEFAULT '',
    manufacturer_id INTEGER NOT NULL,
    supplier_id INTEGER NOT NULL,
    price REAL NOT NULL CHECK(price >= 0),
    unit TEXT NOT NULL DEFAULT 'шт.',
    quantity INTEGER NOT NULL DEFAULT 0 CHECK(quantity >= 0),
    discount INTEGER NOT NULL DEFAULT 0 CHECK(discount >= 0 AND discount <= 100),
    image_path TEXT,
    FOREIGN KEY (category_id) REFERENCES Categories(id)
        ON UPDATE CASCADE
        ON DELETE RESTRICT,
    FOREIGN KEY (manufacturer_id) REFERENCES Manufacturers(id)
        ON UPDATE CASCADE
        ON DELETE RESTRICT,
    FOREIGN KEY (supplier_id) REFERENCES Suppliers(id)
        ON UPDATE CASCADE
        ON DELETE RESTRICT
);

CREATE TABLE IF NOT EXISTS Orders (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    status TEXT NOT NULL,
    pickup_point TEXT NOT NULL,
    order_date TEXT NOT NULL,
    issue_date TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS OrderProducts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    order_id INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL DEFAULT 1 CHECK(quantity > 0),
    FOREIGN KEY (order_id) REFERENCES Orders(id)
        ON UPDATE CASCADE
        ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES Products(id)
        ON UPDATE CASCADE
        ON DELETE RESTRICT
);



template...

# Протокол тестирования

| № | Проверка | Входные данные | Ожидаемый результат | Фактический результат | Статус |
|---|---|---|---|---|---|
| 1 | Вход администратора | admin / admin | Открывается список товаров, доступны добавление/редактирование/удаление |  |  |
| 2 | Вход менеджера | manager / manager | Открывается список товаров, доступны поиск, фильтр, сортировка, заказы |  |  |
| 3 | Вход клиента | client / client | Открывается список товаров без администрирования |  |  |
| 4 | Вход гостя | Кнопка «Войти как гость» | Открывается список товаров без поиска, фильтра и редактирования |  |  |
| 5 | Неверный пароль | admin / 123 | Появляется сообщение об ошибке авторизации |  |  |
| 6 | Поиск товара | Ввести часть названия товара | Список обновляется в реальном времени |  |  |
| 7 | Фильтр по поставщику | Выбрать поставщика | Показаны товары выбранного поставщика |  |  |
| 8 | Сброс фильтра | Выбрать «Все поставщики» | Отображаются все товары |  |  |
| 9 | Сортировка по количеству | Количество ↑ / ↓ | Товары сортируются по остатку |  |  |
| 10 | Добавление товара | Заполнить форму корректными данными | Новый товар появляется в списке |  |  |
| 11 | Отрицательная цена | Цена = -10 | Появляется сообщение об ошибке |  |  |
| 12 | Удаление товара в заказе | Удалить товар из заказа | Удаление запрещено, показано сообщение |  |  |
| 13 | Просмотр заказов менеджером | manager / manager → «Заказы» | Открывается список заказов без кнопки добавления |  |  |
| 14 | Добавление заказа администратором | admin / admin → «Заказы» → «Добавить заказ» | Заказ сохраняется и появляется в списке |  |  |


Руководство пользователя
1. Общие сведения

Программа предназначена для учёта товаров, управления заказами и пользователей. Поддерживаются три роли:

Администратор — полный доступ к товарам, заказам, пользователям.
Менеджер — может фильтровать и просматривать товары и заказы.
Гость — просмотр только списка товаров.

Интерфейс выполнен на Tkinter, база данных хранится в SQLite (demo_exam.db).

2. Запуск программы
Убедитесь, что Python установлен.
Распакуйте проект в любую папку.
Убедитесь, что установлены необходимые библиотеки:
pip install pandas
Если база пустая, выполните импорт данных:
python import_data.py
Запустите программу:
python main.py
3. Авторизация
Введите логин и пароль для входа:
admin/admin — администратор
manager/manager — менеджер
client/client — клиент
Для работы без авторизации нажмите кнопку «Войти как гость».
4. Работа с товарами
После входа открывается окно «Список товаров».
Таблица отображает: Артикул, Наименование, Категорию, Описание, Производителя, Поставщика, Цену, Единицу, Количество, Скидку.
Фильтры и поиск доступны для админа и менеджера:
Поиск по наименованию
Фильтр по поставщику
Сортировка по количеству
Добавление товара (только администратор):
Нажмите кнопку «Добавить товар» → заполните поля → сохраните.
Редактирование товара (только администратор):
Дважды кликните на товар → внесите изменения → сохраните.
5. Работа с заказами
Для админа и менеджера доступна кнопка «Заказы».
Таблица заказов отображает: номер, статус, пункт выдачи, дату заказа, дату выдачи.
Добавление и редактирование заказов аналогично товарам.
6. Примечания
Все изменения сохраняются в базе demo_exam.db.
Фото товаров выводится по пути, указанному в базе; при отсутствии файла используется заглушка resources/picture.png.
Для тестирования используйте таблицу тест-кейсов из tests_protocol_template.md.

