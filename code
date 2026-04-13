

import telebot
import requests

# --- КОНФИГУРАЦИЯ ---
TG_BOT_TOKEN = 'token'
ZABBIX_URL = 'http://0.0.0.0:8080'
ZABBIX_API_TOKEN = 'zabbix token'

bot = telebot.TeleBot(TG_BOT_TOKEN)

HEADERS = {
    'Content-Type': 'application/json-rpc',
    'Authorization': f'Bearer {ZABBIX_API_TOKEN}'
}

def zabbix_api_request(method, params):
    payload = {"jsonrpc": "2.0", "method": method, "params": params, "id": 1}
    response = requests.post(f"{ZABBIX_URL}/api_jsonrpc.php", json=payload, headers=HEADERS)
    return response.json().get('result', [])

@bot.message_handler(commands=['start', 'list'])
def send_host_list(message):
    chat_id = message.chat.id
    bot.send_message(chat_id, "Ищу узлы в Zabbix...")
    
    hosts = zabbix_api_request("host.get", {
        "output": ["hostid", "name"],
        "selectInterfaces": ["ip"]
    })
    
    if not hosts:
        bot.send_message(chat_id, "Узлы не найдены.")
        return

    response_text = "<b>Список агентов:</b>\n\n"
    for h in hosts:
        ip = h['interfaces'][0]['ip'] if h.get('interfaces') else 'Нет IP'
        response_text += f"[>] <code>{h['name']}</code> ({ip})\n"
        
    response_text += "\nОтправьте имя хоста для запроса данных."
    bot.send_message(chat_id, response_text, parse_mode='HTML')

@bot.message_handler(content_types=['text'])
def send_dashboard(message):
    chat_id = message.chat.id
    host_name = message.text.strip()

    hosts = zabbix_api_request("host.get", {
        "output": ["hostid", "name"],
        "filter": {"name": host_name}
    })

    if not hosts:
        bot.send_message(chat_id, f"Хост <code>{host_name}</code> не найден.", parse_mode='HTML')
        return

    host_id = hosts[0]['hostid']
    bot.send_message(chat_id, f"Запрос данных: <b>{host_name}</b>...", parse_mode='HTML')

    items = zabbix_api_request("item.get", {
        "output": ["itemid", "name", "key_", "lastvalue", "units"],
        "hostids": host_id,
        "limit": 500
    })

    if not items:
        bot.send_message(chat_id, "Метрики не найдены.")
        return

    allowed_keys = [
        "cpu.util", "system.cpu.util", "vm.cpu.util", 
        "vm.memory.util", "system.memory.utilization", 
        "vfs.fs.size", "vfs.fs.dependent.size",         
        "net.if.in", "net.if.out"                     
    ]

    response_text = f"=== {host_name} ===\n\n"
    
    for item in items:
        key = item['key_'].lower()
        
        if not any(key.startswith(allowed) for allowed in allowed_keys):
            continue
            
        if 'vfs.fs' in key:
            if ',pfree' not in key and ',free' not in key and ',pused' not in key:
                continue
                
        if 'net.if' in key:
            if 'dropped' in key or 'errors' in key or 'speed' in key or 'status' in key:
                continue
            
        value = item.get('lastvalue', 'N/A')
        unit = item.get('units', '')
        name = item.get('name', item['key_'])
        
        # Стильные ASCII-теги
        if 'cpu' in key:
            icon = '[CPU]'
        elif 'memory' in key:
            icon = '[RAM]'
        elif 'net.if.in' in key:
            icon = '[IN ]'
        elif 'net.if.out' in key:
            icon = '[OUT]'
        else:
            icon = '[HDD]' 
       
        formatted_name = f"{name}:".ljust(40)
        response_text += f"{icon} {formatted_name} {value} {unit}\n"

    bot.send_message(chat_id, f"<pre>{response_text}</pre>", parse_mode='HTML')

if name == 'main':
    print("Bot started (ASCII mode)...")
    bot.infinity_polling()
