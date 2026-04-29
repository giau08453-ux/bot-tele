import telebot
import json
from telebot.types import ReplyKeyboardMarkup

TOKEN = "8730864516:AAHq7--PRivF7QfEwrkEtYZnqpRtQ7Zj-Vo"
ADMIN_ID = 7105492479

bot = telebot.TeleBot(TOKEN)

# ===== LOAD / SAVE =====
def load():
    try:
        with open("data.json", "r") as f:
            return json.load(f)
    except:
        return {}

def save():
    with open("data.json", "w") as f:
        json.dump(users, f)

users = load()

# ===== SẢN PHẨM =====
products = {
    "Br PC (1 ngày)": {"price": 45000, "stock": 0},
    "Br PC (7 ngày)": {"price": 90000, "stock": 0},
    "Br PC (30 ngày)": {"price": 180000, "stock": 0},

    "Drip (1 ngày)": {"price": 40000, "stock": 18},
    "Drip (7 ngày)": {"price": 90000, "stock": 19},
    "Drip (15 ngày)": {"price": 120000, "stock": 12},
    "Drip (30 ngày)": {"price": 160000, "stock": 33},

    "Flu iOS (1 ngày)": {"price": 90000, "stock": 0},
    "Flu iOS (7 ngày)": {"price": 220000, "stock": 17},
    "Flu iOS (30 ngày)": {"price": 3500000, "stock": 0},

    "Pato Xanh (1 ngày)": {"price": 30000, "stock": 0},
    "Pato Xanh (7 ngày)": {"price": 70000, "stock": 46},
    "Pato Xanh (15 ngày)": {"price": 100000, "stock": 4},
    "Pato Xanh (30 ngày)": {"price": 150000, "stock": 10},

    "Pato Đỏ (1 ngày)": {"price": 40000, "stock": 23},
    "Pato Đỏ (7 ngày)": {"price": 90000, "stock": 70},
    "Pato Đỏ (15 ngày)": {"price": 120000, "stock": 20},
    "Pato Đỏ (30 ngày)": {"price": 160000, "stock": 46},
}

# ===== MENU =====
def menu():
    kb = ReplyKeyboardMarkup(resize_keyboard=True)
    kb.add("🛒 Mua Sản Phẩm", "👤 Tài Khoản")
    return kb

# ===== START =====
@bot.message_handler(commands=['start'])
def start(m):
    uid = str(m.from_user.id)
    if uid not in users:
        users[uid] = {"money": 0}
        save()
    bot.send_message(m.chat.id, "👋 Chào mừng đến shop!", reply_markup=menu())

# ===== TÀI KHOẢN =====
@bot.message_handler(func=lambda m: m.text == "👤 Tài Khoản")
def account(m):
    uid = str(m.from_user.id)
    money = users.get(uid, {}).get("money", 0)
    bot.send_message(m.chat.id, f"💰 Số dư: {money}đ")

# ===== SHOP =====
@bot.message_handler(func=lambda m: m.text == "🛒 Mua Sản Phẩm")
def shop(m):
    text = "📦 DANH SÁCH SẢN PHẨM:\n\n"
    for name, data in products.items():
        text += f"{name} | {data['price']}đ (Còn: {data['stock']})\n"
    text += "\n👉 Gõ đúng tên sản phẩm để mua"
    bot.send_message(m.chat.id, text)

# ===== CHỌN MUA → TẠO QR =====
@bot.message_handler(func=lambda m: m.text in products)
def buy(m):
    uid = str(m.from_user.id)
    product = products[m.text]

    if product["stock"] <= 0:
        bot.send_message(m.chat.id, "❌ Hết hàng!")
        return

    price = product["price"]

    qr_link = f"https://img.vietqr.io/image/970422-0355998600-compact2.png?amount={price}&addInfo=NAP_{uid}&accountName=DO+VAN+GIAO"

    bot.send_photo(
        m.chat.id,
        photo=qr_link,
        caption=(
            f"💸 THANH TOÁN\n\n"
            f"📦 {m.text}\n"
            f"💰 {price}đ\n\n"
            f"📌 Nội dung: NAP_{uid}\n\n"
            "⚠️ Chuyển xong nhắn: nap xong"
        )
    )

    users[uid]["pending"] = m.text
    save()

# ===== USER XÁC NHẬN =====
@bot.message_handler(func=lambda m: m.text.lower() == "nap xong")
def confirm(m):
    uid = str(m.from_user.id)

    if "pending" not in users[uid]:
        bot.send_message(m.chat.id, "❌ Bạn chưa chọn sản phẩm")
        return

    product_name = users[uid]["pending"]
    product = products[product_name]

    bot.send_message(
        ADMIN_ID,
        f"💰 DUYỆT ĐƠN\n"
        f"User: {uid}\n"
        f"Sản phẩm: {product_name}\n"
        f"Tiền: {product['price']}đ\n\n"
        f"Duyệt: /ok {uid}"
    )

    bot.send_message(m.chat.id, "⏳ Đợi admin duyệt")

# ===== ADMIN DUYỆT =====
@bot.message_handler(commands=['ok'])
def ok(m):
    if m.from_user.id != ADMIN_ID:
        return

    try:
        _, uid = m.text.split()
        uid = str(uid)

        product_name = users[uid]["pending"]
        product = products[product_name]

        if product["stock"] <= 0:
            bot.send_message(m.chat.id, "❌ Hết hàng")
            return

        product["stock"] -= 1

        key = "KEY-" + uid[-4:]

        bot.send_message(int(uid), f"✅ Thanh toán thành công!\n🔑 {key}")

        del users[uid]["pending"]
        save()

        bot.send_message(m.chat.id, "✔️ Đã duyệt")

    except:
        bot.send_message(m.chat.id, "Sai cú pháp /ok id")

# ===== ADMIN THÊM HÀNG =====
@bot.message_handler(commands=['add'])
def add_stock(m):
    if m.from_user.id != ADMIN_ID:
        return

    try:
        _, data = m.text.split(" ", 1)
        name, amount = data.split("|")
        amount = int(amount)

        if name in products:
            products[name]["stock"] += amount
            bot.send_message(m.chat.id, "✔️ Đã thêm hàng")
        else:
            bot.send_message(m.chat.id, "❌ Không có sản phẩm")

    except:
        bot.send_message(m.chat.id, "Sai cú pháp: /add tên|số")

print("Bot đang chạy...")
bot.infinity_polling()
