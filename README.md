# --- Parça 1 Başlangıcı ---
import pyautogui
import time
import numpy as np
import cv2
import random
import logging
import os
import threading
import tkinter as tk
from tkinter import scrolledtext, ttk, messagebox
import keyboard
import psutil
import platform
import csv
from pynput import mouse as pynput_mouse
from pynput import keyboard as pynput_keyboard
from PIL import Image, ImageDraw
import ctypes
import screeninfo
from ctypes import wintypes
try:
    import matplotlib.pyplot as plt
    from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
    MATPLOTLIB_AVAILABLE = True
except ImportError:
    MATPLOTLIB_AVAILABLE = False

# Loglama ayarları
logging.basicConfig(
    filename='merged_bot_log.txt',
    level=logging.DEBUG,
    format='%(asctime)s - %(levelname)s - %(message)s',
    encoding='utf-8'
)

# Dizinler
SCRIPT_BASE_DIR = 'scripts'
OBJECTS_DIR = 'objeler'
os.makedirs(SCRIPT_BASE_DIR, exist_ok=True)
os.makedirs(OBJECTS_DIR, exist_ok=True)

# Global değişkenler
EMERGENCY_STOP = False
MOUSE_SPEED = 125      # Hata çözümü için eklendi (BornozBot.py referansı)
MOUSE_SPEED_MIN = 150    # ms
MOUSE_SPEED_MAX = 220   # ms
TOTAL_CLICKS = 0
POPUP_CLOSED_COUNT = 0
PRIORITY_OBJ1_CLICKS = 0
PRIORITY_OBJ2_CLICKS = 0
MOUSE_SPEED_MIN_LIMIT = 50  # Minimum hız sınırı
MOUSE_SPEED_MAX_LIMIT = 500  # Maksimum hız sınırı
F11_MODE = False  # F11 modu varsayılan olarak kapalı
# --- Parça 1 Sonu ---

# --- Parça 2 Başlangıcı ---
# Windows API tanımlamaları, yapı ve fonksiyonlar
user32 = ctypes.WinDLL('user32', use_last_error=True)
kernel32 = ctypes.WinDLL('kernel32', use_last_error=True)

INPUT_MOUSE = 0
INPUT_KEYBOARD = 1
INPUT_HARDWARE = 2

KEYEVENTF_EXTENDEDKEY = 0x0001
KEYEVENTF_KEYUP = 0x0002
KEYEVENTF_UNICODE = 0x0004
KEYEVENTF_SCANCODE = 0x0008

MOUSEEVENTF_MOVE = 0x0001
MOUSEEVENTF_LEFTDOWN = 0x0002
MOUSEEVENTF_LEFTUP = 0x0004
MOUSEEVENTF_RIGHTDOWN = 0x0008
MOUSEEVENTF_RIGHTUP = 0x0010
MOUSEEVENTF_MIDDLEDOWN = 0x0020
MOUSEEVENTF_MIDDLEUP = 0x0040
MOUSEEVENTF_WHEEL = 0x0800
MOUSEEVENTF_HWHEEL = 0x1000
MOUSEEVENTF_ABSOLUTE = 0x8000

VK_SPACE = 0x20
VK_PRIOR = 0x21
VK_NEXT = 0x22
VK_END = 0x23
VK_HOME = 0x24
VK_LEFT = 0x25
VK_UP = 0x26
VK_RIGHT = 0x27
VK_DOWN = 0x28
VK_TAB = 0x09
VK_SHIFT = 0x10

class KEYBDINPUT(ctypes.Structure):
    _fields_ = [
        ("wVk", wintypes.WORD),
        ("wScan", wintypes.WORD),
        ("dwFlags", wintypes.DWORD),
        ("time", wintypes.DWORD),
        ("dwExtraInfo", ctypes.POINTER(ctypes.c_ulong))
    ]

class MOUSEINPUT(ctypes.Structure):
    _fields_ = [
        ("dx", wintypes.LONG),
        ("dy", wintypes.LONG),
        ("mouseData", wintypes.DWORD),
        ("dwFlags", wintypes.DWORD),
        ("time", wintypes.DWORD),
        ("dwExtraInfo", ctypes.POINTER(ctypes.c_ulong))
    ]

class HARDWAREINPUT(ctypes.Structure):
    _fields_ = [
        ("uMsg", wintypes.DWORD),
        ("wParamL", wintypes.WORD),
        ("wParamH", wintypes.WORD)
    ]

class INPUT_union(ctypes.Union):
    _fields_ = [
        ("ki", KEYBDINPUT),
        ("mi", MOUSEINPUT),
        ("hi", HARDWAREINPUT)
    ]

class INPUT(ctypes.Structure):
    _fields_ = [
        ("type", wintypes.DWORD),
        ("ii", INPUT_union)
    ]

def send_key_event(key_code, key_down=True):
    extra = ctypes.c_ulong(0)
    ii_ = INPUT_union()
    ii_.ki = KEYBDINPUT(key_code, 0, 0 if key_down else KEYEVENTF_KEYUP, 0, ctypes.pointer(extra))
    x = INPUT(INPUT_KEYBOARD, ii_)
    ctypes.windll.user32.SendInput(1, ctypes.byref(x), ctypes.sizeof(x))

def normalize_coords(x, y):
    screen_width, screen_height = pyautogui.size()
    normalized_x = int(x * 65536 / screen_width)
    normalized_y = int(y * 65536 / screen_height)
    return normalized_x, normalized_y

def send_mouse_event(x, y, flags, mouse_data=0):
    screen_width, screen_height = pyautogui.size()
    norm_x, norm_y = normalize_coords(x, y)
    extra = ctypes.c_ulong(0)
    ii_ = INPUT_union()
    ii_.mi = MOUSEINPUT(norm_x, norm_y, mouse_data, flags | MOUSEEVENTF_ABSOLUTE, 0, ctypes.pointer(extra))
    inp = INPUT(INPUT_MOUSE, ii_)
    ctypes.windll.user32.SendInput(1, ctypes.byref(inp), ctypes.sizeof(inp))

def send_mouse_wheel(direction, amount=120):
    try:
        if direction == 'down':
            mouse_data = -amount
            flags = MOUSEEVENTF_WHEEL
        elif direction == 'up':
            mouse_data = amount
            flags = MOUSEEVENTF_WHEEL
        elif direction == 'right':
            mouse_data = -amount
            flags = MOUSEEVENTF_HWHEEL
        elif direction == 'left':
            mouse_data = amount
            flags = MOUSEEVENTF_HWHEEL
        else:
            return False
        center_x, center_y = pyautogui.size()[0]//2, pyautogui.size()[1]//2
        send_mouse_event(center_x, center_y, MOUSEEVENTF_MOVE)
        time.sleep(0.01)
        send_mouse_event(center_x, center_y, flags, mouse_data)
        time.sleep(0.01)
        return True
    except Exception as e:
        logging.error(f"Fare tekerleği hatası: {str(e)}")
        return False
# --- Parça 2 Sonu ---

# --- Parça 3 Başlangıcı ---
# Ekran boyutları ve ölçeklendirme
def get_scaling_factor():
    try:
        user32 = ctypes.WinDLL('user32', use_last_error=True)
        user32.SetProcessDPIAware()
        dpi = user32.GetDpiForSystem()
        return dpi / 96.0
    except Exception as e:
        logging.error(f"Ölçeklendirme faktörü alınamadı: {str(e)}")
        return 1.0

SCALE_FACTOR = get_scaling_factor()
monitor = screeninfo.get_monitors()[0]
SCREEN_WIDTH, SCREEN_HEIGHT = monitor.width, monitor.height
EFFECTIVE_WIDTH = int(SCREEN_WIDTH / SCALE_FACTOR)
EFFECTIVE_HEIGHT = int(SCREEN_HEIGHT / SCALE_FACTOR)

# Normal mod için yasak bölgeler
NORMAL_UPPER_FORBIDDEN_HEIGHT = int(EFFECTIVE_HEIGHT * 0.15)
NORMAL_LOWER_FORBIDDEN_HEIGHT = int(EFFECTIVE_HEIGHT * 0.95)
NORMAL_RIGHT_UPPER_QUADRANT_X = EFFECTIVE_WIDTH // 2
NORMAL_RIGHT_UPPER_QUADRANT_Y = EFFECTIVE_HEIGHT // 2

# F11 modu için yasak bölgeler
F11_UPPER_FORBIDDEN_HEIGHT = int(EFFECTIVE_HEIGHT * 0.05)  # Daha küçük üst sınır
F11_LOWER_FORBIDDEN_HEIGHT = int(EFFECTIVE_HEIGHT * 0.98)  # Daha büyük alt sınır
F11_RIGHT_UPPER_QUADRANT_X = EFFECTIVE_WIDTH // 2
F11_RIGHT_UPPER_QUADRANT_Y = EFFECTIVE_HEIGHT // 2

def get_forbidden_areas():
    """Moda göre yasak bölgeleri döndür."""
    if F11_MODE:
        return (F11_UPPER_FORBIDDEN_HEIGHT, F11_LOWER_FORBIDDEN_HEIGHT, 
                F11_RIGHT_UPPER_QUADRANT_X, F11_RIGHT_UPPER_QUADRANT_Y)
    else:
        return (NORMAL_UPPER_FORBIDDEN_HEIGHT, NORMAL_LOWER_FORBIDDEN_HEIGHT, 
                NORMAL_RIGHT_UPPER_QUADRANT_X, NORMAL_RIGHT_UPPER_QUADRANT_Y)

# İlk başlatmada yasak bölgeleri güncelle
UPPER_FORBIDDEN_HEIGHT, LOWER_FORBIDDEN_HEIGHT, RIGHT_UPPER_QUADRANT_X, RIGHT_UPPER_QUADRANT_Y = get_forbidden_areas()

# Varsayılan değerler
DEFAULT_RECORD_DURATION = 30

# PyAutoGUI ayarları
pyautogui.FAILSAFE = True
pyautogui.PAUSE = 0.001
# --- Parça 3 Sonu ---

# --- Parça 4 Başlangıcı ---
def scroll_with_mouse_wheel(direction, scroll_amount=5, log_callback=None):
    try:
        if log_callback:
            log_callback(f"{direction} kaydırma yapılıyor... ({scroll_amount} kez)")
        for _ in range(scroll_amount):
            send_mouse_wheel(direction, 120)
            time.sleep(0.02)
        return True
    except Exception as e:
        if log_callback:
            log_callback(f"Kaydırma hatası: {str(e)}")
        return False

def mouse_drag_absolute(start_x, start_y, end_x, end_y, duration=0.2, steps=20):
    send_mouse_event(start_x, start_y, MOUSEEVENTF_MOVE)
    time.sleep(0.01)
    send_mouse_event(start_x, start_y, MOUSEEVENTF_LEFTDOWN)
    time.sleep(0.01)
    step_x = (end_x - start_x) / steps
    step_y = (end_y - start_y) / steps
    step_time = duration / steps
    for i in range(1, steps + 1):
        current_x = start_x + step_x * i
        current_y = start_y + step_y * i
        send_mouse_event(current_x, current_y, MOUSEEVENTF_MOVE)
        time.sleep(step_time)
    send_mouse_event(end_x, end_y, MOUSEEVENTF_LEFTUP)
    time.sleep(0.01)

def scroll_with_drag(direction, screen_width, screen_height, distance=300, log_callback=None):
    try:
        if log_callback:
            log_callback(f"Sürükleme ile {direction} kaydırma yapılıyor...")
        safe_top = UPPER_FORBIDDEN_HEIGHT
        safe_bottom = LOWER_FORBIDDEN_HEIGHT
        safe_left = 0
        safe_right = screen_width
        scrollbar_thickness = 10
        scrollbar_buffer = 5 if not F11_MODE else 0  # F11 modunda buffer 0, tam sürgü hedeflenecek
        if direction == 'down':
            start_x = screen_width - scrollbar_thickness // 2
            start_y = (safe_top + safe_bottom) // 2
            end_x = start_x
            end_y = min(start_y + distance, safe_bottom - scrollbar_buffer)
            if start_y >= end_y:
                if log_callback:
                    log_callback("Dikey aşağı kaydırma sınırda, işlem atlandı.")
                return False
            mouse_drag_absolute(start_x, start_y, end_x, end_y, 0.2)
        elif direction == 'up':
            start_x = screen_width - scrollbar_thickness // 2
            start_y = (safe_top + safe_bottom) // 2
            end_x = start_x
            end_y = max(start_y - distance, safe_top + scrollbar_buffer)
            if start_y <= end_y:
                if log_callback:
                    log_callback("Dikey yukarı kaydırma sınırda, işlem atlandı.")
                return False
            mouse_drag_absolute(start_x, start_y, end_x, end_y, 0.2)
        elif direction == 'right':
            start_x = screen_width // 2
            # F11 modunda start_y'yi 10-15 pixel yukarı çek
            start_y = screen_height - scrollbar_thickness - (scrollbar_buffer if not F11_MODE else random.randint(10, 15))
            if start_y > safe_bottom:
                start_y = safe_bottom - (scrollbar_buffer if not F11_MODE else random.randint(10, 15))
            end_x = min(start_x + distance, safe_right - scrollbar_thickness - (scrollbar_buffer if not F11_MODE else random.randint(10, 15)))
            end_y = start_y
            if start_x >= end_x:
                if log_callback:
                    log_callback("Yatay sağ kaydırma sınırda, işlem atlandı.")
                return False
            mouse_drag_absolute(start_x, start_y, end_x, end_y, 0.15 if F11_MODE else 0.2, steps=30 if F11_MODE else 20)  # F11 modunda daha hassas
        elif direction == 'left':
            start_x = screen_width // 2
            start_y = screen_height - scrollbar_thickness - (scrollbar_buffer if not F11_MODE else random.randint(10, 15))
            if start_y > safe_bottom:
                start_y = safe_bottom - (scrollbar_buffer if not F11_MODE else random.randint(10, 15))
            end_x = max(start_x - distance, safe_left + (scrollbar_buffer if not F11_MODE else random.randint(10, 15)))
            end_y = start_y
            if start_x <= end_x:
                if log_callback:
                    log_callback("Yatay sol kaydırma sınırda, işlem atlandı.")
                return False
            mouse_drag_absolute(start_x, start_y, end_x, end_y, 0.15 if F11_MODE else 0.2, steps=30 if F11_MODE else 20)  # F11 modunda daha hassas
        return True
    except Exception as e:
        if log_callback:
            log_callback(f"Sürükleme hatası: {str(e)}")
        return False

def prioritized_scroll(screen_width, screen_height, log_callback=None):
    try:
        if log_callback:
            log_callback("Öncelikli kaydırma: Dikey kontrol başlatılıyor...")
        success = scroll_with_drag('down', screen_width, screen_height, distance=300, log_callback=log_callback)
        if success:
            if log_callback:
                log_callback("Dikey kaydırma (aşağı) başarılı.")
            return True
        if log_callback:
            log_callback("Dikey kaydırma başarısız, yatay kontrol başlatılıyor...")
        success = scroll_with_drag('right', screen_width, screen_height, distance=300, log_callback=log_callback)
        if success:
            if log_callback:
                log_callback("Yatay kaydırma (sağ) başarılı.")
            return True
        if log_callback:
            log_callback("Hem dikey hem yatay kaydırma başarısız, işlem tamamlanamadı.")
        return False
    except Exception as e:
        if log_callback:
            log_callback(f"Öncelikli kaydırma hatası: {str(e)}")
        return False
# --- Parça 4 Sonu ---

# --- Parça 5 Başlangıcı ---
def move_with_win32(x, y, log_callback=None, zigzag=False):
    try:
        if x is None or y is None:
            return False
        if zigzag:
            x += random.uniform(-10, 10)
            y += random.uniform(-10, 10)
            x = max(0, min(x, EFFECTIVE_WIDTH - 1))
            y = max(0, min(y, EFFECTIVE_HEIGHT - 1))
        x_scaled = int(x * SCALE_FACTOR + random.uniform(-1, 1))
        y_scaled = int(y * SCALE_FACTOR + random.uniform(-1, 1))
        x_scaled = max(1, min(x_scaled, SCREEN_WIDTH - 2))
        y_scaled = max(1, min(y_scaled, SCREEN_HEIGHT - 2))
        if is_in_forbidden_area(x, y):
            if log_callback:
                log_callback(f"Yasak bölgeye hareket atlandı: ({x}, {y})")
            return False
        send_mouse_event(x_scaled, y_scaled, MOUSEEVENTF_MOVE)
        return True
    except Exception as e:
        if log_callback:
            log_callback(f"Fare hareket hatası: {str(e)}")
        return False

def click_with_win32(x, y, log_callback=None, zigzag=False, bot_detector=None, context=None):
    try:
        if is_in_forbidden_area(x, y, log_callback=log_callback, bot_detector=bot_detector, context=context):
            if log_callback:
                log_callback(f"Yasak bölgeye tıklama atlandı: ({x}, {y})")
            return False
        if move_with_win32(x, y, log_callback, zigzag):
            send_mouse_event(x, y, MOUSEEVENTF_LEFTDOWN)
            time.sleep(0.001)
            send_mouse_event(x, y, MOUSEEVENTF_LEFTUP)
            return True
        return False
    except Exception as e:
        if log_callback:
            log_callback(f"Tıklama hatası: {str(e)}")
        return False

def is_in_forbidden_area(x, y, log_callback=None, log_only_on_click=False, bot_detector=None, context=None):
    UPPER_FORBIDDEN_HEIGHT, LOWER_FORBIDDEN_HEIGHT, RIGHT_UPPER_QUADRANT_X, RIGHT_UPPER_QUADRANT_Y = get_forbidden_areas()
    is_popup_open = bot_detector.is_popup_open if bot_detector and hasattr(bot_detector, 'is_popup_open') else False
    
    # Sağ alt köşeyi yasakla (10x10 piksellik alan)
    FORBIDDEN_CORNER_SIZE = 10
    if (x > EFFECTIVE_WIDTH - FORBIDDEN_CORNER_SIZE and y > EFFECTIVE_HEIGHT - FORBIDDEN_CORNER_SIZE):
        if log_callback and not log_only_on_click:
            log_callback(f"Yasak bölge (sağ alt köşe): ({x}, {y})")
        return True

    if F11_MODE:
        # F11 modunda sadece dinamik oranlar kontrol ediliyor
        if y < UPPER_FORBIDDEN_HEIGHT or y > LOWER_FORBIDDEN_HEIGHT:
            if log_callback and not log_only_on_click:
                log_callback(f"F11 Modu - Yasak bölge (üst/alt sınır): ({x}, {y})")
            return True
        if context != "detect_and_click_bot_popup" and (x > RIGHT_UPPER_QUADRANT_X and y < RIGHT_UPPER_QUADRANT_Y):
            if log_callback and not log_only_on_click:
                log_callback(f"F11 Modu - Yasak bölge (sağ üst çeyrek): ({x}, {y})")
            return True
    else:
        # Normal modda üst ve alt kısımlar agresif şekilde yasak
        browser_header_height = 150  # Artırıldı, daha geniş bir üst yasak bölge
        taskbar_height = 60         # Artırıldı, daha geniş bir alt yasak bölge
        if y < browser_header_height or y > (EFFECTIVE_HEIGHT - taskbar_height):
            if log_callback and not log_only_on_click:
                log_callback(f"Yasak bölge (Tarayıcı veya görev çubuğu dışı): ({x}, {y})")
            return True
        if is_popup_open and context == "detect_and_click_bot_popup":
            if log_callback and not log_only_on_click:
                log_callback(f"Pop-up açık, diğer yasaklar atlandı: ({x}, {y})")
            return False
        if y < UPPER_FORBIDDEN_HEIGHT or y > LOWER_FORBIDDEN_HEIGHT:
            if log_callback and not log_only_on_click:
                log_callback(f"Yasak bölge (üst/alt sınır): ({x}, {y})")
            return True
        if context != "detect_and_click_bot_popup" and (x > RIGHT_UPPER_QUADRANT_X and y < RIGHT_UPPER_QUADRANT_Y):
            if log_callback and not log_only_on_click:
                log_callback(f"Yasak bölge (sağ üst çeyrek): ({x}, {y})")
            return True
    return False

def is_forbidden_button(ekran_gri, x, y, log_callback=None):
    try:
        forbidden_buttons = ['forbidden_withdraw.png', 'forbidden_grow_more.png', 'forbidden_multiplier.png']
        threshold = 0.7
        for button in forbidden_buttons:
            button_path = os.path.join(OBJECTS_DIR, button)
            if not os.path.exists(button_path):
                continue
            button_img = cv2.imread(button_path, cv2.IMREAD_GRAYSCALE)
            if button_img is None:
                continue
            h, w = button_img.shape
            region_x = max(0, int(x - w // 2))
            region_y = max(0, int(y - h // 2))
            region_w = min(w, EFFECTIVE_WIDTH - region_x)
            region_h = min(h, EFFECTIVE_HEIGHT - region_y)
            if region_w <= 0 or region_h <= 0:
                continue
            region = ekran_gri[region_y:region_y + region_h, region_x:region_x + region_w]
            if region.shape[0] < h or region.shape[1] < w:
                continue
            result = cv2.matchTemplate(region, button_img, cv2.TM_CCOEFF_NORMED)
            if cv2.minMaxLoc(result)[1] >= threshold:
                return True
        return False
    except Exception as e:
        if log_callback:
            log_callback(f"Yasak buton kontrol hatası: {str(e)}")
        return False

def bot_check_buton_bul_ve_tikla(ekran_gri, log_callback=None, bot_check_sayaci=0):
    try:
        sablon_yolu = os.path.join(OBJECTS_DIR, 'bot_check_buton.png')
        if not os.path.exists(sablon_yolu):
            return False, None, None, bot_check_sayaci
        bot_check_sablon = cv2.imread(sablon_yolu, cv2.IMREAD_GRAYSCALE)
        if bot_check_sablon is None:
            return False, None, None, bot_check_sayaci
        threshold = 0.4
        scales = [0.7, 0.8, 0.9, 1.0, 1.1, 1.2, 1.3]
        max_attempts = 3
        x_start = EFFECTIVE_WIDTH // 2
        y_start = EFFECTIVE_HEIGHT // 2
        region_gri = ekran_gri[y_start:, x_start:]
        while bot_check_sayaci < max_attempts:
            for scale in scales:
                scaled_sablon = cv2.resize(bot_check_sablon, None, fx=scale, fy=scale)
                h, w = scaled_sablon.shape
                if region_gri.shape[0] < h or region_gri.shape[1] < w:
                    continue
                result = cv2.matchTemplate(region_gri, scaled_sablon, cv2.TM_CCOEFF_NORMED)
                locations = np.where(result >= threshold)
                if len(locations[0]) > 0:
                    y, x = locations[0][0], locations[1][0]
                    center_x = x_start + x + w // 2
                    center_y = y_start + y + h // 2 + 20
                    if is_in_forbidden_area(center_x, center_y):
                        return False, None, None, bot_check_sayaci
                    if click_with_win32(center_x, center_y, log_callback):
                        bot_check_sayaci += 1
                        return True, center_x, center_y, bot_check_sayaci
            return False, None, None, bot_check_sayaci
        return True, None, None, 0
    except Exception as e:
        if log_callback:
            log_callback(f"Bot check bulma hatası: {str(e)}")
        return False, None, None, bot_check_sayaci
# --- Parça 5 Sonu ---

# --- Parça 6-1 Başlangıcı ---
class BotDetector:
    def __init__(self, log_callback=None, popup_being_handled=None):
        self.popup_counter = 0
        self.running = False
        self.stop_event = threading.Event()
        self.log_callback = log_callback
        self.popup_being_handled = popup_being_handled
        self.debug_mode = True
        self.screen_width, self.screen_height = pyautogui.size()
        self.half_width = self.screen_width // 2
        self.half_height = self.screen_height // 2
        self.browser_header_height = 150  # Normal mod için genişletilmiş üst yasak bölge
        self.taskbar_height = 60          # Normal mod için genişletilmiş alt yasak bölge
        self.safe_top = self.browser_header_height
        self.safe_bottom = self.screen_height - self.taskbar_height
        self.robot_icon_path = os.path.join(OBJECTS_DIR, 'popupobje.png')
        self.popup_path = os.path.join(OBJECTS_DIR, 'popup.png')
        self.bot_check_button_path = os.path.join(OBJECTS_DIR, 'bot_check_buton.png')
        self.bot_check_button_alt_path = os.path.join(OBJECTS_DIR, 'bot_check_buton11.png')
        self.robot_to_button_ratio = 5 if F11_MODE else 4.3  # F11 modunda 5, Normal modda 4.3
        self.scroll_speed = 8
        self.scroll_directions = ['right', 'down', 'left', 'up']
        self.scroll_method = "drag"
        self.template_threshold = 0.5
        self.button_template_threshold = 0.7
        self.robot_icon_scales = [0.6, 0.8, 1.0, 1.2, 1.5]
        self.leaderboard_exclusion_zone = (self.screen_width - 400, 0, 400, self.screen_height)
        self.normal_brightness_threshold = 120
        self.is_popup_open = False
        self._check_template_files()

    def log(self, message):
        if self.log_callback:
            self.log_callback(message)

    def _check_template_files(self):
        missing_files = []
        if not os.path.exists(self.robot_icon_path):
            missing_files.append(self.robot_icon_path)
        if not os.path.exists(self.popup_path):
            missing_files.append(self.popup_path)
        if not os.path.exists(self.bot_check_button_path):
            missing_files.append(self.bot_check_button_path)
        if not os.path.exists(self.bot_check_button_alt_path):
            missing_files.append(self.bot_check_button_alt_path)
        if missing_files:
            self.log(f"Uyarı: Şablon dosyaları eksik: {', '.join(missing_files)}")
        else:
            self.log("Robot simgesi ve buton şablonları başarıyla yüklendi.")

    def detect_darkened_state(self, screenshot):
        try:
            gray = cv2.cvtColor(screenshot, cv2.COLOR_RGB2GRAY)
            avg_brightness = np.mean(gray)
            is_dark = avg_brightness < self.normal_brightness_threshold
            self.is_popup_open = is_dark
            self.log(f"Ortalama parlaklık: {avg_brightness:.2f}, Koyu durum: {is_dark}")
            return is_dark
        except Exception as e:
            self.log(f"Koyu durum kontrol hatası: {str(e)}")
            return False

    def is_in_leaderboard(self, x, y):
        lx, ly, lw, lh = self.leaderboard_exclusion_zone
        return lx <= x <= lx + lw and ly <= y <= ly + lh

    def save_debug_image(self, image, filename, boxes=None):
        if not self.debug_mode:
            return
        try:
            pil_image = Image.fromarray(cv2.cvtColor(image, cv2.COLOR_BGR2RGB) if len(image.shape) == 3 else image)
            draw = ImageDraw.Draw(pil_image)
            if boxes:
                for box in boxes:
                    x, y, w, h, label = box
                    draw.rectangle([(x, y), (x+w, y+h)], outline="red", width=2)
                    draw.text((x, y-15), label, fill="red")
            debug_path = "debug_images"
            os.makedirs(debug_path, exist_ok=True)
            timestamp = time.strftime("%Y%m%d-%H%M%S")
            pil_image.save(os.path.join(debug_path, f"{filename}_{timestamp}.png"))
            self.log(f"Debug görüntüsü kaydedildi: {filename}_{timestamp}.png")
        except Exception as e:
            self.log(f"Debug görüntüsü kaydedilirken hata: {str(e)}")

    def take_screenshot(self):
        try:
            if self._check_emergency_stop():
                return None
            screenshot = pyautogui.screenshot()
            return np.array(screenshot)
        except Exception as e:
            self.log(f"Ekran görüntüsü alınırken hata: {str(e)}")
            return None

    def get_bottom_half_screen(self, screenshot):
        try:
            if screenshot is None:
                return None
            bottom_half = screenshot[max(self.safe_top, self.half_height):self.safe_bottom, :]
            return bottom_half
        except Exception as e:
            self.log(f"Ekranın alt yarısı alınırken hata: {str(e)}")
            return None

    def detect_robot_icon(self, image, context=None):
        try:
            if self._check_emergency_stop() or image is None:
                return False, None, None, None

            button_found, button_x, button_y = self.detect_bot_check_button(image)
            if button_found:
                self.log(f"'No, I'm not a bot' butonu bulundu: ({button_x}, {button_y})")
                return True, button_x, button_y, None

            if not os.path.exists(self.robot_icon_path):
                self.log("Robot simgesi şablonu bulunamadı.")
                return False, None, None, None

            if len(image.shape) == 3:
                gray = cv2.cvtColor(image, cv2.COLOR_RGB2GRAY)
            else:
                gray = image.copy()
            template = cv2.imread(self.robot_icon_path, cv2.IMREAD_GRAYSCALE)
            if template is None:
                self.log("Robot şablonu yüklenemedi.")
                return False, None, None, None

            best_match = None
            best_val = 0
            for scale in self.robot_icon_scales:
                if self._check_emergency_stop():
                    return False, None, None, None
                scaled_w = int(template.shape[1] * scale)
                scaled_h = int(template.shape[0] * scale)
                if scaled_w > 0 and scaled_h > 0 and scaled_w < gray.shape[1] and scaled_h < gray.shape[0]:
                    resized_template = cv2.resize(template, (scaled_w, scaled_h))
                    try:
                        result = cv2.matchTemplate(gray, resized_template, cv2.TM_CCOEFF_NORMED)
                        min_val, max_val, min_loc, max_loc = cv2.minMaxLoc(result)
                        if max_val > best_val and max_val > self.template_threshold:
                            best_val = max_val
                            x, y = max_loc
                            icon_center_x = x + scaled_w // 2
                            icon_center_y = y + scaled_h // 2
                            if context == "detect_and_click_bot_popup" and self.is_in_leaderboard(icon_center_x, icon_center_y):
                                self.log(f"Robot simgesi leaderboard bölgesinde algılandı ({icon_center_x}, {icon_center_y}), pop-up kontekstinde atlanıyor.")
                                continue
                            best_match = (icon_center_x, icon_center_y, scaled_h, max_val, scale)
                    except Exception as e:
                        self.log(f"Şablon eşleştirme hatası (ölçek {scale}): {str(e)}")
                        continue

            if best_match:
                icon_center_x, icon_center_y, icon_height, match_val, match_scale = best_match
                self.log(f"ROBOT OBJESİ TESPİTİ: Robot simgesi bulundu: ({icon_center_x},{icon_center_y}), "
                         f"Yükseklik: {icon_height}, Ölçek: {match_scale}, Eşleşme: {match_val:.2f}")
                if self.debug_mode:
                    scaled_w = int(template.shape[1] * match_scale)
                    scaled_h = int(template.shape[0] * match_scale)
                    debug_boxes = [(icon_center_x - scaled_w // 2, icon_center_y - scaled_h // 2, scaled_w, scaled_h, f"Robot ({match_val:.2f})")]
                    self.save_debug_image(image, "robot_tespiti", debug_boxes)
                return True, icon_center_x, icon_center_y, icon_height
            self.log("Robot simgesi bulunamadı")
            return False, None, None, None
        except Exception as e:
            self.log(f"Robot simgesi arama hatası: {str(e)}")
            return False, None, None, None

    def detect_bot_check_button(self, image):
        try:
            if len(image.shape) == 3:
                gray = cv2.cvtColor(image, cv2.COLOR_RGB2GRAY)
            else:
                gray = image.copy()

            template = cv2.imread(self.bot_check_button_path, cv2.IMREAD_GRAYSCALE)
            if template is None:
                self.log("Bot check buton şablonu yüklenemedi.")
                return False, None, None

            best_match = None
            best_val = 0
            for scale in self.robot_icon_scales:
                scaled_w = int(template.shape[1] * scale)
                scaled_h = int(template.shape[0] * scale)
                if scaled_w > 0 and scaled_h > 0 and scaled_w < gray.shape[1] and scaled_h < gray.shape[0]:
                    resized_template = cv2.resize(template, (scaled_w, scaled_h))
                    result = cv2.matchTemplate(gray, resized_template, cv2.TM_CCOEFF_NORMED)
                    min_val, max_val, min_loc, max_loc = cv2.minMaxLoc(result)
                    if max_val > best_val and max_val > self.button_template_threshold:
                        best_val = max_val
                        x, y = max_loc
                        best_match = (x + scaled_w // 2, y + scaled_h // 2)

            if best_match:
                button_x, button_y = best_match
                self.log(f"'No, I'm not a bot' butonu bulundu: ({button_x}, {button_y}), Eşleşme: {best_val:.2f}")
                return True, button_x, button_y

            self.log("Bot check buton bulunamadı, yedek şablon (bot_check_buton11.png) aranıyor...")
            template_alt = cv2.imread(self.bot_check_button_alt_path, cv2.IMREAD_GRAYSCALE)
            if template_alt is None:
                self.log("Yedek bot check buton şablonu yüklenemedi.")
                return False, None, None

            best_match = None
            best_val = 0
            for scale in self.robot_icon_scales:
                scaled_w = int(template_alt.shape[1] * scale)
                scaled_h = int(template_alt.shape[0] * scale)
                if scaled_w > 0 and scaled_h > 0 and scaled_w < gray.shape[1] and scaled_h < gray.shape[0]:
                    resized_template = cv2.resize(template_alt, (scaled_w, scaled_h))
                    result = cv2.matchTemplate(gray, resized_template, cv2.TM_CCOEFF_NORMED)
                    min_val, max_val, min_loc, max_loc = cv2.minMaxLoc(result)
                    if max_val > best_val and max_val > self.button_template_threshold:
                        best_val = max_val
                        x, y = max_loc
                        best_match = (x + scaled_w // 2, y + scaled_h // 2)

            if best_match:
                button_x, button_y = best_match
                self.log(f"Yedek 'No, I'm' butonu bulundu: ({button_x}, {button_y}), Eşleşme: {best_val:.2f}")
                return True, button_x, button_y

            self.log("Bot check buton veya yedek buton bulunamadı.")
            return False, None, None
        except Exception as e:
            self.log(f"Bot check buton arama hatası: {str(e)}")
            return False, None, None

    def calculate_button_position(self, icon_center_x, icon_center_y, icon_height):
        try:
            button_x = icon_center_x
            button_y = icon_center_y + int(icon_height * self.robot_to_button_ratio)
            self.log(f"BUTON KONUMU HESAPLAMA: Robot simgesi merkezi: ({icon_center_x}, {icon_center_y}), "
                     f"Yükseklik: {icon_height}, Oran: {self.robot_to_button_ratio}, "
                     f"Hesaplanan buton konumu: ({button_x}, {button_y})")
            return button_x, button_y
        except Exception as e:
            self.log(f"Buton konumu hesaplama hatası: {str(e)}")
            return None, None

    def click_at_coordinates(self, x, y, origin_bottom_half=True, bypass_forbidden_check=False):
        try:
            if self._check_emergency_stop():
                return False
            if origin_bottom_half:
                global_x = x
                global_y = max(self.safe_top, self.half_height) + y
                self.log(f"global_y hesaplama: max({self.safe_top}, {self.half_height}) + {y} = {global_y}")
            else:
                global_x = x
                global_y = y
            self.log(f"Tıklama koordinatları: ({global_x}, {global_y}), bypass_forbidden_check={bypass_forbidden_check}")
            if not bypass_forbidden_check and is_in_forbidden_area(global_x, global_y, log_callback=self.log, bot_detector=self, context="detect_and_click_bot_popup"):
                self.log(f"UYARI: Y koordinatı yasak bölgede: {global_y}")
                return False
            self.log(f"'No, I'm not a bot' butonuna tıklılıyor: ({global_x}, {global_y})")
            return click_with_win32(global_x, global_y, self.log, zigzag=False, bot_detector=self, context="detect_and_click_bot_popup")
        except Exception as e:
            self.log(f"Tıklama hatası: {str(e)}")
            return False

    def _check_emergency_stop(self):
        return EMERGENCY_STOP or not self.running or self.stop_event.is_set()

    def scroll_to_bottom(self, log_callback=None):
        return scroll_with_drag('down', self.screen_width, self.screen_height, distance=300, log_callback=log_callback)

    def scroll_to_right(self, log_callback=None):
        return scroll_with_drag('right', self.screen_width, self.screen_height, distance=300, log_callback=log_callback)

    def is_popup_fully_visible(self, button_x, button_y):
        try:
            # Buton ekran sınırları içinde mi?
            if (0 <= button_x <= self.screen_width and 
                self.safe_top <= button_y <= self.safe_bottom):
                self.log("Buton ekran sınırları içinde, pop-up görünür kabul ediliyor.")
                return True
            
            # Pop-up’ın tahmini boyutunu dinamik hesapla
            popup_width = 300  # Görselden tahmini genişlik
            popup_height = 200  # Görselden tahmini yükseklik
            top_left_x = button_x - popup_width // 2
            top_left_y = button_y - popup_height // 2
            bottom_right_x = button_x + popup_width // 2
            bottom_right_y = button_y + popup_height // 2
            
            # Pop-up tamamen ekran sınırları içinde mi?
            if (top_left_x >= 0 and top_left_y >= self.safe_top and 
                bottom_right_x <= self.screen_width and bottom_right_y <= self.safe_bottom):
                self.log("Pop-up tamamen ekran sınırları içinde.")
                return True
            else:
                self.log("Pop-up ekran sınırları dışında, kaydırma gerekebilir.")
                return False
        except Exception as e:
            self.log(f"Pop-up görünürlük kontrol hatası: {str(e)}")
            return False
# --- Parça 6-1 Sonu ---

# --- Parça 6-2 Başlangıcı ---
    def detect_and_click_bot_popup(self):
        try:
            if self._check_emergency_stop():
                return False
            successful_clicks = 0
            max_clicks_needed = 3
            max_attempts = 50
            popup_attempts = 0
            previous_button_coords = None  # Önceki buton koordinatını sakla

            while popup_attempts < max_attempts and not self._check_emergency_stop():
                popup_attempts += 1
                self.log(f"Bot kontrolü algılama denemesi {popup_attempts}/{max_attempts}")
                screenshot = self.take_screenshot()
                if screenshot is None or self._check_emergency_stop():
                    continue

                if not self.detect_darkened_state(screenshot):
                    self.log("Sayfa koyulaşmadı, pop-up'lar bitti.")
                    if successful_clicks > 0:
                        self.log(f"Toplam {successful_clicks} pop-up kapatıldı.")
                    return True

                bottom_half = self.get_bottom_half_screen(screenshot)
                if bottom_half is None or self._check_emergency_stop():
                    continue

                # Robot simgesi veya buton kontrolü
                found, center_x, center_y, icon_height = self.detect_robot_icon(bottom_half, context="detect_and_click_bot_popup")
                button_x, button_y = None, None
                if found and not self._check_emergency_stop():
                    if icon_height is not None:  # Robot simgesi bulunduysa
                        button_x, button_y = self.calculate_button_position(center_x, center_y, icon_height)
                    else:  # Buton doğrudan bulunduysa
                        button_x, button_y = center_x, center_y

                    if button_x is not None and button_y is not None:
                        # Koordinatın değişip değişmediğini kontrol et
                        if previous_button_coords == (button_x, button_y):
                            self.log("Buton koordinatı değişmedi, pop-up sabit, tıklama yapılıyor.")
                            if self.click_at_coordinates(button_x, button_y, origin_bottom_half=True, bypass_forbidden_check=True):
                                successful_clicks += 1
                                self.popup_counter += 1
                                self.log(f"Pop-up başarıyla tıklandı, toplam başarılı tıklamalar: {successful_clicks}, popup_counter: {self.popup_counter}")
                                time.sleep(0.2)
                                continue
                        elif self.is_popup_fully_visible(button_x, button_y):
                            self.log("Pop-up tam göründü, tıklama yapılıyor.")
                            if self.click_at_coordinates(button_x, button_y, origin_bottom_half=True, bypass_forbidden_check=True):
                                successful_clicks += 1
                                self.popup_counter += 1
                                self.log(f"Pop-up başarıyla tıklandı, toplam başarılı tıklamalar: {successful_clicks}, popup_counter: {self.popup_counter}")
                                time.sleep(0.2)
                                continue
                        else:
                            self.log("Pop-up tam görünmüyor, kaydırma yapılacak.")
                            previous_button_coords = (button_x, button_y)  # Koordinatı güncelle

                # Pop-up bulunamadıysa veya tam görünmüyorsa kaydırma yap
                self.log("Pop-up görünmüyor, kaydırma mantığı başlatılıyor...")
                if self.scroll_to_bottom(self.log):
                    time.sleep(0.2)
                    screenshot = self.take_screenshot()
                    bottom_half = self.get_bottom_half_screen(screenshot)
                    if bottom_half is None:
                        continue

                    found, center_x, center_y, icon_height = self.detect_robot_icon(bottom_half, context="detect_and_click_bot_popup")
                    if found:
                        if icon_height is not None:
                            button_x, button_y = self.calculate_button_position(center_x, center_y, icon_height)
                        else:
                            button_x, button_y = center_x, center_y
                        if button_x is not None and button_y is not None:
                            if self.is_popup_fully_visible(button_x, button_y):
                                self.log("Pop-up tam göründü, tıklama yapılıyor.")
                                if self.click_at_coordinates(button_x, button_y, origin_bottom_half=True, bypass_forbidden_check=True):
                                    successful_clicks += 1
                                    self.popup_counter += 1
                                    self.log(f"Pop-up başarıyla tıklandı, toplam başarılı tıklamalar: {successful_clicks}, popup_counter: {self.popup_counter}")
                                    time.sleep(0.2)
                                    continue

                    self.log("Dikey kaydırma sonrası pop-up bulunamadı, yatay kaydırma yapılıyor...")
                    if self.scroll_to_right(self.log):
                        time.sleep(0.2)
                        screenshot = self.take_screenshot()
                        bottom_half = self.get_bottom_half_screen(screenshot)
                        if bottom_half is None:
                            continue

                        found, center_x, center_y, icon_height = self.detect_robot_icon(bottom_half, context="detect_and_click_bot_popup")
                        if found:
                            if icon_height is not None:
                                button_x, button_y = self.calculate_button_position(center_x, center_y, icon_height)
                            else:
                                button_x, button_y = center_x, center_y
                            if button_x is not None and button_y is not None:
                                if self.is_popup_fully_visible(button_x, button_y):
                                    self.log("Pop-up tam göründü, tıklama yapılıyor.")
                                    if self.click_at_coordinates(button_x, button_y, origin_bottom_half=True, bypass_forbidden_check=True):
                                        successful_clicks += 1
                                        self.popup_counter += 1
                                        self.log(f"Pop-up başarıyla tıklandı, toplam başarılı tıklamalar: {successful_clicks}, popup_counter: {self.popup_counter}")
                                        time.sleep(0.2)
                                        continue

                if self.detect_darkened_state(screenshot):
                    self.log("Sayfa koyu ama pop-up bulunamadı, tekrar deneniyor...")
                    time.sleep(0.5)

            if successful_clicks >= max_clicks_needed:
                self.log("Tüm gerekli pop-up'lar kapatıldı.")
                return True
            else:
                self.log(f"Pop-up kapatma tamamlanamadı, başarılı tıklamalar: {successful_clicks}/{max_clicks_needed}")
                return False
        except Exception as e:
            self.log(f"Bot kontrol algılama hatası: {str(e)}")
            return False

    def reset_popup_counter(self):
        self.popup_counter = 0
        self.log("Pop-up sayacı sıfırlandı")

    # BornozBot.py'den alınan stabil start metodu
    def start(self):
        if not self.running:
            self.running = True
            self.stop_event.clear()
            threading.Thread(target=self._run_loop, daemon=True).start()
            self.log("BotDetector başlatıldı")

    # BornozBot.py'den alınan stabil stop metodu
    def stop(self):
        if self.running:
            self.stop_event.set()
            self.running = False
            time.sleep(0.1)  # Kısa bir bekleme süresi
            self.log("BotDetector durduruldu")
            return True
        return False

    def _run_loop(self):
        while not self._check_emergency_stop():
            self.log("\n--- Yeni algılama döngüsü başlatılıyor ---")
            if self.popup_being_handled:
                self.popup_being_handled.set()
            popup_success = self.detect_and_click_bot_popup()
            if popup_success:
                self.log("Tüm pop-up'lar kapatıldı, diğer tıklama moduna geçiliyor")
                # self.reset_popup_counter() # BU SATIR KALDIRILDI
                if self.popup_being_handled:
                    self.popup_being_handled.clear()
                time.sleep(0.1)
            else:
                self.log("Pop-up kapatma tamamlanamadı, döngü devam ediyor")
                if self.popup_being_handled:
                    self.popup_being_handled.clear()
                time.sleep(0.1)
# --- Parça 6-2 Sonu ---


# --- Parça 7 Başlangıcı ---
def priority_obje_bul_ve_tikla(ekran, ekran_gri, log_callback=None, ignore_obje_2_until=None, son_eslesme_sayaci=None, check_counter=None, kontrol_paneli=None):
    try:
        if check_counter is not None and check_counter % 5 != 0:
            return False, None, None, ignore_obje_2_until
        priority_objeler = ['priority_obje_1.png', 'priority_obje_2.png']
        threshold = 0.6
        if son_eslesme_sayaci is None:
            son_eslesme_sayaci = {}
        for obje_adi in priority_objeler:
            obje_yolu = os.path.join(OBJECTS_DIR, obje_adi)
            if obje_adi not in son_eslesme_sayaci:
                son_eslesme_sayaci[obje_adi] = 0
            if not os.path.exists(obje_yolu):
                continue
            obje = cv2.imread(obje_yolu, cv2.IMREAD_GRAYSCALE)
            if obje is None:
                continue
            if obje_adi == 'priority_obje_2.png' and ignore_obje_2_until and time.time() < ignore_obje_2_until:
                continue
            result = cv2.matchTemplate(ekran_gri, obje, cv2.TM_CCOEFF_NORMED)
            min_val, max_val, min_loc, max_loc = cv2.minMaxLoc(result)
            if max_val >= threshold:
                if son_eslesme_sayaci[obje_adi] >= 3:
                    continue
                h, w = obje.shape
                center_x = max_loc[0] + w // 2
                center_y = max_loc[1] + h // 2
                if obje_adi == 'priority_obje_2.png':
                    region = ekran[center_y - h // 2:center_y + h // 2, center_x - w // 2:center_x + w // 2]
                    if region.size == 0:
                        continue
                    mean_color = np.mean(region, axis=(0, 1))
                    if mean_color[2] < 150 and mean_color[1] < 150 and mean_color[0] < 150:
                        continue
                if is_in_forbidden_area(center_x, center_y) or is_forbidden_button(ekran_gri, center_x, center_y):
                    continue
                son_eslesme_sayaci[obje_adi] += 1
                if obje_adi == 'priority_obje_2.png':
                    ignore_obje_2_until = time.time() + 30
                if kontrol_paneli:
                    if obje_adi == 'priority_obje_1.png':
                        kontrol_paneli.priority_obje_1_clicks += 1
                    elif obje_adi == 'priority_obje_2.png':
                        kontrol_paneli.priority_obje_2_clicks += 1
                return True, center_x, center_y, ignore_obje_2_until
        return False, None, None, ignore_obje_2_until
    except Exception as e:
        if log_callback:
            log_callback(f"Öncelikli obje bulma hatası: {str(e)}")
        return False, None, None, ignore_obje_2_until

def script_obje_bul_ve_tikla(ekran_gri, script_objeler, target_x, target_y, log_callback=None, check_counter=None):
    try:
        if check_counter is not None and check_counter % 5 != 0:
            return False, None, None
        threshold = 0.6
        scale_factor = 0.5
        ekran_gri_small = cv2.resize(ekran_gri, None, fx=scale_factor, fy=scale_factor)
        for obje_adi in script_objeler:
            obje_path = os.path.join(OBJECTS_DIR, obje_adi)
            if not os.path.exists(obje_path):
                continue
            obje = cv2.imread(obje_path, cv2.IMREAD_GRAYSCALE)
            if obje is None:
                continue
            obje_small = cv2.resize(obje, None, fx=scale_factor, fy=scale_factor)
            result = cv2.matchTemplate(ekran_gri_small, obje_small, cv2.TM_CCOEFF_NORMED)
            min_val, max_val, min_loc, max_loc = cv2.minMaxLoc(result)
            if max_val >= threshold:
                h, w = obje_small.shape
                center_x = int((max_loc[0] + w // 2) / scale_factor)
                center_y = int((max_loc[1] + h // 2) / scale_factor)
                if is_in_forbidden_area(center_x, center_y) or is_forbidden_button(ekran_gri, center_x, center_y):
                    continue
                return True, center_x, center_y
        return False, None, None
    except Exception as e:
        if log_callback:
            log_callback(f"Script obje bulma hatası: {str(e)}")
        return False, None, None

def script_kaydet(script_yolu, click_positions):
    try:
        with open(script_yolu, 'w', newline='') as f:
            writer = csv.writer(f)
            writer.writerow(['x', 'y'])
            for pos in click_positions:
                writer.writerow([pos[0], pos[1]])
    except Exception as e:
        logging.error(f"Script kaydetme hatası: {str(e)}")
        raise

def simule_et_yapay_zeka(durdurma_olayi, click_positions, script_objeler, log_callback=None, popup_being_handled=None, bot_detector=None, kontrol_paneli=None):
    try:
        bot_check_sayaci = 0
        ignore_obje_2_until = None
        son_eslesme_sayaci = {}
        click_count = 0
        check_counter = 0
        if not click_positions:
            return
        while not durdurma_olayi.is_set():
            if EMERGENCY_STOP:
                log_callback("Acil durdurma algılandı, simülasyon sonlandırılıyor.")
                break
            if popup_being_handled and popup_being_handled.is_set():
                time.sleep(0.1)
                continue
            start_time = time.time()
            ekran = pyautogui.screenshot()
            ekran_np = np.array(ekran)
            ekran_rgb = cv2.cvtColor(ekran_np, cv2.COLOR_RGB2BGR)
            ekran_gri = cv2.cvtColor(ekran_np, cv2.COLOR_RGB2GRAY)
            if check_counter % 5 == 0:
                basarili, x, y, bot_check_sayaci = bot_check_buton_bul_ve_tikla(ekran_gri, log_callback, bot_check_sayaci)
                if basarili:
                    if kontrol_paneli:
                        kontrol_paneli.total_clicks += 1
                    continue
            if check_counter % 2 == 0:  # priority_obje_2.png için kontrol sıklığı artırıldı (her 3 döngüde bir)
                bulundu, x, y, ignore_obje_2_until = priority_obje_bul_ve_tikla(ekran_rgb, ekran_gri, log_callback, ignore_obje_2_until, son_eslesme_sayaci, check_counter, kontrol_paneli)
                if bulundu and x is not None and y is not None:
                    if click_with_win32(x, y, log_callback, zigzag=True, bot_detector=bot_detector, context=None):
                        click_count += 1
                        if kontrol_paneli:
                            kontrol_paneli.total_clicks += 1
                    sleep_time = random.uniform(MOUSE_SPEED_MIN / 1000, MOUSE_SPEED_MAX / 1000) + random.uniform(-0.01, 0.01)
                    time.sleep(sleep_time)
                    check_counter += 1
                    continue
            if click_positions:
                target_x, target_y = random.choice(click_positions)
                if is_in_forbidden_area(target_x, target_y, log_callback=log_callback, bot_detector=bot_detector, context=None) or is_forbidden_button(ekran_gri, target_x, target_y):
                    check_counter += 1
                    continue
                bulundu, x, y = script_obje_bul_ve_tikla(ekran_gri, script_objeler, target_x, target_y, log_callback, check_counter)
                if bulundu and x is not None and y is not None:
                    if click_with_win32(x, y, log_callback, zigzag=True, bot_detector=bot_detector, context=None):
                        click_count += 1
                        if kontrol_paneli:
                            kontrol_paneli.total_clicks += 1
                else:
                    if click_with_win32(target_x, target_y, log_callback, zigzag=True, bot_detector=bot_detector, context=None):
                        click_count += 1
                        if kontrol_paneli:
                            kontrol_paneli.total_clicks += 1
                sleep_time = random.uniform(MOUSE_SPEED_MIN / 1000, MOUSE_SPEED_MAX / 1000) + random.uniform(-0.01, 0.01)
                elapsed_time = time.time() - start_time
                time.sleep(max(0, sleep_time - elapsed_time))
            check_counter += 1
    except Exception as e:
        if log_callback:
            log_callback(f"Yapay zeka simülasyon hatası: {str(e)}")
# --- Parça 7 Sonu ---

# --- Parça 8-1 Başlangıcı ---
class KontrolPaneli:
    def __init__(self):
        self.root = tk.Tk()
        self.root.title("Bornoz Bot & Turbo Tap")
        self.root.geometry("600x500")
        self.root.configure(bg='#1E1E1E')  # Koyu gri arka plan
        self.calisiyor = False
        self.click_positions = []
        self.script_objeler = []
        self.durdurma_olayi = threading.Event()
        self.simulasyon_thread = None
        self.kayit_thread = None
        self.refresh_thread = None
        self.popup_being_handled = threading.Event()
        self.detector = BotDetector(self.log, self.popup_being_handled)
        self.mode = "Normal"
        self.total_clicks = 0
        self.priority_obje_1_clicks = 0
        self.priority_obje_2_clicks = 0
        self.priority_obje_2_coords = {False: None, True: None}  # Normal ve F11 modları için
        self.stats_window = None
        self.popup_active = False  # Pop-up testi için ek değişken
        self.refresh_interval = 300  # Varsayılan yenileme süresi
        self.style = ttk.Style()
        # Modern tema için stil ayarları
        self.style.theme_use('clam')
        self.style.configure("TButton", font=("Segoe UI", 10, "bold"), padding=10, background="#4CAF50", foreground="white")
        self.style.map("TButton", background=[("active", "#45A049"), ("disabled", "#D32F2F")])
        self.style.configure("Horizontal.TScale", troughcolor="#FF9800", background="#1E1E1E", sliderrelief="flat")
        self.style.configure("TCombobox", fieldbackground="#2E2E2E", background="#2E2E2E", foreground="white")
        self.style.configure("TEntry", fieldbackground="#2E2E2E", foreground="white")
        self._create_widgets()
        self.listener = pynput_keyboard.Listener(on_press=self.on_key_press)
        self.listener.start()
        self.root.protocol("WM_DELETE_WINDOW", self.on_closing)

    def _create_widgets(self):
        # Ana çerçeve için padding ve arka plan
        main_frame = tk.Frame(self.root, bg='#1E1E1E', padx=10, pady=10)
        main_frame.pack(fill=tk.BOTH, expand=True)

        # Üst çerçeve (butonlar ve kontroller)
        top_frame = tk.Frame(main_frame, bg='#1E1E1E', relief=tk.RAISED, borderwidth=2, highlightbackground="#4CAF50", highlightthickness=1)
        top_frame.pack(pady=5, fill=tk.X)

        # İlk satır: Temel butonlar
        self.start_btn = ttk.Button(top_frame, text="Başlat (F6)", command=self.toggle_all, style="TButton")
        self.start_btn.grid(row=0, column=0, padx=5, pady=5, sticky="ew")

        self.record_btn = ttk.Button(top_frame, text="Kayıt Başlat", command=self.kaydet_baslat, style="TButton")
        self.record_btn.grid(row=0, column=1, padx=5, pady=5, sticky="ew")

        self.stop_record_btn = ttk.Button(top_frame, text="Kayıt Durdur (F7)", command=self.kaydet_durdur, style="TButton")
        self.stop_record_btn.grid(row=0, column=2, padx=5, pady=5, sticky="ew")

        self.test_popup_btn = ttk.Button(top_frame, text="Pop-up Test (ESC)", command=self.test_popup, style="TButton")
        self.test_popup_btn.grid(row=0, column=3, padx=5, pady=5, sticky="ew")

        self.mode_btn = ttk.Button(top_frame, text="Normal Mod", command=self.toggle_mode, style="TButton")
        self.mode_btn.grid(row=0, column=4, padx=5, pady=5, sticky="ew")

        # İkinci satır: Script seçimi ve sistem bilgisi
        second_frame = tk.Frame(main_frame, bg='#1E1E1E', relief=tk.RAISED, borderwidth=2, highlightbackground="#4CAF50", highlightthickness=1)
        second_frame.pack(pady=5, fill=tk.X)

        tk.Label(second_frame, text="Script:", bg='#1E1E1E', fg='white', font=("Segoe UI", 10)).grid(row=0, column=0, padx=5, pady=5)
        self.script_combobox = ttk.Combobox(second_frame, values=tuple(os.listdir(SCRIPT_BASE_DIR)), style="TCombobox")
        self.script_combobox.grid(row=0, column=1, padx=5, pady=5)
        ttk.Button(second_frame, text="Yükle", command=self.script_yukle, style="TButton").grid(row=0, column=2, padx=5, pady=5)

        tk.Label(second_frame, text="Script Adı:", bg='#1E1E1E', fg='white', font=("Segoe UI", 10)).grid(row=0, column=3, padx=5, pady=5)
        self.script_name_entry = ttk.Entry(second_frame, width=15, style="TEntry")
        self.script_name_entry.grid(row=0, column=4, padx=5, pady=5)
        ttk.Button(second_frame, text="Kaydet", command=self.save_script_with_name, style="TButton").grid(row=0, column=5, padx=5, pady=5)

        # Üçüncü satır: Yenileme süresi ve sistem bilgisi
        third_frame = tk.Frame(main_frame, bg='#1E1E1E', relief=tk.RAISED, borderwidth=2, highlightbackground="#4CAF50", highlightthickness=1)
        third_frame.pack(pady=5, fill=tk.X)

        tk.Label(third_frame, text="Yenileme (dk:sn):", bg='#1E1E1E', fg='white', font=("Segoe UI", 10)).grid(row=0, column=0, padx=5, pady=5)
        self.refresh_min = ttk.Entry(third_frame, width=5, style="TEntry")
        self.refresh_min.grid(row=0, column=1, padx=5, pady=5)
        self.refresh_sec = ttk.Entry(third_frame, width=5, style="TEntry")
        self.refresh_sec.grid(row=0, column=2, padx=5, pady=5)
        ttk.Button(third_frame, text="Kaydet", command=self.save_refresh_time, style="TButton").grid(row=0, column=3, padx=5, pady=5)

        ttk.Button(third_frame, text="Sistem Bilgisi", command=self.sistem_bilgisi_goster, style="TButton").grid(row=0, column=4, padx=5, pady=5)
        ttk.Button(third_frame, text="Tıklama İstatistikleri", command=self.show_stats, style="TButton").grid(row=0, column=5, padx=5, pady=5)

        # Dördüncü satır: Hız ayarı
        fourth_frame = tk.Frame(main_frame, bg='#1E1E1E', relief=tk.RAISED, borderwidth=2, highlightbackground="#4CAF50", highlightthickness=1)
        fourth_frame.pack(pady=5, fill=tk.X)

        tk.Label(fourth_frame, text="Min Hız (ms):", bg='#1E1E1E', fg='white', font=("Segoe UI", 10)).grid(row=0, column=0, padx=5, pady=5)
        self.speed_min_entry = ttk.Entry(fourth_frame, width=5, style="TEntry")
        self.speed_min_entry.insert(0, str(MOUSE_SPEED_MIN))
        self.speed_min_entry.grid(row=0, column=1, padx=5, pady=5)

        tk.Label(fourth_frame, text="Max Hız (ms):", bg='#1E1E1E', fg='white', font=("Segoe UI", 10)).grid(row=0, column=2, padx=5, pady=5)
        self.speed_max_entry = ttk.Entry(fourth_frame, width=5, style="TEntry")
        self.speed_max_entry.insert(0, str(MOUSE_SPEED_MAX))
        self.speed_max_entry.grid(row=0, column=3, padx=5, pady=5)

        ttk.Button(fourth_frame, text="Hız Kaydet", command=self.update_speed, style="TButton").grid(row=0, column=4, padx=5, pady=5)

        # Beşinci satır: Pop-up tıklama mesafesi
        fifth_frame = tk.Frame(main_frame, bg='#1E1E1E', relief=tk.RAISED, borderwidth=2, highlightbackground="#4CAF50", highlightthickness=1)
        fifth_frame.pack(pady=5, fill=tk.X)

        tk.Label(fifth_frame, text="Pop-up Tıklama Mesafesi (px):", bg='#1E1E1E', fg='white', font=("Segoe UI", 10)).grid(row=0, column=0, padx=5, pady=5)
        self.popup_click_offset_scale = ttk.Scale(fifth_frame, from_=1, to=20, orient=tk.HORIZONTAL, length=150, style="Horizontal.TScale")
        self.popup_click_offset_scale.set(self.detector.robot_to_button_ratio)
        self.popup_click_offset_scale.config(command=self.update_popup_click_offset)
        self.popup_click_offset_scale.grid(row=0, column=1, columnspan=2, padx=5, pady=5)
        self.popup_click_offset_label = tk.Label(fifth_frame, text=str(self.detector.robot_to_button_ratio), bg='#1E1E1E', fg='white', font=("Segoe UI", 10))
        self.popup_click_offset_label.grid(row=0, column=3, padx=5, pady=5)

        # Durum etiketi
        self.durum_etiket = tk.Label(main_frame, text="Durum: Durduruldu", bg='#1E1E1E', fg='#4CAF50', font=("Segoe UI", 12, "bold"), relief=tk.SUNKEN, borderwidth=2)
        self.durum_etiket.pack(pady=10, fill=tk.X)

        # Log ekranı
        self.log_text = scrolledtext.ScrolledText(main_frame, width=70, height=20, bg='#2E2E2E', fg='white', font=("Segoe UI", 9), insertbackground="white", relief=tk.SUNKEN, borderwidth=2)
        self.log_text.pack(pady=10, fill=tk.BOTH, expand=True)

    def toggle_mode(self):
        global F11_MODE, UPPER_FORBIDDEN_HEIGHT, LOWER_FORBIDDEN_HEIGHT, RIGHT_UPPER_QUADRANT_X, RIGHT_UPPER_QUADRANT_Y
        start_time = time.time()
        if self.mode == "Normal":
            self.mode = "F11"
            F11_MODE = True
            self.detector.robot_to_button_ratio = 5
            self.mode_btn.config(text="F11 Modu")
            self.log("F11 Modu etkinleştirildi")
        else:
            self.mode = "Normal"
            F11_MODE = False
            self.detector.robot_to_button_ratio = 4.3
            self.mode_btn.config(text="Normal Mod")
            self.log("Normal Mod etkinleştirildi")
        # Yasak bölgeleri anında güncelle
        UPPER_FORBIDDEN_HEIGHT, LOWER_FORBIDDEN_HEIGHT, RIGHT_UPPER_QUADRANT_X, RIGHT_UPPER_QUADRANT_Y = get_forbidden_areas()
        self.root.update_idletasks()  # GUI güncellemelerini anında uygula
        self.log(f"Mod geçişi tamamlandı, süre: {time.time() - start_time:.3f} saniye")

    def log(self, message):
        if hasattr(self, 'log_text'):
            self.log_text.insert(tk.END, f"[{time.strftime('%H:%M:%S')}] {message}\n")
            self.log_text.see(tk.END)
        else:
            print(f"Log Hatası: {message}")
# --- Parça 8-1 Sonu ---

# --- Parça 8-2 Başlangıcı ---
    def update_speed(self):
        global MOUSE_SPEED_MIN, MOUSE_SPEED_MAX
        try:
            min_speed = int(self.speed_min_entry.get())
            max_speed = int(self.speed_max_entry.get())
            if min_speed < MOUSE_SPEED_MIN_LIMIT or max_speed > MOUSE_SPEED_MAX_LIMIT:
                raise ValueError(f"Hız aralığı {MOUSE_SPEED_MIN_LIMIT} ms ile {MOUSE_SPEED_MAX_LIMIT} ms arasında olmalıdır")
            if min_speed > max_speed:
                raise ValueError("Minimum hız, maksimum hızdan büyük olamaz")
            MOUSE_SPEED_MIN = min_speed
            MOUSE_SPEED_MAX = max_speed
            self.log(f"Tıklama hızı güncellendi: Min {MOUSE_SPEED_MIN} ms, Max {MOUSE_SPEED_MAX} ms")
        except ValueError as e:
            self.log(f"Hata: {str(e)}")

    def update_popup_click_offset(self, value):
        self.detector.robot_to_button_ratio = float(value)
        if hasattr(self, 'popup_click_offset_label'):
            self.popup_click_offset_label.config(text=f"{self.detector.robot_to_button_ratio:.1f}")
            self.log(f"Pop-up tıklama mesafesi güncellendi: {self.detector.robot_to_button_ratio:.1f} px")
        else:
            self.log("Hata: popup_click_offset_label tanımlı değil!")

    def save_refresh_time(self):
        try:
            mins = int(self.refresh_min.get() or 0)
            secs = int(self.refresh_sec.get() or 0)
            if mins < 0 or secs < 0:
                raise ValueError("Negatif değerler girilemez")
            self.refresh_interval = mins * 60 + secs
            self.log(f"Yenileme süresi kaydedildi: {mins} dk {secs} sn")
            if self.calisiyor and self.refresh_interval > 0:
                self.start_refresh()
        except ValueError as e:
            self.log(f"Hata: {str(e)}")

    def on_key_press(self, key):
        try:
            if key == pynput_keyboard.Key.f6 and not self.durdurma_olayi.is_set():  # F6 tetiklenmesini tek seferle sınırla
                self.toggle_all()
            elif key == pynput_keyboard.Key.f7:  # Kayıt durdurma için F7
                if hasattr(self, 'kayit_durdurma_olayi') and not self.kayit_durdurma_olayi.is_set():
                    self.kaydet_durdur()
            elif key == pynput_keyboard.Key.esc:
                global EMERGENCY_STOP
                EMERGENCY_STOP = True
                if self.calisiyor:
                    self.toggle_all()
                if self.popup_active:
                    self.test_popup()
        except Exception as e:
            self.log(f"Klavye hatası: {str(e)}")

    def toggle_all(self):
        global EMERGENCY_STOP
        if self.calisiyor:
            self.log("Durdurma işlemi başlatılıyor...")
            self.durdurma_olayi.set()
            if hasattr(self.detector, 'stop'):  # stop metodu varsa çağır
                self.detector.stop()
            if self.simulasyon_thread:
                self.simulasyon_thread.join(timeout=3.0)
                if self.simulasyon_thread.is_alive():
                    self.log("Simülasyon thread'i tam kapanmadı, zorla sonlandırılıyor!")
            if self.refresh_thread:
                self.refresh_thread.join(timeout=3.0)
                if self.refresh_thread.is_alive():
                    self.log("Yenileme thread'i tam kapanmadı, zorla sonlandırılıyor!")
            self.calisiyor = False
            EMERGENCY_STOP = False
            self.durum_etiket.config(text="Durum: Durduruldu")
            self.start_btn.config(text="Başlat (F6)", state=tk.NORMAL)
            self.record_btn.config(state=tk.NORMAL)
            self.test_popup_btn.config(state=tk.NORMAL)
            self.mode_btn.config(state=tk.NORMAL)
            self.log("Tüm işlemler durduruldu (F6)")
        else:
            if not self.click_positions:
                messagebox.showwarning("Uyarı", "Lütfen bir script yükleyin!")
                return
            self.log("Başlatma işlemi başlatılıyor...")
            self.calisiyor = True
            self.durdurma_olayi.clear()
            self.detector.start()  # start metodu zaten mevcut
            self.simulasyon_thread = threading.Thread(target=simule_et_yapay_zeka, args=(self.durdurma_olayi, self.click_positions, self.script_objeler, self.log, self.popup_being_handled, self.detector, self), daemon=True)
            self.simulasyon_thread.start()
            if self.refresh_interval > 0:
                self.start_refresh()
            self.durum_etiket.config(text="Durum: Çalışıyor")
            self.start_btn.config(text="Durdur (F6)", state=tk.NORMAL)
            self.record_btn.config(state=tk.DISABLED)
            self.test_popup_btn.config(state=tk.DISABLED)
            self.mode_btn.config(state=tk.DISABLED)
            self.log("Tüm işlemler başlatıldı (F6)")

    def start_refresh(self):
        if self.refresh_interval > 0 and not self.refresh_thread:
            self.refresh_thread = threading.Thread(target=self.refresh_page, daemon=True)
            self.refresh_thread.start()
            self.log(f"Yenileme başlatıldı: Her {self.refresh_interval} saniyede bir")
        else:
            self.log("Hata: Yenileme süresi kaydedilmedi veya thread zaten aktif")

    def refresh_page(self):
        while not self.durdurma_olayi.is_set():
            send_key_event(0x74, True)  # F5 bas
            time.sleep(0.01)
            send_key_event(0x74, False)  # F5 bırak
            self.log("Sayfa yenilendi (F5)")
            time.sleep(self.refresh_interval)  # Her döngüde güncel refresh_interval kullan

    def test_popup(self):
        if self.popup_active:
            self.popup_active = False
            self.log("Pop-up testi durduruldu (ESC)")
        else:
            self.popup_active = True
            self.detector.detect_and_click_bot_popup()
            self.log("Pop-up testi başlatıldı")

    def script_yukle(self):
        self.click_positions = []
        dosya_adi = self.script_combobox.get()
        if not dosya_adi:
            self.log("HATA: Script seçilmedi!")
            return
        try:
            tam_yol = os.path.join(SCRIPT_BASE_DIR, dosya_adi)
            if not os.path.exists(tam_yol):
                self.log(f"HATA: Script dosyası bulunamadı: {tam_yol}")
                return
            with open(tam_yol, 'r', encoding='utf-8') as f:
                reader = csv.reader(f)
                header = next(reader, None)  # Header yoksa hata vermemesi için None kontrolü
                if header and header != ['x', 'y']:
                    self.log(f"HATA: Geçersiz script formatı: {dosya_adi}")
                    return
                for row in reader:
                    if len(row) == 2 and all(r.strip() for r in row):  # Boş satırları ve geçersiz verileri kontrol et
                        try:
                            x, y = map(float, row)
                            self.click_positions.append((int(x), int(y)))
                        except ValueError:
                            self.log(f"HATA: Geçersiz koordinat formatı: {row}")
                            continue
            self.log(f"Script yüklendi: {dosya_adi}, {len(self.click_positions)} tıklama noktası")
        except Exception as e:
            self.log(f"Script yükleme hatası: {str(e)}")

    def kaydet_baslat(self):
        self.click_positions = []
        self.log("Kayıt başlatıldı. Fare tıklamalarınız 20 saniye boyunca kaydedilecek (F7 ile durdurabilirsiniz)...")
        self.kayit_durdurma_olayi = threading.Event()
        self.kayit_thread = threading.Thread(target=self.kayit_yap, daemon=True)
        self.kayit_thread.start()
        threading.Timer(20.0, self.kaydet_durdur).start()

    def kayit_yap(self):
        def on_click(x, y, button, pressed):
            if self.kayit_durdurma_olayi.is_set():
                return False
            if button == pynput_mouse.Button.left and pressed:
                x = int(x / SCALE_FACTOR)
                y = int(y / SCALE_FACTOR)
                if not is_in_forbidden_area(x, y, self.log, log_only_on_click=True):
                    self.log(f"Tıklama kaydedildi: ({x}, {y})")
                    self.click_positions.append((x, y))
                else:
                    self.log(f"Yasak bölgede tıklama atlandı: ({x}, {y})")
        with pynput_mouse.Listener(on_click=on_click) as listener:
            listener.join()

    def validate_script_name(self, name):
        invalid_chars = ['/', '\\', ':', '*', '?', '"', '<', '>', '|']
        for char in invalid_chars:
            if char in name:
                return False
        return True

    def kaydet_durdur(self):
        if not hasattr(self, 'kayit_durdurma_olayi'):
            self.log("HATA: Kayıt başlatılmamış!")
            return
        self.kayit_durdurma_olayi.set()
        if not self.click_positions:
            self.log("Kaydedilecek tıklama yok!")
            return
        self.log("Kayıt durduruldu. Script adını girin ve 'Kaydet' butonuna basın.")
        self.stop_record_btn.config(state=tk.NORMAL)  # Kayıt durdur butonunu aktif tut

    def save_script_with_name(self):
        if not self.click_positions:
            self.log("Kaydedilecek tıklama yok!")
            return
        script_adi = self.script_name_entry.get().strip()
        if not script_adi:
            script_adi = f"script_{time.strftime('%Y%m%d_%H%M%S')}"
        if not script_adi.endswith('.csv'):
            script_adi += '.csv'
        if not self.validate_script_name(script_adi):
            self.log("HATA: Script adı geçersiz karakterler içeriyor!")
            return
        try:
            script_yolu = os.path.join(SCRIPT_BASE_DIR, script_adi)
            script_kaydet(script_yolu, self.click_positions)
            self.log(f"Script kaydedildi: {script_adi}, {len(self.click_positions)} tıklama")
            self.script_combobox['values'] = tuple(os.listdir(SCRIPT_BASE_DIR))
            self.script_combobox.set(script_adi)
            self.stop_record_btn.config(state=tk.NORMAL)  # Kayıt durdur butonunu tekrar aktif et
        except Exception as e:
            self.log(f"Kaydetme hatası: {str(e)}")

    def sistem_bilgisi_goster(self):
        try:
            platform_info = platform.platform()
            cpu_count = psutil.cpu_count()
            cpu_usage = psutil.cpu_percent(interval=0.5)
            memory = psutil.virtual_memory()
            memory_total = memory.total / (1024 ** 3)
            memory_used = memory.used / (1024 ** 3)
            memory_percent = memory.percent
            disk = psutil.disk_usage('/')
            disk_total = disk.total / (1024 ** 3)
            disk_used = disk.used / (1024 ** 3)
            disk_percent = disk.percent
            net_io = psutil.net_io_counters()
            net_sent_mb = net_io.bytes_sent / (1024 ** 2)
            net_recv_mb = net_io.bytes_recv / (1024 ** 2)
            scaling = SCALE_FACTOR
            monitor_info = f"{SCREEN_WIDTH}x{SCREEN_HEIGHT}"
            effective_res = f"{EFFECTIVE_WIDTH}x{EFFECTIVE_HEIGHT}"
            sistem_bilgisi = f"""
            Sistem bilgisi:
            --------------------------- 
            Sistem: {platform_info}
            CPU çekirdek sayısı: {cpu_count}
            CPU kullanımı: {cpu_usage}%
            Bellek:
            Toplam: {memory_total:.2f} GB
            Kullanılan: {memory_used:.2f} GB ({memory_percent}%)
            Disk:
            Toplam: {disk_total:.2f} GB
            Kullanılan: {disk_used:.2f} GB ({disk_percent}%)
            Ağ:
            Gönderilen: {net_sent_mb:.2f} MB
            Alınan: {net_recv_mb:.2f} MB
            Ekran:
            Ölçeklendirme faktörü: {scaling:.2f}
            Sistem çözünürlüğü: {monitor_info}
            Etkin çözünürlük: {effective_res}
            """
            messagebox.showinfo("Sistem Bilgisi", sistem_bilgisi)
        except Exception as e:
            self.log(f"Sistem bilgisi alınırken hata: {str(e)}")

    def show_stats(self):
        if self.stats_window and self.stats_window.winfo_exists():
            self.stats_window.lift()
            return

        self.stats_window = tk.Toplevel(self.root)
        self.stats_window.title("Tıklama İstatistikleri")
        self.stats_window.geometry("400x500")
        self.stats_window.configure(bg='#1E1E1E')

        tk.Label(self.stats_window, text="Tıklama İstatistikleri", bg='#1E1E1E', fg='white', font=("Segoe UI", 14, "bold")).pack(pady=10)

        self.log(f"Stats gösteriliyor - Popup Counter: {self.detector.popup_counter}")  # Log eklendi
        tk.Label(self.stats_window, text=f"Toplam Tıklama: {self.total_clicks}", bg='#1E1E1E', fg='#4CAF50', font=("Segoe UI", 10)).pack(pady=5)
        tk.Label(self.stats_window, text=f"Pop-up Kapatma: {self.detector.popup_counter}", bg='#1E1E1E', fg='#4CAF50', font=("Segoe UI", 10)).pack(pady=5)
        tk.Label(self.stats_window, text=f"Süt Şişesi Tıklama: {self.priority_obje_1_clicks}", bg='#1E1E1E', fg='#4CAF50', font=("Segoe UI", 10)).pack(pady=5)
        tk.Label(self.stats_window, text=f"Manure Tıklama: {self.priority_obje_2_clicks}", bg='#1E1E1E', fg='#4CAF50', font=("Segoe UI", 10)).pack(pady=5)

        if MATPLOTLIB_AVAILABLE:
            fig, ax = plt.subplots(figsize=(5, 3))
            labels = ['Toplam Tıklama', 'Pop-up Kapatma', 'Obje 1', 'Obje 2']
            values = [self.total_clicks, self.detector.popup_counter, self.priority_obje_1_clicks, self.priority_obje_2_clicks]
            ax.bar(labels, values, color=['#2196F3', '#4CAF50', '#F44336', '#9C27B0'])
            ax.set_title("Tıklama İstatistikleri", color='white')
            ax.set_ylabel("Sayı", color='white')
            ax.tick_params(axis='x', colors='white', rotation=45)
            ax.tick_params(axis='y', colors='white')
            ax.set_facecolor('#2E2E2E')
            fig.set_facecolor('#1E1E1E')

            canvas = FigureCanvasTkAgg(fig, master=self.stats_window)
            canvas.draw()
            canvas.get_tk_widget().pack(pady=10)
        else:
            tk.Label(self.stats_window, text="Grafikler için matplotlib kütüphanesi gerekli!", bg='#1E1E1E', fg='#F44336', font=("Segoe UI", 10)).pack(pady=10)

        self.stats_window.protocol("WM_DELETE_WINDOW", self.close_stats)

    def close_stats(self):
        if self.stats_window:
            self.stats_window.destroy()
            self.stats_window = None

    def on_closing(self):
        try:
            if self.calisiyor:
                if messagebox.askyesno("Çıkış", "İşlemler hala çalışıyor. Çıkmak istediğinize emin misiniz?"):
                    self.toggle_all()
                    self.listener.stop()
                    if self.stats_window and self.stats_window.winfo_exists():
                        self.stats_window.destroy()
                    self.root.destroy()
            else:
                self.listener.stop()
                if self.stats_window and self.stats_window.winfo_exists():
                    self.stats_window.destroy()
                self.root.destroy()
        except Exception as e:
            self.log(f"Kapatma hatası: {str(e)}")
            self.root.destroy()

def main():
    os.makedirs("debug_images", exist_ok=True)
    panel = KontrolPaneli()
    panel.root.mainloop()

if __name__ == "__main__":
    main()
# --- Parça 8-2 Sonu ---
