#!/usr/bin/env python3
"""
👑 KING REPORT BOT - FINAL VERSION 👑
==================================================
✅ 2 EMAIL PENGIRIM (pembantaiponzii & abibanyu88)
✅ TANPA MENAMPILKAN EMAIL DI LAPORAN
✅ HANYA MENAMPILKAN JUMLAH BERHASIL/GAGAL
✅ MONITORING STATUS DOMAIN
✅ 3 BUKTI FOTO
==================================================
"""

import telebot
from telebot.types import InlineKeyboardMarkup, InlineKeyboardButton
import smtplib
import ssl
import requests
import dns.resolver
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from email.mime.image import MIMEImage
from datetime import datetime
import os
import json
import time
import threading

# ==================================================
# 🔧 KONFIGURASI 🔧
# ==================================================
BOT_TOKEN = "8618860963:AAHwyiyK1efOXOvSfmtdf0GW4pvyuYOUSUY"
ADMIN_ID = "8085805782"
DATA_FILE = "reports.json"
MONITOR_FILE = "monitor.json"

# ==================================================
# 📧 EMAIL PENGIRIM (2 AKUN)
# ==================================================
EMAIL_SENDERS = [
    {
        "email": "pembantaiponzii@gmail.com",
        "password": "qrbistvuwlsfhyei",
        "smtp_server": "smtp.gmail.com",
        "smtp_port": 587
    },
    {
        "email": "abibanyu88@gmail.com",
        "password": "ddxucazzfelyuyxm",
        "smtp_server": "smtp.gmail.com",
        "smtp_port": 587
    }
]

# ==================================================
# 📧 EMAIL TUJUAN (PENERIMA LAPORAN)
# ==================================================
IMPORTANT_EMAILS = [
    "abuse@hostinger.com",
    "domainabuse@service.aliyun.com",
    "abuse@namecheap.com",
    "abuse@godaddy.com",
    "abuse@cloudflare.com",
    "reportphishing@apwg.org",
]

def get_target_emails(domain):
    emails = IMPORTANT_EMAILS.copy()
    emails.append(f"abuse@{domain}")
    emails.append(f"admin@{domain}")
    emails.append(f"security@{domain}")
    return list(set(emails))

# ==================================================
# 🔍 MONITORING STATUS DOMAIN
# ==================================================

def cek_status_domain(domain):
    try:
        domain = domain.replace('http://', '').replace('https://', '').split('/')[0]
        
        try:
            dns.resolver.resolve(domain, 'A')
            dns_ok = True
        except:
            dns_ok = False
        
        try:
            r = requests.get(f"https://{domain}", timeout=10)
            http_status = r.status_code
            page_text = r.text.lower()
        except:
            http_status = 0
            page_text = ""
        
        suspend_keywords = [
            'domain is parked', 'parked domain', 'domain suspended', 
            'account suspended', 'this domain is not active', 
            'domain has been suspended', 'suspended page',
            'this site is currently unavailable', 'website is offline',
            'coming soon', 'under construction', 'default page',
            'hostinger parking', 'godaddy parking', 'namecheap parking'
        ]
        
        is_suspended = any(kw in page_text for kw in suspend_keywords)
        
        if not dns_ok:
            status = "BLOCKIR"
            icon = "🔴"
            message = "DNS tidak merespon - Domain kemungkinan di blokir"
        elif http_status in [403, 404, 410, 451, 500, 502, 503]:
            status = "BLOCKIR"
            icon = "🔴"
            message = f"HTTP {http_status} - Domain diblokir"
        elif is_suspended:
            status = "SUSPEND"
            icon = "🟠"
            message = "Halaman parkir/suspend terdeteksi"
        else:
            status = "AKTIF"
            icon = "🟢"
            message = "Domain masih aktif"
        
        return {
            "domain": domain,
            "status": status,
            "icon": icon,
            "message": message,
            "last_check": datetime.now().isoformat()
        }
    except Exception as e:
        return {
            "domain": domain,
            "status": "ERROR",
            "icon": "⚪",
            "message": f"Error: {str(e)}",
            "last_check": datetime.now().isoformat()
        }

def load_monitor():
    try:
        with open(MONITOR_FILE, 'r') as f:
            return json.load(f)
    except:
        return {}

def save_monitor(monitor):
    with open(MONITOR_FILE, 'w') as f:
        json.dump(monitor, f, indent=4)

def start_monitoring():
    def monitor_loop():
        while True:
            try:
                monitor = load_monitor()
                for domain, data in list(monitor.items()):
                    new_status = cek_status_domain(domain)
                    old_status = data.get("status", "")
                    
                    if new_status["status"] != old_status:
                        notif = f"""
{new_status['icon']} **PERUBAHAN STATUS DOMAIN!**
━━━━━━━━━━━━━━━━━━━━━━━
📬 **DOMAIN:** `{domain}`
📊 **STATUS LAMA:** {old_status}
📊 **STATUS BARU:** {new_status['status']}
📝 **KETERANGAN:** {new_status['message']}
🕐 **WAKTU:** {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
━━━━━━━━━━━━━━━━━━━━━━━
                        """
                        bot.send_message(ADMIN_ID, notif, parse_mode="Markdown")
                        
                        data["status"] = new_status["status"]
                        data["last_check"] = new_status["last_check"]
                        data["history"].append({
                            "timestamp": datetime.now().isoformat(),
                            "old": old_status,
                            "new": new_status["status"]
                        })
                        save_monitor(monitor)
                
                time.sleep(21600)
            except Exception as e:
                print(f"Monitor error: {e}")
                time.sleep(3600)
    
    thread = threading.Thread(target=monitor_loop, daemon=True)
    thread.start()

# ==================================================
# 📧 KIRIM EMAIL
# ==================================================

def kirim_laporan(sender_info, domain, data, recipient):
    try:
        msg = MIMEMultipart()
        msg['From'] = sender_info['email']
        msg['To'] = recipient
        msg['Subject'] = f"[URGENT] LAPORAN PENIPUAN - {domain}"
        
        body = f"""
╔══════════════════════════════════════════════════════════╗
║                 LAPORAN PENIPUAN SKEMA PONZI              ║
╚══════════════════════════════════════════════════════════╝

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📋 **DETAIL DOMAIN**
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DOMAIN      : {domain}
LOGIN       : {data.get('login_url', '-')}
DEPOSIT     : {data.get('deposit_url', '-')}
REFERRAL    : {data.get('referral_url', '-')}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📌 **JENIS PELANGGARAN**
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• Menjanjikan keuntungan tidak realistis
• Sistem referral berjenjang (MLM)
• Tidak memiliki izin dari otoritas
• Menyebabkan kerugian materiil bagi masyarakat

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🖼 **BUKTI LAMPIRAN**
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Terdapat {len(data.get('images', []))} gambar bukti terlampir.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Harap segera ditindaklanjuti untuk melindungi masyarakat dari
kerugian lebih lanjut.

Terima kasih.

☕ Tetap jaga internet tetap aman!
        """
        
        msg.attach(MIMEText(body, 'plain'))
        
        for img_id in data.get('images', []):
            try:
                file_info = bot.get_file(img_id)
                img = MIMEImage(bot.download_file(file_info.file_path))
                img.add_header('Content-Disposition', 'attachment', filename='evidence.jpg')
                msg.attach(img)
            except:
                pass
        
        context = ssl.create_default_context()
        with smtplib.SMTP(sender_info['smtp_server'], sender_info['smtp_port']) as server:
            server.starttls(context=context)
            server.login(sender_info['email'], sender_info['password'])
            server.send_message(msg)
        return True
    except Exception as e:
        print(f"Gagal kirim ke {recipient} dari {sender_info['email']}: {e}")
        return False

def kirim_ke_semua_email(target_emails, domain, data, chat_id):
    status_msg = bot.send_message(chat_id, f"📧 Mengirim laporan...")
    
    success_count = 0
    failed_count = 0
    total_emails = len(target_emails) * len(EMAIL_SENDERS)
    current = 0
    
    for sender in EMAIL_SENDERS:
        for email in target_emails:
            current += 1
            bot.edit_message_text(
                f"📧 Mengirim... [{current}/{total_emails}]",
                chat_id,
                status_msg.message_id
            )
            
            if kirim_laporan(sender, domain, data, email):
                success_count += 1
            else:
                failed_count += 1
            
            time.sleep(1)
    
    bot.delete_message(chat_id, status_msg.message_id)
    return success_count, failed_count

# ==================================================
# 🎨 TOMBOL COMMAND
# ==================================================

def menu_utama():
    markup = InlineKeyboardMarkup(row_width=2)
    markup.add(
        InlineKeyboardButton("📋 LAPOR", callback_data="lapor"),
        InlineKeyboardButton("🔍 MONITOR", callback_data="monitor"),
        InlineKeyboardButton("📜 RIWAYAT", callback_data="riwayat"),
        InlineKeyboardButton("🔍 CEK", callback_data="cek"),
        InlineKeyboardButton("❓ BANTUAN", callback_data="bantuan")
    )
    return markup

# ==================================================
# 🤖 HANDLER
# ==================================================

bot = telebot.TeleBot(BOT_TOKEN)
user_sessions = {}

@bot.message_handler(commands=['start'])
def start(message):
    if str(message.from_user.id) != ADMIN_ID:
        bot.reply_to(message, "⛔ Tidak diizinkan!")
        return
    
    welcome = """
👑 **KING REPORT BOT** 👑
━━━━━━━━━━━━━━━━━━━━━━━

📋 **CARA PAKAI:**
1. Klik LAPOR
2. Masukkan domain target
3. Masukkan link LOGIN
4. Masukkan link DEPOSIT
5. Masukkan link REFERRAL
6. Upload 3 foto bukti

☕ **Tetap jaga internet tetap aman!**
    """
    bot.send_message(message.chat.id, welcome, parse_mode="Markdown", reply_markup=menu_utama())

@bot.callback_query_handler(func=lambda call: True)
def callback(call):
    if str(call.from_user.id) != ADMIN_ID:
        bot.answer_callback_query(call.id, "⛔ Tidak diizinkan!")
        return
    
    if call.data == "lapor":
        user_sessions[call.message.chat.id] = {"step": "domain"}
        bot.edit_message_text(
            "📋 **BUAT LAPORAN**\n━━━━━━━━━━━━━━━━━━━━━━━\n\n"
            "**1️⃣ Masukkan DOMAIN target:**\n"
            "Contoh: `example.com`",
            call.message.chat.id,
            call.message.message_id,
            parse_mode="Markdown"
        )
        bot.register_next_step_handler(call.message, input_domain)
    
    elif call.data == "monitor":
        monitor_list = load_monitor()
        if not monitor_list:
            text = "🔍 Belum ada domain yang dimonitor."
        else:
            text = "🔍 **MONITORING DOMAIN**\n━━━━━━━━━━━━━━━━━━━━━━━\n\n"
            for domain, info in monitor_list.items():
                status = info.get("status", "UNKNOWN")
                icon = "🟢" if status == "AKTIF" else "🟠" if status == "SUSPEND" else "🔴" if status == "BLOCKIR" else "⚪"
                text += f"{icon} **{domain}** - {status}\n"
        bot.edit_message_text(text, call.message.chat.id, call.message.message_id, parse_mode="Markdown", reply_markup=menu_utama())
    
    elif call.data == "riwayat":
        try:
            with open(DATA_FILE, 'r') as f:
                reports = json.load(f)
            text = "📜 **RIWAYAT LAPORAN**\n━━━━━━━━━━━━━━━━━━━━━━━\n\n"
            for r in reports[-10:]:
                text += f"• {r['domain']} - {r['timestamp'][:10]}\n"
                text += f"  ✅ {r.get('email_success', 0)}/{r.get('email_total', 0)}\n\n"
            bot.edit_message_text(text, call.message.chat.id, call.message.message_id, parse_mode="Markdown")
        except:
            bot.edit_message_text("Belum ada laporan", call.message.chat.id, call.message.message_id)
    
    elif call.data == "cek":
        bot.edit_message_text(
            "🔍 **CEK DOMAIN**\n━━━━━━━━━━━━━━━━━━━━━━━\n\n"
            "Masukkan domain:\nContoh: `example.com`",
            call.message.chat.id,
            call.message.message_id,
            parse_mode="Markdown"
        )
        bot.register_next_step_handler(call.message, cek_domain)
    
    elif call.data == "bantuan":
        bot.edit_message_text(
            "❓ **BANTUAN**\n━━━━━━━━━━━━━━━━━━━━━━━\n\n"
            "**📋 MEMBUAT LAPORAN:**\n"
            "1. Klik LAPOR\n"
            "2. Masukkan domain target\n"
            "3. Masukkan link LOGIN\n"
            "4. Masukkan link DEPOSIT\n"
            "5. Masukkan link REFERRAL\n"
            "6. Upload 3 foto bukti\n\n"
            "**🔍 MONITORING:**\n"
            "• Domain otomatis dimonitor setelah dilaporkan\n"
            "• Bot cek setiap 6 jam\n"
            "• Notifikasi jika status berubah (SUSPEND/BLOCKIR)\n\n"
            "━━━━━━━━━━━━━━━━━━━━━━━\n"
            "☕ **Tetap jaga internet tetap aman!**",
            call.message.chat.id,
            call.message.message_id,
            parse_mode="Markdown"
        )

def input_domain(message):
    chat_id = message.chat.id
    session = user_sessions.get(chat_id, {})
    session["domain"] = message.text.strip()
    session["step"] = "login"
    user_sessions[chat_id] = session
    
    bot.send_message(
        chat_id,
        "**2️⃣ Masukkan LINK LOGIN:**\nContoh: `https://example.com/login`",
        parse_mode="Markdown"
    )
    bot.register_next_step_handler(message, input_login)

def input_login(message):
    chat_id = message.chat.id
    session = user_sessions.get(chat_id, {})
    session["login_url"] = message.text.strip()
    session["step"] = "deposit"
    user_sessions[chat_id] = session
    
    bot.send_message(
        chat_id,
        "**3️⃣ Masukkan LINK DEPOSIT:**\nContoh: `https://example.com/deposit`",
        parse_mode="Markdown"
    )
    bot.register_next_step_handler(message, input_deposit)

def input_deposit(message):
    chat_id = message.chat.id
    session = user_sessions.get(chat_id, {})
    session["deposit_url"] = message.text.strip()
    session["step"] = "referral"
    user_sessions[chat_id] = session
    
    bot.send_message(
        chat_id,
        "**4️⃣ Masukkan LINK REFERRAL:**\nContoh: `https://example.com/ref/123`",
        parse_mode="Markdown"
    )
    bot.register_next_step_handler(message, input_referral)

def input_referral(message):
    chat_id = message.chat.id
    session = user_sessions.get(chat_id, {})
    session["referral_url"] = message.text.strip()
    session["step"] = "images"
    session["images"] = []
    user_sessions[chat_id] = session
    
    bot.send_message(
        chat_id,
        "**5️⃣ Kirim 3 GAMBAR BUKTI**\n\n"
        "📸 Kirim gambar satu per satu (3 gambar)\n"
        "💡 Ketik `skip` jika tidak ada gambar",
        parse_mode="Markdown"
    )
    bot.register_next_step_handler(message, input_images)

def input_images(message):
    chat_id = message.chat.id
    session = user_sessions.get(chat_id, {})
    
    if not session:
        bot.send_message(chat_id, "❌ Sesi habis. Mulai ulang dengan /start")
        return
    
    if message.text and message.text.lower() == "skip":
        session["images"] = []
        user_sessions[chat_id] = session
        kirim_laporan_akhir(message, chat_id, session)
        return
    
    if message.photo:
        images = session.get("images", [])
        images.append(message.photo[-1].file_id)
        session["images"] = images
        user_sessions[chat_id] = session
        
        if len(images) < 3:
            bot.send_message(chat_id, f"📸 Gambar {len(images)}/3 diterima.\nKirim gambar ke-{len(images)+1}")
            bot.register_next_step_handler(message, input_images)
        else:
            bot.send_message(chat_id, "✅ 3 gambar diterima! Memproses laporan...")
            kirim_laporan_akhir(message, chat_id, session)
    else:
        bot.send_message(chat_id, "❌ Kirim GAMBAR atau ketik 'skip'")
        bot.register_next_step_handler(message, input_images)

def kirim_laporan_akhir(message, chat_id, session):
    domain = session.get("domain")
    target_emails = get_target_emails(domain)
    
    data = {
        "login_url": session.get("login_url"),
        "deposit_url": session.get("deposit_url"),
        "referral_url": session.get("referral_url"),
        "images": session.get("images", [])
    }
    
    success_count, failed_count = kirim_ke_semua_email(target_emails, domain, data, chat_id)
    total_emails = len(target_emails) * len(EMAIL_SENDERS)
    
    monitor = load_monitor()
    if domain not in monitor:
        monitor[domain] = {
            "domain": domain,
            "status": "AKTIF",
            "last_check": datetime.now().isoformat(),
            "history": []
        }
        save_monitor(monitor)
    
    status_domain = cek_status_domain(domain)
    
    report = {
        "domain": domain,
        "timestamp": datetime.now().isoformat(),
        "login": data["login_url"],
        "deposit": data["deposit_url"],
        "referral": data["referral_url"],
        "email_success": success_count,
        "email_total": total_emails,
        "domain_status": status_domain["status"]
    }
    try:
        with open(DATA_FILE, 'r') as f:
            reports = json.load(f)
    except:
        reports = []
    reports.append(report)
    with open(DATA_FILE, 'w') as f:
        json.dump(reports, f, indent=4)
    
    status_icon = "✅" if success_count > 0 else "❌"
    
    laporan_text = f"""
{status_icon} **LAPORAN TERKIRIM!**
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📬 **DOMAIN:** `{domain}`
📧 **TERKIRIM:** {success_count}/{total_emails}
🖼 **BUKTI:** {len(data['images'])} gambar
📊 **STATUS DOMAIN:** {status_domain['icon']} **{status_domain['status']}**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔗 **LOGIN:** {data['login_url']}
💰 **DEPOSIT:** {data['deposit_url']}
👥 **REFERRAL:** {data['referral_url']}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
"""
    
    if failed_count > 0:
        laporan_text += f"\n⚠️ **{failed_count} email gagal dikirim**\n"
    
    laporan_text += "\n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n"
    laporan_text += "🔍 **DOMAIN AKAN DIMONITOR OTOMATIS**\n"
    laporan_text += "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n"
    laporan_text += "☕ **Terima kasih telah menjaga internet tetap aman!**"
    
    bot.send_message(chat_id, laporan_text, parse_mode="Markdown")
    
    for img in data['images']:
        try:
            bot.send_photo(chat_id, img)
        except:
            pass
    
    if chat_id in user_sessions:
        del user_sessions[chat_id]
    
    bot.send_message(chat_id, "Kembali ke menu utama...", reply_markup=menu_utama())

def cek_domain(message):
    chat_id = message.chat.id
    domain = message.text.strip()
    
    status = cek_status_domain(domain)
    emails = get_target_emails(domain)
    
    text = f"""
🔍 **HASIL CEK DOMAIN**
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📬 **DOMAIN:** `{domain}`

📊 **STATUS:** {status['icon']} **{status['status']}**
📝 **KETERANGAN:** {status['message']}
🕐 **WAKTU:** {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📧 **EMAIL TUJUAN ({len(emails)}):**
"""
    for e in emails:
        text += f"• `{e}`\n"
    
    bot.send_message(chat_id, text, parse_mode="Markdown", reply_markup=menu_utama())

# ==================================================
# 🚀 MAIN
# ==================================================
if __name__ == "__main__":
    print("="*60)
    print("👑 KING REPORT BOT - FINAL VERSION")
    print("="*60)
    print("📧 Email Pengirim (AKTIF):")
    for sender in EMAIL_SENDERS:
        print(f"   - {sender['email']} ✅")
    print("="*60)
    print("Bot berjalan...")
    print("Monitoring status domain aktif (setiap 6 jam)")
    
    start_monitoring()
    
    try:
        bot.infinity_polling()
    except KeyboardInterrupt:
        print("\n👋 Bot dihentikan")
