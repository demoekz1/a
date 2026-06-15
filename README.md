```
import tkinter as tk
from tkinter import simpledialog, messagebox, ttk, filedialog
import pymysql
import os
import shutil
from PIL import Image, ImageTk

class DB:
    def __init__(self):
        self.conn = pymysql.connect(
            host="localhost", user="root", password="root",
            database="store_db", cursorclass=pymysql.cursors.DictCursor
        )
        self.cur = self.conn.cursor()

    def login(self, u, p):
        self.cur.execute("SELECT role, full_name FROM users WHERE username=%s AND password=%s", (u, p))
        r = self.cur.fetchone()
        return r if r else None

    def get_categories(self):
        self.cur.execute("SELECT id, name FROM categories")
        return self.cur.fetchall()

    def get_manufacturers(self):
        self.cur.execute("SELECT id, name FROM manufacturers")
        return self.cur.fetchall()

    def get_suppliers(self):
        self.cur.execute("SELECT id, name FROM suppliers")
        return self.cur.fetchall()

    def get_products(self, search='', sort='', supplier_id=None, category_id=None):
        sql = """SELECT p.id, p.name, p.price, p.stock, p.discount, p.photo_path,
                        c.name AS cat_name, s.name AS sup_name
                 FROM products p
                 LEFT JOIN categories c ON p.category_id = c.id
                 LEFT JOIN suppliers s ON p.supplier_id = s.id
                 WHERE 1=1"""
        params = []
        if search:
            sql += " AND (p.name LIKE %s OR c.name LIKE %s OR s.name LIKE %s)"
            params.extend(['%'+search+'%']*3)
        if supplier_id:
            sql += " AND p.supplier_id = %s"
            params.append(supplier_id)
        if category_id:
            sql += " AND p.category_id = %s"
            params.append(category_id)
        if sort == 'asc':
            sql += " ORDER BY p.stock ASC"
        elif sort == 'desc':
            sql += " ORDER BY p.stock DESC"
        self.cur.execute(sql, params)
        return self.cur.fetchall()

    def get_product(self, pid):
        self.cur.execute("SELECT * FROM products WHERE id=%s", (pid,))
        return self.cur.fetchone()

    def add_product(self, data):
        self.cur.execute("""INSERT INTO products (name, price, stock, category_id, manufacturer_id,
                            supplier_id, unit, discount, description, photo_path)
                            VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)""", data)
        self.conn.commit()

    def update_product(self, pid, data):
        self.cur.execute("""UPDATE products SET name=%s, price=%s, stock=%s,
                            category_id=%s, manufacturer_id=%s, supplier_id=%s,
                            unit=%s, discount=%s, description=%s, photo_path=%s
                            WHERE id=%s""", (*data, pid))
        self.conn.commit()

    def delete_product(self, pid):
        self.cur.execute("SELECT COUNT(*) AS cnt FROM orders WHERE product_id=%s", (pid,))
        if self.cur.fetchone()['cnt'] > 0:
            raise Exception("Товар присутствует в заказах")
        self.cur.execute("DELETE FROM products WHERE id=%s", (pid,))
        self.conn.commit()

    def add_order(self, username, product_id, quantity):
        self.cur.execute("INSERT INTO orders (username, product_id, quantity) VALUES (%s,%s,%s)",
                         (username, product_id, quantity))
        self.cur.execute("UPDATE products SET stock = stock - %s WHERE id=%s", (quantity, product_id))
        self.conn.commit()

    def get_orders(self, username=None):
        if username:
            self.cur.execute("""SELECT o.id, p.name, o.quantity, o.order_date, o.status
                                FROM orders o JOIN products p ON o.product_id = p.id
                                WHERE o.username=%s ORDER BY o.order_date DESC""", (username,))
        else:
            self.cur.execute("""SELECT o.id, p.name, o.quantity, o.order_date, o.status
                                FROM orders o JOIN products p ON o.product_id = p.id
                                ORDER BY o.order_date DESC""")
        return self.cur.fetchall()

    def update_order_status(self, order_id, status):
        self.cur.execute("UPDATE orders SET status=%s WHERE id=%s", (status, order_id))
        self.conn.commit()


class App:
    def __init__(self, root):
        self.root = root
        self.root.title("Управление продажами обуви")
        self.root.geometry("850x650")
        self.root.configure(bg='#f5f6f8')
        self.db = DB()
        self.editing_window_open = False
        self.login_screen()

    def clear(self):
        for w in self.root.winfo_children():
            w.destroy()
        for attr in  ('search_var', 'combo_cat', 'combo_sup', 'sort_var'):
            if hasattr(self, attr):
                delattr(self,attr)

    def login_screen(self):
        self.clear()
        frame = tk.Frame(self.root, bg='#f5f6f8')
        frame.place(relx=0.5, rely=0.4, anchor='center')
        tk.Label(frame, text="Вход в систему", font=('Arial', 16, 'bold'), bg='#f5f6f8').pack(pady=10)
        tk.Label(frame, text="Логин:", bg='#f5f6f8').pack()
        self.entry_u = tk.Entry(frame, width=25)
        self.entry_u.pack(pady=2)
        tk.Label(frame, text="Пароль:", bg='#f5f6f8').pack()
        self.entry_p = tk.Entry(frame, show="*", width=25)
        self.entry_p.pack(pady=2)
        tk.Button(frame, text="Войти", width=15, command=self.do_login).pack(pady=5)
        tk.Button(frame, text="Продолжить как гость", width=15, command=self.guest_screen).pack()

    def do_login(self):
        user = self.db.login(self.entry_u.get(), self.entry_p.get())
        if not user:
            messagebox.showerror("Ошибка", "Неверный логин или пароль")
            return
        self.username = self.entry_u.get()
        self.role = user['role']
        self.full_name = user['full_name']
        if self.role == 'admin':
            self.admin_screen()
        elif self.role == 'manager':
            self.manager_screen()
        else:
            self.user_screen()

    def guest_screen(self):
        self.role = 'guest'
        self.username = ''
        self.full_name = 'Гость'
        self.show_product_list()

    def show_product_list(self):
        self.clear()
        top = tk.Frame(self.root, bg='#e9ecf0')
        top.pack(fill='x', padx=5, pady=5)

        tk.Label(top, text=f"Пользователь: {self.full_name} ({self.role})", bg='#e9ecf0').pack(side='left', padx=5)

        if self.role in ('manager', 'admin'):
            self.search_var = tk.StringVar()
            self.search_var.trace('w', lambda *a: self.refresh_products())
            tk.Entry(top, textvariable=self.search_var, width=15).pack(side='left', padx=5)

        tk.Label(top, text="Категория:", bg='#e9ecf0').pack(side='left')
        self.combo_cat = ttk.Combobox(top, state='readonly', width=15)
        self.combo_cat.pack(side='left', padx=5)
        categories = self.db.get_categories()
        self.cat_dict = {c['name']: c['id'] for c in categories}
        self.combo_cat['values'] = ['Все'] + list(self.cat_dict.keys())
        self.combo_cat.current(0)
        self.combo_cat.bind('<<ComboboxSelected>>', lambda e: self.refresh_products())

        if self.role in ('manager', 'admin'):
            tk.Label(top, text="Поставщик:", bg='#e9ecf0').pack(side='left', padx=5)
            self.combo_sup = ttk.Combobox(top, state='readonly', width=15)
            self.combo_sup.pack(side='left', padx=5)
            suppliers = self.db.get_suppliers()
            self.sup_dict = {s['name']: s['id'] for s in suppliers}
            self.combo_sup['values'] = ['Все'] + list(self.sup_dict.keys())
            self.combo_sup.current(0)
            self.combo_sup.bind('<<ComboboxSelected>>', lambda e: self.refresh_products())

            self.sort_var = tk.StringVar(value='none')
            tk.Radiobutton(top, text="Без", variable=self.sort_var, value='none',
                           command=self.refresh_products, bg='#e9ecf0').pack(side='left')
            tk.Radiobutton(top, text="↑ кол-во", variable=self.sort_var, value='asc',
                           command=self.refresh_products, bg='#e9ecf0').pack(side='left')
            tk.Radiobutton(top, text="↓ кол-во", variable=self.sort_var, value='desc',
                           command=self.refresh_products, bg='#e9ecf0').pack(side='left')

        tk.Button(top, text="Выход", command=self.login_screen, bg='#d0d4db').pack(side='right', padx=5)

        self.canvas = tk.Canvas(self.root, bg='#f5f6f8', highlightthickness=0)
        scrollbar = tk.Scrollbar(self.root, orient='vertical', command=self.canvas.yview)
        self.canvas.configure(yscrollcommand=scrollbar.set)
        scrollbar.pack(side='right', fill='y')
        self.canvas.pack(side='left', fill='both', expand=True, padx=5, pady=(0,5))

        self.cards_frame = tk.Frame(self.canvas, bg='#f5f6f8')
        self.canvas.create_window((0,0), window=self.cards_frame, anchor='nw')
        self.cards_frame.bind("<Configure>", lambda e: self.canvas.configure(scrollregion=self.canvas.bbox("all")))

        bottom = tk.Frame(self.root, bg='#e9ecf0')
        bottom.pack(fill='x', pady=5)
        if self.role == 'admin':
            tk.Button(bottom, text="Добавить товар", command=self.add_product_form).pack(side='left', padx=5)
            tk.Button(bottom, text="Заказы", command=lambda: self.show_orders_window(None)).pack(side='left', padx=5)
        elif self.role == 'manager':
            tk.Button(bottom, text="Заказы", command=lambda: self.show_orders_window(None)).pack(side='left', padx=5)
        elif self.role == 'user':
            tk.Button(bottom, text="Мои заказы", command=lambda: self.show_orders_window(self.username)).pack(side='left', padx=5)

        self.refresh_products()

    def refresh_products(self):
        for w in self.cards_frame.winfo_children():
            w.destroy()

        search = self.search_var.get() if hasattr(self, 'search_var') else ''
        sort = self.sort_var.get() if hasattr(self, 'sort_var') else 'none'
        cat_id = None
        if self.combo_cat.get() != 'Все':
            cat_id = self.cat_dict.get(self.combo_cat.get())
        sup_id = None
        if hasattr(self, 'combo_sup') and self.combo_sup.get() != 'Все':
            sup_id = self.sup_dict.get(self.combo_sup.get())

        products = self.db.get_products(search, sort, sup_id, cat_id)

        for p in products:
            if p['stock'] == 0:
                bg = '#b0e0ff'
            elif p['discount'] > 15:
                bg = '#2E8B57'
            else:
                bg = 'white'

            card = tk.Frame(self.cards_frame, bd=1, relief='ridge', bg=bg)
            card.pack(fill='x', padx=5, pady=3)

            photo_path = p.get('photo_path')
            if photo_path and os.path.isfile(photo_path):
                try:
                    img = Image.open(photo_path).resize((60,60), Image.LANCZOS)
                    photo = ImageTk.PhotoImage(img)
                    lbl_img = tk.Label(card, image=photo, bg=bg)
                    lbl_img.image = photo
                    lbl_img.pack(side='left', padx=5)
                except:
                    self._no_photo_label(card, bg)
            else:
                self._no_photo_label(card, bg)

            info = tk.Frame(card, bg=bg)
            info.pack(side='left', fill='both', expand=True)
            tk.Label(info, text=p['name'], font=('Arial', 11, 'bold'), bg=bg).pack(anchor='w')
            price_text = f"Цена: {p['price']:.2f} руб."
            if p['discount'] > 0:
                final_price = p['price'] * (100 - p['discount']) / 100
                price_text += f"  →  {final_price:.2f} руб. (скидка {p['discount']}%)"
            tk.Label(info, text=price_text, bg=bg).pack(anchor='w')
            tk.Label(info, text=f"Остаток: {p['stock']} | Поставщик: {p['sup_name'] or '-'}", bg=bg).pack(anchor='w')
            status = "В наличии" if p['stock'] > 0 else "Нет на складе"
            tk.Label(info, text=status, fg="green" if p['stock']>0 else "red", bg=bg).pack(anchor='w')

            btn_frame = tk.Frame(card, bg=bg)
            btn_frame.pack(side='right', padx=5)
            if self.role == 'admin':
                tk.Button(btn_frame, text="Изменить", command=lambda pid=p['id']: self.edit_product_form(pid)).pack(side='left', padx=2)
                tk.Button(btn_frame, text="Удалить", command=lambda pid=p['id']: self.delete_product(pid)).pack(side='left', padx=2)
            elif self.role == 'user':
                tk.Button(btn_frame, text="Купить", command=lambda prod=p: self.buy_product(prod)).pack(side='left')

    def _no_photo_label(self, parent, bg):
        if os.path.isfile('picture.png'):
            img = Image.open('picture.png').resize((60,60), Image.LANCZOS)
            photo = ImageTk.PhotoImage(img)
            lbl = tk.Label(parent, image=photo, bg=bg)
            lbl.image = photo
        else:
            lbl = tk.Label(parent, text="Нет\nфото", width=8, height=3, bg=bg)
        lbl.pack(side='left', padx=5)

    def add_product_form(self):
        if self.editing_window_open:
            messagebox.showwarning("Предупреждение", "Окно редактирования уже открыто")
            return
        self.editing_window_open = True
        self.product_editor("Добавить товар")

    def edit_product_form(self, pid):
        if self.editing_window_open:
            messagebox.showwarning("Предупреждение", "Окно редактирования уже открыто")
            return
        prod = self.db.get_product(pid)
        if not prod:
            messagebox.showerror("Ошибка", "Товар не найден")
            return
        self.editing_window_open = True
        self.product_editor("Изменить товар", prod)

    def product_editor(self, title, prod=None):
        win = tk.Toplevel(self.root)
        win.title(title)
        win.geometry("420x580")
        win.configure(bg='#f0f0f0')
        win.protocol("WM_DELETE_WINDOW", lambda: self._close_editor(win))

        main_frame = tk.Frame(win, bg='#f0f0f0')
        main_frame.pack(fill='both', expand=True, padx=10, pady=10)

        fields = ['Название', 'Цена', 'Количество', 'Скидка (%)', 'Ед. изм.']
        entries = {}
        for f in fields:
            tk.Label(main_frame, text=f, bg='#f0f0f0').pack(anchor='w')
            ent = tk.Entry(main_frame, width=30)
            ent.pack(fill='x', pady=2)
            entries[f] = ent

        tk.Label(main_frame, text="Категория", bg='#f0f0f0').pack(anchor='w')
        combo_cat = ttk.Combobox(main_frame, state='readonly', width=27)
        combo_cat.pack(fill='x', pady=2)
        cats = self.db.get_categories()
        cat_dict = {c['name']: c['id'] for c in cats}
        combo_cat['values'] = list(cat_dict.keys())
        entries['Категория'] = combo_cat

        tk.Label(main_frame, text="Производитель", bg='#f0f0f0').pack(anchor='w')
        combo_man = ttk.Combobox(main_frame, state='readonly', width=27)
        combo_man.pack(fill='x', pady=2)
        mans = self.db.get_manufacturers()
        man_dict = {m['name']: m['id'] for m in mans}
        combo_man['values'] = list(man_dict.keys())
        entries['Производитель'] = combo_man

        tk.Label(main_frame, text="Поставщик", bg='#f0f0f0').pack(anchor='w')
        combo_sup = ttk.Combobox(main_frame, state='readonly', width=27)
        combo_sup.pack(fill='x', pady=2)
        sups = self.db.get_suppliers()
        sup_dict = {s['name']: s['id'] for s in sups}
        combo_sup['values'] = list(sup_dict.keys())
        entries['Поставщик'] = combo_sup

        tk.Label(main_frame, text="Описание", bg='#f0f0f0').pack(anchor='w')
        desc_ent = tk.Entry(main_frame, width=30)
        desc_ent.pack(fill='x', pady=2)
        entries['Описание'] = desc_ent

        tk.Label(main_frame, text="Фото", bg='#f0f0f0').pack(anchor='w')
        photo_var = tk.StringVar()
        photo_frame = tk.Frame(main_frame, bg='#f0f0f0')
        photo_frame.pack(fill='x', pady=2)
        tk.Button(photo_frame, text="Выбрать изображение",
                  command=lambda: photo_var.set(filedialog.askopenfilename(filetypes=[("Изображения", "*.png *.jpg *.jpeg")])),
                  width=20).pack(side='left', padx=5)
        tk.Label(photo_frame, textvariable=photo_var, bg='#f0f0f0', fg='blue', width=20).pack(side='left')

        if prod:
            entries['Название'].insert(0, prod['name'])
            entries['Цена'].insert(0, str(prod['price']))
            entries['Количество'].insert(0, str(prod['stock']))
            entries['Скидка (%)'].insert(0, str(prod['discount']))
            entries['Ед. изм.'].insert(0, prod.get('unit', 'шт.'))
            desc_ent.insert(0, prod.get('description', ''))
            cat_name = next((c['name'] for c in cats if c['id'] == prod['category_id']), '')
            if cat_name: combo_cat.set(cat_name)
            man_name = next((m['name'] for m in mans if m['id'] == prod['manufacturer_id']), '')
            if man_name: combo_man.set(man_name)
            sup_name = next((s['name'] for s in sups if s['id'] == prod['supplier_id']), '')
            if sup_name: combo_sup.set(sup_name)
            if prod.get('photo_path'):
                photo_var.set(prod['photo_path'])

        def save():
            try:
                name = entries['Название'].get().strip()
                price = float(entries['Цена'].get())
                stock = int(entries['Количество'].get())
                discount = float(entries['Скидка (%)'].get())
                unit = entries['Ед. изм.'].get().strip() or 'шт.'
                description = entries['Описание'].get().strip()
                if price < 0 or stock < 0 or discount < 0 or discount > 100:
                    raise ValueError("Некорректные значения")
                cat_id = cat_dict.get(combo_cat.get())
                man_id = man_dict.get(combo_man.get())
                sup_id = sup_dict.get(combo_sup.get())
                if not cat_id or not man_id or not sup_id:
                    raise ValueError("Выберите категорию, производителя и поставщика")

                new_photo = photo_var.get()
                if new_photo and os.path.isfile(new_photo):
                    img = Image.open(new_photo)
                    img.thumbnail((300, 200), Image.LANCZOS)
                    os.makedirs("images", exist_ok=True)
                    dst = os.path.join("images", os.path.basename(new_photo))
                    img.save(dst)
                    if prod and prod['photo_path'] and os.path.isfile(prod['photo_path']):
                        if os.path.abspath(prod['photo_path']) != os.path.abspath(dst):
                            os.remove(prod['photo_path'])
                    new_photo = dst
                elif prod and prod['photo_path']:
                    new_photo = prod['photo_path']
                else:
                    new_photo = None

                data = (name, price, stock, cat_id, man_id, sup_id, unit, discount, description, new_photo)
                if prod:
                    self.db.update_product(prod['id'], data)
                else:
                    self.db.add_product(data)
                self._close_editor(win)
                self.refresh_products()
                messagebox.showinfo("Успех", "Товар сохранён")
            except Exception as e:
                messagebox.showerror("Ошибка", str(e))

        tk.Button(main_frame, text="Сохранить", command=save, bg='#c0d0e0', width=20).pack(pady=15)

    def _close_editor(self, win):
        self.editing_window_open = False
        win.destroy()

    def delete_product(self, pid):
        if not messagebox.askyesno("Подтверждение", "Удалить товар?"):
            return
        try:
            self.db.delete_product(pid)
            self.refresh_products()
        except Exception as e:
            messagebox.showerror("Ошибка", str(e))

    def buy_product(self, prod):
        if prod['stock'] <= 0:
            messagebox.showwarning("Нет в наличии", "Товар отсутствует")
            return
        qty = simpledialog.askinteger("Покупка", f"{prod['name']}\nВ наличии: {prod['stock']}",
                                      minvalue=1, maxvalue=prod['stock'])
        if qty:
            self.db.add_order(self.username, prod['id'], qty)
            self.refresh_products()
            messagebox.showinfo("Успех", "Заказ оформлен")

    def show_orders_window(self, username):
        win = tk.Toplevel(self.root)
        win.title("Заказы")
        win.geometry("700x450")
        win.configure(bg='#f5f6f8')

        columns = ("ID", "Товар", "Кол-во", "Дата", "Статус")
        tree = ttk.Treeview(win, columns=columns, show="headings", height=15)
        for col in columns:
            tree.heading(col, text=col)
            tree.column(col, width=100)
        tree.pack(fill='both', expand=True, padx=5, pady=5)

        orders = self.db.get_orders(username)
        for o in orders:
            tree.insert("", "end", iid=str(o['id']),
                        values=(o['id'], o['name'], o['quantity'], o['order_date'], o['status']))

        if self.role in ('manager', 'admin') and username is None:
            btn_frame = tk.Frame(win, bg='#e9ecf0')
            btn_frame.pack(fill='x', padx=5, pady=5)
            tk.Label(btn_frame, text="Новый статус:", bg='#e9ecf0').pack(side='left')
            status_combo = ttk.Combobox(btn_frame, state='readonly', width=15,
                                        values=['pending','processing','completed','cancelled'])
            status_combo.pack(side='left', padx=5)

            def change_status():
                sel = tree.focus()
                if not sel:
                    messagebox.showwarning("Не выбрано", "Выберите заказ")
                    return
                new_status = status_combo.get()
                if not new_status:
                    return
                self.db.update_order_status(int(sel), new_status)
                for item in tree.get_children():
                    tree.delete(item)
                for o in self.db.get_orders(username):
                    tree.insert("", "end", iid=str(o['id']),
                                values=(o['id'], o['name'], o['quantity'], o['order_date'], o['status']))

            tk.Button(btn_frame, text="Применить", command=change_status, bg='#c0d0e0').pack(side='left', padx=5)

    def admin_screen(self):
        self.show_product_list()

    def user_screen(self):
        self.show_product_list()

    def manager_screen(self):
        self.show_product_list()


if __name__ == '__main__':
    if not os.path.isfile('picture.png'):
        img = Image.new('RGB', (60,60), color='gray')
        img.save('picture.png')
    root = tk.Tk()
    App(root)
    root.mainloop()


CREATE DATABASE IF NOT EXISTS store_db;
USE store_db;

CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    role ENUM('admin','manager','user','guest') NOT NULL,
    full_name VARCHAR(100) DEFAULT ''
);

CREATE TABLE categories (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL	
);

CREATE TABLE manufacturers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE suppliers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    stock INT NOT NULL DEFAULT 0,
    category_id INT,
    manufacturer_id INT,
    supplier_id INT,
    unit VARCHAR(20) DEFAULT 'шт.',
    discount DECIMAL(5,2) DEFAULT 0,
    description TEXT,
    photo_path VARCHAR(255) DEFAULT NULL,
    FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE SET NULL,
    FOREIGN KEY (manufacturer_id) REFERENCES manufacturers(id) ON DELETE SET NULL,
    FOREIGN KEY (supplier_id) REFERENCES suppliers(id) ON DELETE SET NULL
);

CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status ENUM('pending','processing','completed','cancelled') DEFAULT 'pending',
    FOREIGN KEY (username) REFERENCES users(username) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE RESTRICT
);

INSERT INTO users (username, password, role, full_name) VALUES
('admin','admin','admin','Администратор'),
('manager','manager','manager','Менеджер'),
('user','user','user','Покупатель');

INSERT INTO categories (name) VALUES ('Обувь'), ('Спорттовары'), ('Аксессуары');
INSERT INTO manufacturers (name) VALUES ('Nike'), ('Adidas'), ('Puma');
INSERT INTO suppliers (name) VALUES ('ООО "ОбувьПром"'), ('ИП Иванов'), ('Adidas');

INSERT INTO products (name, price, stock, category_id, manufacturer_id, supplier_id, unit, discount, description, photo_path) VALUES
('Кроссовки Air Max', 8999.00, 10, 1, 1, 1, 'пара', 20, 'Лёгкие кроссовки для бега', NULL),
('Ботинки зимние', 5999.00, 0, 1, 2, 2, 'пара', 10, 'Тёплые ботинки', NULL),
('Футболка спортивная', 1499.00, 25, 2, 3, 3, 'шт.', 5, 'Дышащая ткань', NULL),
('Сумка для обуви', 1299.00, 5, 3, 1, 1, 'шт.', 0, 'Удобная сумка', NULL);


```
