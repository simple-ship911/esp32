from machine import Pin, I2C
import time
from ssd1306 import SSD1306_I2C
import network
from nettest import net_auto_connect, time_get
from wifi_logo import draw_wifi_full
import _thread
from leftwindow import left_window




is_main_window = True
DEBOUNCE_TIME = 200
minute=0
hour=0
second=0
selections=["wifi","bluetooth","boot"]
tools = ["message","control","game","music"]


def is_key_pressed(pin):
    """轻量化按键检测，仅检测按下边缘，无长阻塞"""
    # 检测电平从1→0（按下瞬间）
    if pin.value() == 0:
        time.sleep_ms(50)  # 短防抖，仅50ms
        if pin.value() == 0:
            # 等待松开但超时保护（最多1秒）
            timeout = 0
            while pin.value() == 0 and timeout < 100:
                time.sleep_ms(10)
                timeout += 1
            return True
    return False




def wlan_check(wlan, oled):
    if wlan and wlan.isconnected():
        print("连接成功")
        oled.fill(0)
        draw_wifi_full(oled, 120, 1)
    else:
        pass





def refresh_time_from_ntp():
    global hour, minute, second, date, moment
    to_time = time_get()
    if to_time and isinstance(to_time, tuple) and len(to_time) >= 6:
        hour = to_time[3]
        minute = to_time[4]
        second = to_time[5]
        date = f"{to_time[0]}.{to_time[1]}.{to_time[2]}"
        moment = f"{hour} : {minute}"


def show_time(oled,date, moment):
    if is_main_window==True:
        oled.text(str(date), 0, 0, 1)
        oled.text(str(moment), 40, 26, 1)
        oled.text("set          ord", 0, 50, 1)
        oled.show()
def time_count():
    global second, minute, hour, moment, is_main_window
    while True:
        second = second + 1

        # 时间进位
        if second >= 60:
            second = 0
            minute = minute + 1
        if minute >= 60:
            hour = hour + 1
            minute = 0
        moment = f"{hour} : {minute}"
        if minute == 59 and second == 0:
            refresh_time_from_ntp()
        if is_main_window==True:
            
            show_time(oled,date, moment)
        else:
            for y in range(0, 8):
                for x in range(60, 128):
                    oled.pixel(x, y, 0)
            oled.text(moment, 60, 0, 1)  
            oled.show()
        time.sleep(1)

def main_window():
    wlan_check(wlan, oled)
    show_time(oled,date, moment)
    oled.show()
    
left_pin=Pin(23,Pin.IN,Pin.PULL_UP)
right_pin=Pin(15,Pin.IN,Pin.PULL_UP)
i2c = I2C(0, scl=Pin(18), sda=Pin(13), freq=400000)
oled = SSD1306_I2C(128, 64, i2c)
oled.fill(0)
oled.show()
# wlan初始化

wlan = network.WLAN(network.STA_IF)
wlan.active(True)
try:
    net_auto_connect("wzs", "12345678")
except OSError:
    pass

time.sleep(5)

to_time = time_get()
print(to_time)

hour = to_time[3]
minute = to_time[4]
second = to_time[5]

date = f"{to_time[0]}.{to_time[1]}.{to_time[2]}"
moment = f"{hour} : {minute}"
_thread.start_new_thread(time_count, ())

if __name__=="__main__":
    main_window()
    while True:
        current_time = time.ticks_ms()
        
        if is_key_pressed(left_pin):
            print(f"切换界面：is_main_window={is_main_window}")
            if is_main_window:
                window_name="set"
                oled.fill(0)
                left_window(oled,window_name,selections)
                oled.text(moment, 60, 0, 1)
                oled.show()
                is_main_window = False
            else:
                is_main_window = True
                oled.fill(0)
                main_window()
        if is_key_pressed(right_pin):
            print(f"切换界面：is_main_window={is_main_window}")
            if is_main_window:
                window_name="ord"
                oled.fill(0)
                left_window(oled,window_name,tools)
                oled.text(moment, 60, 0, 1)
                oled.show()
                is_main_window = False
        time.sleep_ms(10)
