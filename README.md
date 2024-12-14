from datetime import datetime
import datetime
from PyQt6.QtGui import QIntValidator
import requests
import openpyxl
import re
from openpyxl.styles import Font
from openpyxl.styles import Alignment
import sys
import os
from PyQt6.QtWidgets import (QApplication,QWidget,QLabel,QLineEdit,QPushButton,QVBoxLayout,QHBoxLayout,QComboBox,QGridLayout,QMessageBox,QScrollArea,QToolButton,QFormLayout,QRadioButton,QButtonGroup)
from PyQt6.QtCore import Qt, pyqtSignal
from PyQt6.QtGui import QPixmap, QIcon
import sqlite3

CREATE_PUBLISHER_TABLE = '''
CREATE TABLE IF NOT EXISTS publisher(
    id_publisher INTEGER PRIMARY KEY AUTOINCREMENT,
    name_publisher TEXT
)
'''
CREATE_AUTHOR_TABLE = '''
CREATE TABLE IF NOT EXISTS author(
    id_author INTEGER PRIMARY KEY AUTOINCREMENT,
    name_author TEXT,
    country TEXT
)
'''
CREATE_GENRE_TABLE = '''
CREATE TABLE IF NOT EXISTS genre(
    id_genre INTEGER PRIMARY KEY AUTOINCREMENT,
    name_game TEXT
)
'''
CREATE_BOOKS_TABLE = '''
CREATE TABLE IF NOT EXISTS books(
    id_book INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT,
    id_author INTEGER,
    id_genre INTEGER,
    id_publisher INTEGER,
    publication_year INTEGER,
    price REAL,
    add_features TEXT,
    FOREIGN KEY (id_publisher) REFERENCES publisher (id_publisher),
    FOREIGN KEY (id_author) REFERENCES author (id_author),
    FOREIGN KEY (id_genre) REFERENCES genre (id_genre)
)
'''
CREATE_SHOPPINGCART_TABLE = '''
CREATE TABLE IF NOT EXISTS shoppingcart (
    id_shoppingcart INTEGER PRIMARY KEY AUTOINCREMENT,
    id_customer INTEGER,
    id_book INTEGER,
    amount_cart REAL,
    FOREIGN KEY (id_customer) REFERENCES users (id_customer),
    FOREIGN KEY (id_book) REFERENCES books (id_book)
)
'''
CREATE_ORDER_TABLE = '''
CREATE TABLE IF NOT EXISTS orders (
id_order INTEGER PRIMARY KEY AUTOINCREMENT,
id_shoppingcart INTEGER,
order_date TEXT,
town TEXT,
street TEXT,
house TEXT,
flat INTEGER,
payment_method TEXT DEFAULT 'Оплата при получении',
FOREIGN KEY (id_shoppingcart) REFERENCES shoppingcart (id_shoppingcart)
)'''
CREATE_USER_TABLE = '''
CREATE TABLE IF NOT EXISTS users(
id_customer  INTEGER PRIMARY KEY,
name TEXT,
login TEXT,
password TEXT,
role TEXT NOT NULL
)
'''
def connect_db():
    return sqlite3.connect('shop.db')
# Функция для создания всех таблиц
def create_tables():
    conn = connect_db()
    cursor = conn.cursor()

    cursor.execute(CREATE_PUBLISHER_TABLE)
    cursor.execute(CREATE_AUTHOR_TABLE)
    cursor.execute(CREATE_GENRE_TABLE)
    cursor.execute(CREATE_BOOKS_TABLE)
    cursor.execute(CREATE_SHOPPINGCART_TABLE)
    cursor.execute(CREATE_ORDER_TABLE)
    cursor.execute(CREATE_USER_TABLE)

    conn.commit()
    conn.close()
def insert_initial_data():
    conn = connect_db()
    cursor = conn.cursor()

    def insert_if_not_exists(table, column, value, additional_columns=None, additional_values=None):
        """
        Вставляет данные в таблицу, если записи с указанным значением не существует.
        """
        additional_columns = additional_columns or []
        additional_values = additional_values or []
        columns = [column] + additional_columns
        placeholders = ["?"] * len(columns)

        query_check = f"SELECT 1 FROM {table} WHERE {column} = ?"
        query_insert = f"INSERT INTO {table} ({', '.join(columns)}) VALUES ({', '.join(placeholders)})"
        cursor.execute(query_check, [value])
        if not cursor.fetchone():
            cursor.execute(query_insert, [value] + additional_values)

    def insert_book_if_not_exists(title, author, genre, publisher, year, price, features=None):
        query_check = """
            SELECT 1 FROM books 
            WHERE title = ? 
              AND id_author = (SELECT id_author FROM author WHERE name_author = ?)
              AND id_genre = (SELECT id_genre FROM genre WHERE name_game = ?)
              AND id_publisher = (SELECT id_publisher FROM publisher WHERE name_publisher = ?)
        """
        query_insert = """
            INSERT INTO books (title, id_author, id_genre, id_publisher, publication_year, price, add_features)
            VALUES (
                ?, 
                (SELECT id_author FROM author WHERE name_author = ?),
                (SELECT id_genre FROM genre WHERE name_game = ?),
                (SELECT id_publisher FROM publisher WHERE name_publisher = ?),
                ?, ?, ?
            )
        """
        cursor.execute(query_check, [title, author, genre, publisher])
        if not cursor.fetchone():
            cursor.execute(query_insert, [title, author, genre, publisher, year, price, features])

    try:
        # Вставка издателей
        publishers = ["Neoclassic", "Азбука", "Эксмо"]
        for publisher in publishers:
            insert_if_not_exists("publisher", "name_publisher", publisher)

        # Вставка авторов
        authors = [
            ("Пушкин Александр Сергеевич", "Россия"),
            ("Чак Паланик", "США"),
            ("Льюис Кэрролл", "Великобритания"),
            ("Антуан де Сент-Экзюпери", "Франция"),
            ("Джордж Клейсон", "США"),
            ("Коллинз Уилки", "Великобритания"),
            ("Леру Гастон", "Франция"),
        ]
        for name, country in authors:
            insert_if_not_exists("author", "name_author", name, ["country"], [country])

        # Вставка жанров
        genres = ["Роман в стихах", "Роман", "Сказка", "Повесть-сказка", "Финансовая притча"]
        for genre in genres:
            insert_if_not_exists("genre", "name_game", genre)

        # Вставка книг
        books = [
            ("Евгений Онегин", "Пушкин Александр Сергеевич", "Роман в стихах", "Neoclassic", 1823, 250.00, None),
            ("Бойцовский клуб", "Чак Паланик", "Роман", "Neoclassic", 1996, 350.00,None),
            ("Алиса в Стране чудес и Алиса в Зазеркалье", "Льюис Кэрролл", "Сказка", "Азбука", 1865, 350.00, None),
            ("Маленький принц", "Антуан де Сент-Экзюпери", "Повесть-сказка", "Эксмо", 1943, 350, None),
            ("Самый богатый человек в Вавилоне", "Джордж Клейсон", "Финансовая притча", "Эксмо", 1926, 450.00, None),
            ("Отель с приведениями", "Коллинз Уилки", "Роман", "Neoclassic", 1878, 300.00, None),
            ("Призрак оперы", "Леру Гастон", "Роман", "Neoclassic", 1909, 250.00, None),
            ("Капитанская дочка", "Пушкин Александр Сергеевич", "Роман в стихах","Neoclassic",1836,540,None),
            ("Евгений Онегин", "Пушкин Александр Сергеевич", "Роман в стихах", "Азбука", 1823, 280.00, None)
        ]
        for book in books:

            insert_book_if_not_exists(*book)

        conn.commit()
        print("Данные успешно добавлены!")
    except sqlite3.Error as e:
        print(f"Ошибка вставки данных: {e}")
        conn.rollback()
    finally:
        conn.close()
# Сначала создаем таблицы
create_tables()
# Затем добавляем начальные данные
insert_initial_data()

class ProductWidget(QWidget):
    addToCart = pyqtSignal(int)

    def __init__(self, name, price, image_path, description, genre, author, publisher, book_id):
        super().__init__()
        self.name = name
        self.book_id = book_id

        # Основной вертикальный layout
        main_layout = QVBoxLayout()

        # Контейнер для блока книги с улучшенным дизайном
        book_container = QWidget()
        book_layout = QVBoxLayout(book_container)
        book_layout.setAlignment(Qt.AlignmentFlag.AlignCenter)

        # Новый стиль для контейнера книги
        book_container.setStyleSheet(
            """
            QWidget {
                border: 1px solid #ddd;          /* Легкая серая рамка */
                border-radius: 12px;             /* Скругленные углы */
                padding: 15px;                   /* Внутренний отступ */
                background-color: #ffffff;      /* Белый фон */
                box-shadow: 0px 4px 10px rgba(0, 0, 0, 0.1); /* Тень для объема */
            }
            """
        )

        # Загрузка изображения
        pixmap = self.get_pixmap(image_path)
        if pixmap:
            label_image = QLabel()
            label_image.setPixmap(pixmap.scaled(300, 500, Qt.AspectRatioMode.KeepAspectRatio))  # Размер изображения
            label_image.setAlignment(Qt.AlignmentFlag.AlignCenter)
            book_layout.addWidget(label_image)
        else:
            placeholder_label = QLabel("Изображение не найдено")
            placeholder_label.setAlignment(Qt.AlignmentFlag.AlignCenter)
            book_layout.addWidget(placeholder_label)

        # Добавление информации о книге
        info_label = QLabel(f"<b>{name}</b> <i>({genre}, {author}, {publisher})</i>")
        info_label.setAlignment(Qt.AlignmentFlag.AlignCenter)
        info_label.setStyleSheet("font-size: 18px; color: #333;")
        book_layout.addWidget(info_label)

        # Отображение цены
        price_label = QLabel(f"<font color='green'>Цена: {price}₽</font>")
        price_label.setAlignment(Qt.AlignmentFlag.AlignCenter)
        price_label.setStyleSheet("font-size: 16px; font-weight: bold; margin-top: 10px;")
        book_layout.addWidget(price_label)

        # Описание книги
        desc_label = QLabel(f"<i>{description}</i>")
        desc_label.setAlignment(Qt.AlignmentFlag.AlignCenter)
        desc_label.setStyleSheet("font-size: 14px; color: #666; margin-top: 10px;")
        book_layout.addWidget(desc_label)

        # Добавление контейнера книги в основной layout
        main_layout.addWidget(book_container)

        # Кнопка добавления в корзину с улучшенным дизайном
        add_button = QPushButton("Добавить в корзину")
        add_button.setStyleSheet("""
            QPushButton {
                background-color: #4CAF50;       /* Зеленый фон */
                color: white;                    /* Белый текст */
                border-radius: 8px;              /* Скругленные углы */
                padding: 10px 20px;              /* Внутренние отступы */
                font-size: 16px;
                transition: background-color 0.3s; /* Плавный переход при наведении */
            }
            QPushButton:hover {
                background-color: #45a049;       /* Тёмно-зеленый при наведении */
            }
            QPushButton:pressed {
                background-color: #388e3c;       /* Еще более тёмный при нажатии */
            }
        """)
        add_button.setFixedSize(200, 40)
        add_button.clicked.connect(self.add_to_cart)
        main_layout.addWidget(add_button, alignment=Qt.AlignmentFlag.AlignCenter)

        self.setLayout(main_layout)

    def get_pixmap(self, image_path):
        """
        Загружает QPixmap из локального файла или URL.
        """
        if image_path.startswith("http://") or image_path.startswith("https://"):
            try:
                response = requests.get(image_path, timeout=5)
                response.raise_for_status()
                pixmap = QPixmap()
                pixmap.loadFromData(response.content)
                return pixmap
            except requests.RequestException as e:
                print(f"Ошибка загрузки изображения по URL: {e}")
        elif os.path.exists(image_path):
            return QPixmap(image_path)
        return None

    def add_to_cart(self):
        # Код для добавления товара в корзину
        conn = None
        try:
            conn = connect_db()
            cursor = conn.cursor()
            cursor.execute(
                "INSERT INTO shoppingcart (id_book, amount_cart) VALUES (?, ?)",
                (self.book_id, 1),
            )
            conn.commit()
            QMessageBox.information(self, "Успех", f"{self.name} добавлена в корзину!")
            self.addToCart.emit(self.book_id)
        except sqlite3.Error as e:
            QMessageBox.critical(self, "Ошибка базы данных", f"Ошибка добавления в корзину: {e}")
        finally:
            if conn:
                conn.close()



class AuthWindow(QWidget):
    def __init__(self, catalog_window):
        super().__init__()
        self.setWindowTitle("Авторизация/Регистрация")
        self.setFixedSize(400, 350)
        self.catalog_window = catalog_window

        self.setStyleSheet("""
            QWidget {
                border: 1px solid #ddd;
                border-radius: 12px;
                padding: 15px;
                background-color: #ffffff;
                box-shadow: 0px 4px 10px rgba(0, 0, 0, 0.1);
                font-family: Arial, sans-serif;
            }
            QLineEdit {
                border: 1px solid #ccc;
                border-radius: 8px;
                padding: 8px;
                font-size: 14px;
            }
            QPushButton {
                background-color: #0078d4;
                color: #fff;
                border: none;
                border-radius: 8px;
                padding: 10px 15px;
                font-size: 14px;
            }
            QPushButton:hover {
                background-color: #005a9e;
            }
            QPushButton:disabled {
                background-color: #ccc;
                color: #666;
            }
            QRadioButton {
                font-size: 14px;
                padding: 5px;
            }
            QLabel {
                font-size: 14px;
                font-weight: bold;
                color: #333;
            }
        """)

        # Радиокнопки для выбора роли
        self.user_radio = QRadioButton("Пользователь")
        self.user_radio.setChecked(True)  # Устанавливаем, чтобы пользователь был выбран по умолчанию
        self.admin_radio = QRadioButton("Администратор")

        self.role_group = QButtonGroup()
        self.role_group.addButton(self.user_radio)
        self.role_group.addButton(self.admin_radio)

        # Обработчик изменения роли
        self.user_radio.toggled.connect(self.update_ui_for_role)

        # Поля ввода
        self.name_input = QLineEdit()
        self.name_input.setPlaceholderText("Введите имя (только для регистрации)")

        self.login_input = QLineEdit()
        self.login_input.setPlaceholderText("Введите логин")

        self.password_input = QLineEdit()
        self.password_input.setPlaceholderText("Введите пароль")
        self.password_input.setEchoMode(QLineEdit.EchoMode.Password)

        # Кнопки авторизации и регистрации
        self.auth_button = QPushButton("Войти")
        self.auth_button.clicked.connect(self.login_user)

        self.register_button = QPushButton("Зарегистрироваться")
        self.register_button.clicked.connect(self.register_user)

        # Компоновка виджетов
        role_layout = QHBoxLayout()
        role_layout.addWidget(self.user_radio)
        role_layout.addWidget(self.admin_radio)

        layout = QFormLayout()
        layout.addRow("Роль:", role_layout)
        layout.addRow("Имя:", self.name_input)
        layout.addRow("Логин:", self.login_input)
        layout.addRow("Пароль:", self.password_input)

        button_layout = QHBoxLayout()
        button_layout.addWidget(self.auth_button)
        button_layout.addWidget(self.register_button)

        main_layout = QVBoxLayout()
        main_layout.addLayout(layout)
        main_layout.addLayout(button_layout)

        self.setLayout(main_layout)
        self.update_ui_for_role()

    def update_ui_for_role(self):
        """Обновляет интерфейс в зависимости от выбранной роли."""
        if self.admin_radio.isChecked():
            self.name_input.setVisible(False)  # Поле "Имя" скрывается для администратора
            self.register_button.setVisible(False)  # Убираем кнопку регистрации
        else:
            self.name_input.setVisible(True)
            self.register_button.setVisible(True)

    def login_user(self):
        login = self.login_input.text().strip()
        password = self.password_input.text().strip()
        role = "admin" if self.admin_radio.isChecked() else "user"

        if not login or not password:
            QMessageBox.warning(self, "Ошибка", "Заполните все поля!")
            return

        if role == "admin":
            static_admin_login = "KIRA"
            static_admin_password = "admin123"

            if login == static_admin_login and password == static_admin_password:
                QMessageBox.information(self, "Уведомление", "Авторизация успешна как администратор!")
                self.catalog_window.role = "Администратор"  # Передача роли администратора в каталог
                self.open_catalog()

            else:
                QMessageBox.warning(self, "Ошибка", "Неверный логин или пароль администратора!")
        else:
            conn = connect_db()
            cursor = conn.cursor()
            cursor.execute(
                "SELECT * FROM users WHERE login = ? AND password = ? AND role = ?",
                (login, password, role)
            )
            user = cursor.fetchone()
            conn.close()

            if user:
                QMessageBox.information(self, "Уведомление", "Авторизация успешна!")
                self.catalog_window.role = role  # Устанавливаем роль пользователя
                self.open_catalog()
            else:
                QMessageBox.warning(self, "Ошибка", "Неверный логин, пароль или роль!")

    def register_user(self):
        name = self.name_input.text().strip()
        login = self.login_input.text().strip()
        password = self.password_input.text().strip()

        if not name or not login or not password:
            QMessageBox.warning(self, "Ошибка", "Заполните все поля!")
            return

        if not re.match(r"^[a-zA-Zа-яА-ЯёЁ]+$", name):
            QMessageBox.warning(self, "Ошибка", "Имя может содержать только буквы!")
            return

        conn = connect_db()
        cursor = conn.cursor()

        try:
            cursor.execute(
                "INSERT INTO users (name, login, password, role) VALUES (?, ?, ?, ?)",
                (name, login, password, "user")
            )
            conn.commit()
            QMessageBox.information(self, "Уведомление", "Регистрация успешна! Теперь вы можете войти.")
        except sqlite3.IntegrityError:
            QMessageBox.warning(self, "Ошибка", "Логин уже занят!")
        except Exception as e:
            QMessageBox.critical(self, "Ошибка базы данных", f"Ошибка при регистрации: {e}")
        finally:
            conn.close()

    def open_catalog(self):
        self.catalog_window.show()
        self.close()


class CatalogWindow(QWidget):
    def __init__(self, image_dir="images",role = None):
        super().__init__()
        self.setWindowTitle("Книжная полочка")
        self.image_dir = image_dir
        self.role = role  # Здесь задается роль пользователя, по умолчанию - "user"
        self.products = []
        self.setFixedSize(1200, 700)

        # Лейбл для отображения количества книг
        self.book_count_label = QLabel()
        self.book_count_label.setStyleSheet("font-size: 16px; font-weight: bold; color: #333;")
        self.update_book_count()  # Обновление количества книг при создании окна

        # Поля для фильтрации по цене
        self.min_price_input = QLineEdit()
        self.min_price_input.setPlaceholderText("Мин. цена")
        self.min_price_input.setFixedWidth(80)
        self.min_price_input.setValidator(QIntValidator())  # Только числа

        self.max_price_input = QLineEdit()
        self.max_price_input.setPlaceholderText("Макс. цена")
        self.max_price_input.setFixedWidth(80)
        self.max_price_input.setValidator(QIntValidator())  # Только числа

        # Комбобоксы для фильтрации
        self.genre_combo = QComboBox()
        self.genre_combo.addItem("Все жанры")
        self.author_combo = QComboBox()
        self.author_combo.addItem("Все авторы")
        self.publisher_combo = QComboBox()
        self.publisher_combo.addItem("Все издательства")
        self.populate_comboboxes()

        # Поле поиска
        self.search_bar = QLineEdit()
        self.search_bar.setPlaceholderText("Поиск...")
        self.search_button = QToolButton()
        self.search_button.setIcon(QIcon(os.path.join(self.image_dir, "search_icon")))
        self.search_button.clicked.connect(self.filter_products)
        self.search_bar.returnPressed.connect(self.filter_products)

        # Область для отображения списка книг
        self.scroll_area = QScrollArea()
        self.scroll_area.setWidgetResizable(True)
        scroll_content = QWidget()
        self.scroll_layout = QVBoxLayout(scroll_content)
        self.scroll_area.setWidget(scroll_content)

        # Кнопка для перехода в корзину
        self.cart_button = QPushButton("Корзина")
        self.cart_button.clicked.connect(self.show_cart)

        # Расположение фильтров и кнопок в одну строку
        hbox = QHBoxLayout()
        hbox.addWidget(self.search_bar)
        hbox.addWidget(self.search_button)
        hbox.addWidget(QLabel("Жанр:"))
        hbox.addWidget(self.genre_combo)
        hbox.addWidget(QLabel("Автор:"))
        hbox.addWidget(self.author_combo)
        hbox.addWidget(QLabel("Издательство:"))
        hbox.addWidget(self.publisher_combo)
        hbox.addWidget(QLabel("Цена от:"))
        hbox.addWidget(self.min_price_input)
        hbox.addWidget(QLabel("до:"))
        hbox.addWidget(self.max_price_input)
        hbox.addWidget(self.cart_button, alignment=Qt.AlignmentFlag.AlignRight)

        # Основной макет
        vbox = QVBoxLayout()
        vbox.addWidget(self.book_count_label)  # Лейбл количества книг
        vbox.addLayout(hbox)
        vbox.addWidget(self.scroll_area)
        self.setLayout(vbox)

        # Инициализация продуктов и фильтров
        self.populate_products()
        self.filter_products()

    def count_books(self):
        """Получает количество книг из базы данных."""
        conn = connect_db()
        cursor = conn.cursor()
        cursor.execute("SELECT COUNT(*) FROM books")
        count = cursor.fetchone()[0]
        conn.close()
        return count

    def update_book_count(self):
        """Обновляет текст лейбла с количеством книг."""
        book_count = self.count_books()
        self.book_count_label.setText(f"Общее количество книг: {book_count}")

    def populate_comboboxes(self):
        """Заполняет комбобоксы жанров, авторов и издательств из базы данных."""
        conn = connect_db()
        cursor = conn.cursor()

        cursor.execute("SELECT name_game FROM genre")
        genres = [row[0] for row in cursor.fetchall()]
        self.genre_combo.clear()
        self.genre_combo.addItem("Все жанры")
        self.genre_combo.addItems(genres)

        cursor.execute("SELECT name_author FROM author")
        authors = [row[0] for row in cursor.fetchall()]
        self.author_combo.clear()
        self.author_combo.addItem("Все авторы")
        self.author_combo.addItems(authors)

        cursor.execute("SELECT name_publisher FROM publisher")
        publishers = [row[0] for row in cursor.fetchall()]
        self.publisher_combo.clear()
        self.publisher_combo.addItem("Все издательства")
        self.publisher_combo.addItems(publishers)

        conn.close()

        self.genre_combo.currentIndexChanged.connect(self.filter_products)
        self.author_combo.currentIndexChanged.connect(self.filter_products)
        self.publisher_combo.currentIndexChanged.connect(self.filter_products)

    def populate_products(self):
        """Заполняет список книг из базы данных."""
        conn = connect_db()
        cursor = conn.cursor()
        cursor.execute(
            "SELECT id_book, title, price, id_genre, id_author, id_publisher FROM books"
        )
        rows = cursor.fetchall()

        self.products = []
        for row in rows:
            book_id, title, price, genre_id, author_id, publisher_id = row
            cursor.execute("SELECT name_game FROM genre WHERE id_genre = ?", (genre_id,))
            genre_result = cursor.fetchone()
            genre = genre_result[0] if genre_result else "Unknown"

            cursor.execute("SELECT name_author FROM author WHERE id_author = ?", (author_id,))
            author_result = cursor.fetchone()
            author = author_result[0] if author_result else "Unknown"
            cursor.execute(
                "SELECT name_publisher FROM publisher WHERE id_publisher = ?", (publisher_id,)
            )
            publisher_result = cursor.fetchone()
            publisher = publisher_result[0] if publisher_result else "Unknown"

            image_path = os.path.join(self.image_dir, f"{book_id}.jpg")
            self.products.append(
                (title, price, image_path, "Бумажная версия книги", genre, author, publisher, book_id)
            )
        conn.close()

    def populate_scroll_area(self, products):
        """Заполняет область прокрутки виджетами продуктов."""
        for i in reversed(range(self.scroll_layout.count())):
            widget = self.scroll_layout.itemAt(i).widget()
            if widget:
                widget.deleteLater()

        for product in products:
            product_widget = self.create_product_widget(*product, role=self.role)  # передаем роль пользователя
            self.scroll_layout.addWidget(product_widget)

    def create_product_widget(self, name, price, image_path, description, genre, author, publisher, book_id, role):
        widget = ProductWidget(name, price, image_path, description, genre, author, publisher, book_id)
        self.delete_button = QPushButton("Удалить")  # Move outside the if statement
        self.delete_button.clicked.connect(lambda: self.remove_product(book_id))
        self.delete_button.setVisible(role == "admin")  # Control visibility here
        widget.layout().addWidget(self.delete_button)
        return widget

    def filter_products(self):
        """Фильтрует продукты на основе введенных значений."""
        search_text = self.search_bar.text().lower()
        genre_filter = self.genre_combo.currentText()
        author_filter = self.author_combo.currentText()
        publisher_filter = self.publisher_combo.currentText()
        min_price = self.min_price_input.text()
        max_price = self.max_price_input.text()

        filtered = []
        for product in self.products:
            title, price, image, desc, genre, author, publisher, book_id = product
            if min_price and price < int(min_price):
                continue
            if max_price and price > int(max_price):
                continue
            if (search_text in title.lower() or search_text in desc.lower()) and (
                    genre_filter == "Все жанры" or genre_filter == genre
            ) and (author_filter == "Все авторы" or author_filter == author) and (
                    publisher_filter == "Все издательства" or publisher_filter == publisher
            ):
                filtered.append(product)

        self.populate_scroll_area(filtered)

    def show_cart(self):
        """Открывает окно корзины."""
        self.cart_window = ShoppingCartWindow()
        self.cart_window.show()

    def remove_product(self, book_id):
        confirm = QMessageBox.question(
            self, "Подтверждение", "Вы уверены, что хотите удалить выбранный товар?",
            QMessageBox.Yes | QMessageBox.No
        )

        if confirm == QMessageBox.Yes:
            try:
                conn = connect_db()
                cursor = conn.cursor()
                cursor.execute("DELETE FROM books WHERE id_book = ?", (book_id,))
                conn.commit()
                QMessageBox.information(self, "Уведомление", "Товар успешно удалён!")
            except sqlite3.Error as e:
                QMessageBox.critical(self, "Ошибка базы данных", f"Ошибка удаления из базы данных: {e}")
            finally:
                if conn:
                    conn.close()

            # Перезаполнить каталог после удаления
            self.populate_products()
            self.filter_products()
        else:
            QMessageBox.information(self, "Отмена", "Удаление отменено.")


class ShoppingCartWindow(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Корзина")
        self.setFixedSize(400, 300)
        self.layout = QVBoxLayout()
        self.setLayout(self.layout)
        self.populate_cart()

    def populate_cart(self):
        # Очистить текущий layout перед добавлением новых элементов
        for i in reversed(range(self.layout.count())):
            widget = self.layout.itemAt(i).widget()
            if widget:
                widget.deleteLater()
        try:
            conn = connect_db()
            cursor = conn.cursor()

            cursor.execute(
                "SELECT books.title, shoppingcart.amount_cart, books.price FROM shoppingcart JOIN books ON shoppingcart.id_book = books.id_book"
            )
            cart_items = cursor.fetchall()
            total_price = 0
            for title, amount, price in cart_items:
                total_price += amount * price
                self.layout.addWidget(QLabel(f"{title} (x{amount}) - {price} за штуку"))
            self.layout.addWidget(QLabel(f"Итоговая сумма: {total_price}"))
            # Кнопка для оформления заказа
            checkout_button = QPushButton("Оформить заказ")
            checkout_button.clicked.connect(self.checkout)
            self.layout.addWidget(checkout_button)
            # Кнопка для очистки корзины
            clear_cart_button = QPushButton("Очистить корзину")
            clear_cart_button.clicked.connect(self.clear_cart)
            self.layout.addWidget(clear_cart_button)

        except sqlite3.Error as e:
            QMessageBox.critical(self, "Ошибка", f"Ошибка загрузки данных: {e}")
        finally:
            conn.close()

    def clear_cart(self):
        try:
            conn = connect_db()
            cursor = conn.cursor()
            cursor.execute("DELETE FROM shoppingcart")
            conn.commit()
            QMessageBox.information(self, "Уведомление", "Корзина успешно очищена.")
            self.populate_cart()  # Обновление содержимого корзины
        except sqlite3.Error as e:
            QMessageBox.critical(self, "Ошибка", f"Ошибка очистки корзины: {e}")
        finally:
            conn.close()

    def checkout(self):
        shopping_cart_id = 1  # Для примера, можно использовать реальный id корзины
        self.order_form = OrderForm(shopping_cart_id=shopping_cart_id)
        self.order_form.show()

class OrderForm(QWidget):
    def __init__(self, shopping_cart_id=None, parent=None):
        super().__init__(parent)  # Передаем родительский виджет (для доступа к корзине)
        self.db_path = "shop.db"
        self.shopping_cart_id = shopping_cart_id  # Получаем ID корзины
        self.initUI()

    def initUI(self):
        self.setWindowTitle("Оформление заказа")
        self.setFixedSize(400, 300)
        grid = QGridLayout()

        # Поля ввода
        grid.addWidget(QLabel("Город:"), 0, 0)
        self.town_edit = QLineEdit()
        grid.addWidget(self.town_edit, 0, 1)

        grid.addWidget(QLabel("Улица:"), 1, 0)
        self.street_edit = QLineEdit()
        grid.addWidget(self.street_edit, 1, 1)

        grid.addWidget(QLabel("Дом:"), 2, 0)
        self.house_edit = QLineEdit()
        grid.addWidget(self.house_edit, 2, 1)

        grid.addWidget(QLabel("Квартира:"), 3, 0)
        self.flat_edit = QLineEdit()
        grid.addWidget(self.flat_edit, 3, 1)

        # Выбор способа оплаты
        grid.addWidget(QLabel("Способ оплаты:"), 4, 0)
        self.payment_combo = QComboBox()
        self.payment_combo.addItem("Оплата при получении")
        self.payment_combo.addItem("Оплата онлайн")
        grid.addWidget(self.payment_combo, 4, 1)

        # Кнопка оформления заказа
        place_order_button = QPushButton("Подтвердить")
        place_order_button.clicked.connect(self.place_order)
        grid.addWidget(place_order_button, 5, 0, 1, 2)

        self.setLayout(grid)

    def place_order(self):
        # Получение данных из полей
        town = self.town_edit.text()
        street = self.street_edit.text()
        house = self.house_edit.text()
        flat = self.flat_edit.text()
        payment_method = self.payment_combo.currentText()
        order_date = datetime.date.today().strftime("%Y-%m-%d")
        # Проверка на заполненность полей
        if not all([town, street, house, flat]):
            QMessageBox.warning(self, "Ошибка", "Заполните все поля.")
            return
        try:
            conn = sqlite3.connect(self.db_path)
            cursor = conn.cursor()
            # Оформление заказа в базе данных
            cursor.execute(
                """
                INSERT INTO orders (id_shoppingcart, order_date, town, street, house, flat, payment_method)
                VALUES (?, ?, ?, ?, ?, ?, ?)
                """,
                (self.shopping_cart_id, order_date, town, street, house, flat, payment_method),
            )
            conn.commit()
            # Очистка корзины после оформления заказа
            cursor.execute(
                "DELETE FROM shoppingcart WHERE id_shoppingcart = ?",
                (self.shopping_cart_id,)
            )
            conn.commit()
            # Уведомление об успешном оформлении заказа
            QMessageBox.information(self, "Уведомление", "Ваш заказ успешно оформлен!")

            # Вызов метода для выгрузки в Excel
            self.export_to_excel()
            self.close()
            # Очищаем корзину в окне корзины, если родитель существует
            if self.parent():  # Если у нас есть родитель (окно корзины)
                self.parent().populate_cart()

        except sqlite3.Error as e:
            QMessageBox.critical(self, "Ошибка", f"Произошла ошибка: {e}")
            print(f"Ошибка оформления заказа: {e}")
        finally:
            conn.close()

    def export_to_excel(self):
        try:
            conn = sqlite3.connect(self.db_path)
            cursor = conn.cursor()

            # Запрос для получения данных из корзины
            cursor.execute(
                "SELECT books.title, shoppingcart.amount_cart, books.price FROM shoppingcart JOIN books ON shoppingcart.id_book = books.id_book"
            )
            cart_items = cursor.fetchall()

            # Проверка, что корзина не пуста
            if not cart_items:
                QMessageBox.warning(self, "Пустая корзина", "Корзина пуста, нечего выгружать.")
                return

            # Создаем Excel-документ
            workbook = openpyxl.Workbook()
            sheet = workbook.active
            sheet.title = "Корзина"

            # Устанавливаем заголовок с текущей датой
            today_date = datetime.datetime.now().strftime("%Y-%m-%d")
            sheet.merge_cells(start_row=1, start_column=1, end_row=1, end_column=4)
            date_cell = sheet.cell(row=1, column=1, value=f"Отчет на {today_date}")
            date_cell.font = Font(bold=True, size=14)
            date_cell.alignment = Alignment(horizontal="center", vertical="center")

            # Устанавливаем заголовки таблицы
            headers = ["Название книги", "Количество", "Цена за единицу", "Общая стоимость"]
            for col_num, header in enumerate(headers, start=1):
                cell = sheet.cell(row=2, column=col_num, value=header)
                cell.font = Font(bold=True)
                cell.alignment = Alignment(horizontal="center", vertical="center")

            # Добавляем данные в таблицу
            total_price = 0
            for row_num, (title, amount, price) in enumerate(cart_items, start=3):
                # Заполнение строк таблицы
                sheet.cell(row=row_num, column=1, value=title)  # Название книги
                sheet.cell(row=row_num, column=2, value=amount)  # Количество
                price_cell = sheet.cell(row=row_num, column=3, value=price)  # Цена за единицу
                price_cell.number_format = '"₽"#,##0.00'  # Формат денежного значения
                total_cost = amount * price  # Общая стоимость
                total_cost_cell = sheet.cell(row=row_num, column=4, value=total_cost)
                total_cost_cell.number_format = '"₽"#,##0.00'

                # Выравнивание данных
                for col in range(1, 5):
                    cell = sheet.cell(row=row_num, column=col)
                    cell.alignment = Alignment(horizontal="center", vertical="center")

                # Суммирование общей стоимости
                total_price += total_cost

            # Добавление итоговой суммы
            total_row = len(cart_items) + 3
            sheet.cell(row=total_row, column=3, value="Итоговая сумма:")
            total_sum_cell = sheet.cell(row=total_row, column=4, value=total_price)
            total_sum_cell.number_format = '"₽"#,##0.00'
            total_sum_cell.alignment = Alignment(horizontal="center", vertical="center")

            # Автоширина колонок
            for col in range(1, 5):
                max_length = 0
                column_letter = openpyxl.utils.get_column_letter(col)  # Получение буквы колонки
                for row in range(2, total_row + 1):  # Пропускаем объединённые строки
                    cell = sheet.cell(row=row, column=col)
                    if cell.value:
                        max_length = max(max_length, len(str(cell.value)))
                sheet.column_dimensions[column_letter].width = max_length + 2

            # Сохранение файла
            output_path = f"Корзина_{today_date}.xlsx"
            workbook.save(output_path)
            QMessageBox.information(self, "Уведомление", f"Данные успешно выгружены в файл {output_path}")

            # Открываем файл для проверки
            os.startfile(output_path)

        except sqlite3.Error as e:
            QMessageBox.critical(self, "Ошибка", f"Ошибка выгрузки данных из базы: {e}")
        except Exception as e:
            QMessageBox.critical(self, "Ошибка", f"Произошла ошибка: {e}")
        finally:
            if 'conn' in locals():
                conn.close()


if __name__ == "__main__":
    app = QApplication(sys.argv)

    # Указываем абсолютный путь к папке с изображениями
    image_dir = r"C:\Users\Lenovo\Desktop\images"

    # Проверяем существование папки
    if not os.path.exists(image_dir):
        print(f"Папка {image_dir} не найдена. Убедитесь, что путь указан правильно.")
        sys.exit(1)

    # Создаем экземпляр окна каталога с указанным путем
    catalog_window = CatalogWindow(image_dir=image_dir)

    catalog_window.hide()  # Скрываем его до успешной авторизации

    auth_window = AuthWindow(catalog_window)  # Передаем ссылку на каталог в AuthWindow
    auth_window.show()

    sys.exit(app.exec())
