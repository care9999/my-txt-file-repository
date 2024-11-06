import requests
import time
from datetime import datetime
import matplotlib.pyplot as plt
import io

LINE_TOKEN = 'zVZB3qC1Zge8RtnWN0u2rzfJ818xWIH8vTEcAXUs9uD'
EXCHANGE_RATE = 33.7
GECKO_TERMINAL_API = "https://api.geckoterminal.com/api/v2"
POOL_ADDRESS = "EQDq1cQKb6vQItJgBhwn7bS2FpJNnDLOKDUIBF8hHJgQwqlD"
COINGECKO_API = "https://api.coingecko.com/api/v3"

def send_line_notification(message, image=None):
    url = 'https://notify-api.line.me/api/notify'
    headers = {'Authorization': f'Bearer {LINE_TOKEN}'}
    
    data = {'message': message}
    files = None
    
    if image:
        files = {'imageFile': ('chart.png', image, 'image/png')}
    
    response = requests.post(url, headers=headers, data=data, files=files)
    if response.status_code == 200:
        print("Notification sent successfully.")
    else:
        print("Failed to send notification:", response.text)

def get_ton_price():
    try:
        response = requests.get(f"{COINGECKO_API}/simple/price?ids=the-open-network&vs_currencies=usd")
        if response.status_code == 200:
            data = response.json()
            return data['the-open-network']['usd']
    except Exception as e:
        print(f"Error fetching TON price: {e}")
    return 0

def get_pool_info():
    try:
        # ดึงข้อมูล pool จาก GeckoTerminal สำหรับ Catbox
        response = requests.get(f"{GECKO_TERMINAL_API}/networks/ton/pools/{POOL_ADDRESS}")
        if response.status_code == 200:
            data = response.json()
            pool_data = data['data']['attributes']
            
            # ดึงราคา Catbox
            base_token_price_usd = float(pool_data['base_token_price_usd'] or 0)
            
            # ดึงราคา TON แยกจาก CoinGecko
            ton_price = get_ton_price()
            
            return {
                'catbox': base_token_price_usd,
                'toncoin': ton_price
            }
    except Exception as e:
        print(f"Error fetching pool info: {e}")
    return {'catbox': 0, 'toncoin': 0}

def get_price_chart():
    try:
        # ดึงข้อมูลกราฟราคา
        response = requests.get(f"{GECKO_TERMINAL_API}/networks/ton/pools/{POOL_ADDRESS}/ohlcv/hour")
        if response.status_code == 200:
            data = response.json()
            chart_data = data['data']['attributes']['ohlcv_list']
            
            # แปลงข้อมูลสำหรับการสร้างกราฟ
            times = [datetime.fromtimestamp(int(x[0])/1000) for x in chart_data]
            prices = [float(x[4]) for x in chart_data]  # ใช้ราคาปิด
            
            # สร้างกราฟ
            plt.figure(figsize=(10, 6))
            plt.plot(times, prices)
            plt.title('Catbox Price Chart (24H)')
            plt.xlabel('Time')
            plt.ylabel('Price (USD)')
            plt.xticks(rotation=45)
            plt.tight_layout()
            
            # แปลงกราฟเป็น PNG
            img_buffer = io.BytesIO()
            plt.savefig(img_buffer, format='PNG')
            img_buffer.seek(0)
            plt.close()
            
            return img_buffer
    except Exception as e:
        print(f"Error creating price chart: {e}")
    return None

def main():
    while True:
        try:
            current_prices = get_pool_info()
            
            # แปลงราคาเป็น THB
            catbox_thb = current_prices['catbox'] * EXCHANGE_RATE
            toncoin_thb = current_prices['toncoin'] * EXCHANGE_RATE

            # สร้างข้อความแจ้งเตือน
            message = (
                f"\n📊 [แจ้งเตือนราคา]\n"
                f"⏰ เวลา: {datetime.now().strftime('%Y-%m-%d %H:%M')}\n"
                f"🐱 Catbox: {current_prices['catbox']:.4f} USD ({catbox_thb:.2f} บาท)\n"
                f"💎 Toncoin: {current_prices['toncoin']:.4f} USD ({toncoin_thb:.2f} บาท)\n"
                f"🔄 ที่มา: GeckoTerminal + CoinGecko"
            )

            # สร้างและส่งกราฟ
            chart_buffer = get_price_chart()
            if chart_buffer:
                send_line_notification(message, chart_buffer)
            else:
                send_line_notification(message)
            
        except Exception as e:
            print(f"Error in main loop: {e}")
            send_line_notification(f"\n⚠️ เกิดข้อผิดพลาด: {str(e)}")
        
        # รอ 60 วินาที
        time.sleep(60)

if __name__ == "__main__":
    main()
