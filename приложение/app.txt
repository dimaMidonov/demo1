import tkinter as tk
from tkinter import ttk, messagebox
from database import Database
from logic import ProductManager
import os


class AppIconMixin:
    """Миксин для установки иконки на все окна"""

    def set_app_icon(self, window):
        """Устанавливает иконку на указанное окно"""
        try:
            icon_path = os.path.join(os.path.dirname(__file__), 'Наш декор.ico')
            if os.path.exists(icon_path):
                window.iconbitmap(icon_path)
            else:
                window.iconbitmap(default='')
        except Exception as e:
            print(f"Не удалось установить иконку: {e}")


class WallpaperApp(AppIconMixin):
    def __init__(self, root):
        self.root = root
        self.root.title("Наш декор")
        self.root.geometry("950x600")

        # Установка иконки на главное окно
        self.set_app_icon(self.root)

        self.db = Database()
        self.manager = ProductManager(self.db)

        self._setup_ui()
        self._load_products()

    def _setup_ui(self):
        """Настраивает интерфейс"""
        toolbar = ttk.Frame(self.root)
        toolbar.pack(fill=tk.X, padx=5, pady=5)

        ttk.Button(toolbar, text="Добавить", command=self._add_product).pack(side=tk.LEFT)
        ttk.Button(toolbar, text="Обновить", command=self._load_products).pack(side=tk.LEFT, padx=5)

        self.main_frame = ttk.Frame(self.root)
        self.main_frame.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)

        self.canvas = tk.Canvas(self.main_frame)
        self.scrollbar = ttk.Scrollbar(self.main_frame, orient="vertical", command=self.canvas.yview)
        self.scrollable_frame = ttk.Frame(self.canvas)

        self.scrollable_frame.bind(
            "<Configure>",
            lambda e: self.canvas.configure(scrollregion=self.canvas.bbox("all"))
        )

        self.canvas.create_window((0, 0), window=self.scrollable_frame, anchor="nw")
        self.canvas.configure(yscrollcommand=self.scrollbar.set)

        self.canvas.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
        self.scrollbar.pack(side=tk.RIGHT, fill=tk.Y)

    def _load_products(self):
        for widget in self.scrollable_frame.winfo_children():
            widget.destroy()

        products = self.manager.get_all_products()
        if products:
            print("Available keys in product:", products[0].keys())

        for product in products:
            product_type = product['type_name']
            if product_type is None:
                print(f"Warning: No 'type_name' found in product: {product}")
                continue

            cost = self.manager.calculate_product_cost(product['id'])

            product_frame = ttk.LabelFrame(self.scrollable_frame, text="", padding=5)
            product_frame.pack(fill=tk.X, padx=10, pady=5)
            product_frame.bind("<Double-1>", lambda e, p=product: self._edit_product(p))

            ttk.Label(product_frame, text=f"{product['type_name']} | {product['name']}",
                      font=("Gabriola", 18, "bold")).grid(
                row=0, column=0, columnspan=2, sticky="ew", padx=5, pady=2
            )

            product_frame.grid_columnconfigure(0, weight=1)
            product_frame.grid_columnconfigure(1, weight=1)

            ttk.Label(product_frame, text="Артикул:", font=("Gabriola", 14)).grid(row=1, column=0, sticky=tk.W, padx=5, pady=2)
            ttk.Label(product_frame, text=product['article'], font=("Gabriola", 14)).grid(row=1, column=1, sticky=tk.E, padx=5, pady=2)

            ttk.Label(product_frame, text="Минимальная стоимость для партнера (р):", font=("Gabriola", 14)).grid(row=2, column=0, sticky=tk.W, padx=5, pady=2)
            ttk.Label(product_frame, text=f"{product['min_price']:.2f}", font=("Gabriola", 14)).grid(row=2, column=1, sticky=tk.E, padx=5, pady=2)

            ttk.Label(product_frame, text="Стоимость (р):", font=('Gabriola', 18, 'bold')).grid(row=3, column=0, sticky=tk.W, padx=5, pady=2)
            ttk.Label(product_frame, text=f"{cost:.2f}", font=('Gabriola', 18, 'bold')).grid(row=3, column=1, sticky=tk.E, padx=5, pady=2)

            ttk.Label(product_frame, text="Ширина (м):", font=("Gabriola", 14)).grid(row=4, column=0, sticky=tk.W, padx=5, pady=2)
            ttk.Label(product_frame, text=f"{product['width']:.2f}", font=("Gabriola", 14)).grid(row=4, column=1, sticky=tk.E, padx=5, pady=2)

    def _add_product(self):
        ProductForm(self.root, self.db, self.manager, self._load_products)

    def _edit_product(self, product):
        ProductForm(self.root, self.db, self.manager, self._load_products, product)


class ProductForm(tk.Toplevel, AppIconMixin):
    def __init__(self, parent, db, manager, callback, product=None):
        super().__init__(parent)
        self.db = db
        self.manager = manager
        self.callback = callback
        self.product = product

        self.title("Добавить продукт" if not product else "Редактировать продукт")
        self.geometry("400x300")

        # Установка иконки на окно формы
        self.set_app_icon(self)

        self._setup_ui()
        if product:
            self._load_data()

        self.title("Добавить продукт" if not product else "Редактировать продукт")
        self.geometry("400x300")

        self._setup_ui()
        if product:
            self._load_data()

    def _setup_ui(self):
        ttk.Label(self, text="Артикул:", font=("Gabriola", 14)).grid(row=0, column=0, sticky=tk.W, padx=5, pady=5)
        self.article_entry = ttk.Entry(self)
        self.article_entry.grid(row=0, column=1, sticky=tk.EW, padx=5, pady=5)

        ttk.Label(self, text="Наименование:", font=("Gabriola", 14)).grid(row=1, column=0, sticky=tk.W, padx=5, pady=5)
        self.name_entry = ttk.Entry(self)
        self.name_entry.grid(row=1, column=1, sticky=tk.EW, padx=5, pady=5)

        ttk.Label(self, text="Тип:", font=("Gabriola", 14)).grid(row=2, column=0, sticky=tk.W, padx=5, pady=5)
        self.type_combobox = ttk.Combobox(self)
        self.type_combobox.grid(row=2, column=1, sticky=tk.EW, padx=5, pady=5)
        types = self.db.fetch_all("SELECT name FROM product_types")
        self.type_combobox['values'] = [t['name'] for t in types]

        ttk.Label(self, text="Мин. цена:", font=("Gabriola", 14)).grid(row=3, column=0, sticky=tk.W, padx=5, pady=5)
        self.price_entry = ttk.Entry(self)
        self.price_entry.grid(row=3, column=1, sticky=tk.EW, padx=5, pady=5)

        ttk.Label(self, text="Ширина:", font=("Gabriola", 14)).grid(row=5, column=0, sticky=tk.W, padx=5, pady=5)
        self.width_entry = ttk.Entry(self)
        self.width_entry.grid(row=5, column=1, sticky=tk.EW, padx=5, pady=5)

        btn_frame = ttk.Frame(self)
        btn_frame.grid(row=6, column=0, columnspan=2, pady=10)

        ttk.Button(btn_frame, text="Сохранить", command=self._save).pack(side=tk.RIGHT, padx=5)
        ttk.Button(btn_frame, text="Назад", command=self.destroy).pack(side=tk.RIGHT)



    def _load_data(self):
        self.article_entry.insert(0, self.product['article'])
        self.name_entry.insert(0, self.product['name'])
        self.price_entry.insert(0, str(self.product['min_price']))
        self.width_entry.insert(0, str(self.product['width']))

        type_name = self.db.fetch_one(
            "SELECT name FROM product_types WHERE id = ?",
            (self.product['type_id'],)
        )['name']
        self.type_combobox.set(type_name)

    def _save(self):
        try:
            article = self.article_entry.get()
            name = self.name_entry.get()
            type_name = self.type_combobox.get()
            price = float(self.price_entry.get())
            width = float(self.width_entry.get())

            type_id = self.db.fetch_one(
                "SELECT id FROM product_types WHERE name = ?",
                (type_name,)
            )['id']

            if self.product:
                self.manager.update_product(
                    product_id=self.product['id'],
                    name=name,
                    type_id=type_id,
                    min_price=price,
                    width=width
                )
                messagebox.showinfo("Успех", "Продукт обновлен")
            else:
                self.manager.add_product(
                    article=article,
                    name=name,
                    type_id=type_id,
                    min_price=price,
                    width=width
                )
                messagebox.showinfo("Успех", "Продукт добавлен")

            self.callback()
            self.destroy()
        except ValueError as e:
            messagebox.showerror("Ошибка", str(e))
        except Exception as e:
            messagebox.showerror("Ошибка", f"Не удалось сохранить: {str(e)}")

if __name__ == "__main__":
    root = tk.Tk()
    app = WallpaperApp(root)
    root.mainloop()
