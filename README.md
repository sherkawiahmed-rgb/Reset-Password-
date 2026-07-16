import telebot
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.common.exceptions import TimeoutException, NoSuchElementException
from selenium.webdriver.chrome.options import Options
import time
import logging
import re
import threading
import sys
import os
from datetime import datetime
import traceback

# إعداد التسجيل المتقدم
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('unified_bot.log', encoding='utf-8'),
        logging.StreamHandler(sys.stdout)
    ]
)
logger = logging.getLogger(__name__)

# إعدادات البوت
TOKEN = "8222859446:AAHmcWKOs0jYdu1wYYriEEa2KLFN_PQDimo"

class UnifiedResetBot:
    def __init__(self, token):
        self.token = token
        self.bot = None
        self.user_data = {}
        self.max_restarts = 200
        self.restart_delay = 10
        self.restart_count = 0
        self.last_restart_time = None
        
    def initialize_bot(self):
        """تهيئة البوت"""
        self.bot = telebot.TeleBot(self.token)
        self.setup_handlers()
        
    def setup_driver(self, fast_mode=False):
        """إعداد متصفح Chrome"""
        options = Options()
        
        if fast_mode:
            # وضع سريع للملفات
            options.add_argument('--headless=new')
            options.add_argument('--disable-images')
            options.add_argument('--disable-javascript')
            options.add_argument('--blink-settings=imagesEnabled=false')
            options.add_argument('--disable-extensions')
            options.add_argument('--disable-plugins')
            prefs = {
                'profile.default_content_setting_values': {
                    'images': 2,
                    'javascript': 1,
                }
            }
            options.add_experimental_option('prefs', prefs)
        else:
            # وضع عادي لاستخراج الكود
            options.add_experimental_option("excludeSwitches", ["enable-automation"])
            options.add_experimental_option('useAutomationExtension', False)
            options.add_argument('--disable-blink-features=AutomationControlled')
            options.add_argument('--headless')
        
        # إعدادات مشتركة
        options.add_argument('--no-sandbox')
        options.add_argument('--disable-dev-shm-usage')
        options.add_argument('--disable-gpu')
        options.add_argument('--window-size=1920,1080')
        options.add_argument('--user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36')
        
        driver = webdriver.Chrome(options=options)
        
        if not fast_mode:
            driver.execute_script("Object.defineProperty(navigator, 'webdriver', {get: () => undefined})")
        
        return driver

    def login_to_site(self, driver):
        """تسجيل الدخول إلى الموقع"""
        try:
            logger.info("جاري تحميل صفحة التسجيل...")
            driver.get("https://devutils.myviewcloud.com:9100/#/login?redirect=%2Fdashboard")
            time.sleep(2)
            
            logger.info("البحث عن حقول التسجيل...")
            all_inputs = driver.find_elements(By.TAG_NAME, "input")
            logger.info(f"عدد حقول الإدخال: {len(all_inputs)}")
            
            username_field = None
            password_field = None
            
            for input_field in all_inputs:
                placeholder = input_field.get_attribute("placeholder") or ""
                input_type = input_field.get_attribute("type")
                
                if "user" in placeholder.lower() or input_type == "text":
                    username_field = input_field
                    logger.info("✅ تم العثور على حقل username")
                
                elif "pass" in placeholder.lower() or input_type == "password":
                    password_field = input_field
                    logger.info("✅ تم العثور على حقل password")
            
            if not username_field and len(all_inputs) >= 1:
                username_field = all_inputs[0]
            
            if not password_field and len(all_inputs) >= 2:
                password_field = all_inputs[1]
            
            if username_field:
                username_field.clear()
                username_field.send_keys("Elsharkawy")
                logger.info("تم إدخال username")
                time.sleep(0.5)
            
            if password_field:
                password_field.clear()
                password_field.send_keys("Tiandy@123")
                logger.info("تم إدخال password")
                time.sleep(0.5)
            
            login_button = None
            buttons = driver.find_elements(By.TAG_NAME, "button")
            
            for button in buttons:
                text = button.text.strip()
                if "login" in text.lower():
                    login_button = button
                    logger.info("✅ تم العثور على زر Login")
                    break
            
            if not login_button and buttons:
                login_button = buttons[0]
            
            if login_button:
                login_button.click()
                logger.info("تم النقر على زر Login")
                time.sleep(2)
                return True
            
            return False
            
        except Exception as e:
            logger.error(f"خطأ في التسجيل: {e}")
            return False

    def extract_reset_code(self, driver, user_id, runtime):
        """استخراج كود الإعادة من الصفحة"""
        try:
            logger.info("جاري الانتقال إلى صفحة الاستعادة...")
            driver.get("https://devutils.myviewcloud.com:9100/#/recovepwd/recover")
            time.sleep(2)
            
            all_inputs = driver.find_elements(By.TAG_NAME, "input")
            logger.info(f"عدد حقول الإدخال في الاستعادة: {len(all_inputs)}")
            
            id_field = None
            runtime_field = None
            
            for input_field in all_inputs:
                placeholder = input_field.get_attribute("placeholder") or ""
                
                if "device id" in placeholder.lower() or "id" in placeholder.lower():
                    id_field = input_field
                    logger.info("✅ تم العثور على حقل Device ID")
                
                elif "runtime" in placeholder.lower():
                    runtime_field = input_field
                    logger.info("✅ تم العثور على حقل runtime")
            
            if not id_field and len(all_inputs) >= 1:
                id_field = all_inputs[0]
            
            if not runtime_field and len(all_inputs) >= 2:
                runtime_field = all_inputs[1]
            
            if id_field:
                id_field.clear()
                id_field.send_keys(user_id)
                logger.info(f"تم إدخال ID: {user_id}")
                time.sleep(0.5)
            
            if runtime_field:
                runtime_field.clear()
                runtime_field.send_keys(runtime)
                logger.info(f"تم إدخال runtime: {runtime}")
                time.sleep(0.5)
            
            generate_button = None
            buttons = driver.find_elements(By.TAG_NAME, "button")
            
            for button in buttons:
                text = button.text.strip()
                if "generate" in text.lower():
                    generate_button = button
                    logger.info("✅ تم العثور على زر Generate")
                    break
            
            if not generate_button and buttons:
                generate_button = buttons[0]
            
            if generate_button:
                generate_button.click()
                logger.info("تم النقر على زر Generate")
                time.sleep(2)
                
                page_text = driver.page_source
                cipher_pattern = r'Cipher string:\s*([a-f0-9]+)'
                match = re.search(cipher_pattern, page_text, re.IGNORECASE)
                
                if match:
                    reset_code = match.group(1)
                    logger.info(f"✅ تم العثور على الكود: {reset_code}")
                    return reset_code
                
                all_elements = driver.find_elements(By.XPATH, "//*[contains(text(), 'Cipher string')]")
                for element in all_elements:
                    text = element.text
                    if "Cipher string:" in text:
                        code_match = re.search(r'Cipher string:\s*([a-f0-9]+)', text, re.IGNORECASE)
                        if code_match:
                            reset_code = code_match.group(1)
                            logger.info(f"✅ تم العثور على الكود من العنصر: {reset_code}")
                            return reset_code
                
                logger.warning("❌ لم يتم العثور على الكود في الصفحة")
                return None
            
            return None
            
        except Exception as e:
            logger.error(f"خطأ في الحصول على الكود: {e}")
            return None

    def quick_login(self, driver):
        """تسجيل دخول سريع"""
        try:
            logger.info("🚀 جاري التسجيل السريع...")
            driver.get("https://devutils.myviewcloud.com:9100/#/login?redirect=%2Fdashboard")
            time.sleep(1)
            
            inputs = driver.find_elements(By.TAG_NAME, "input")
            
            if len(inputs) < 2:
                return False, "لم يتم العثور على حقول الإدخال"
            
            username_field = inputs[0]
            password_field = inputs[1]
            
            username_field.clear()
            username_field.send_keys("Elsharkawy")
            
            password_field.clear()
            password_field.send_keys("Tiandy@123")
            
            buttons = driver.find_elements(By.TAG_NAME, "button")
            login_button = None
            
            for button in buttons:
                if button.is_enabled():
                    login_button = button
                    break
            
            if login_button:
                driver.execute_script("arguments[0].click();", login_button)
            else:
                from selenium.webdriver.common.keys import Keys
                password_field.send_keys(Keys.RETURN)
            
            time.sleep(1)
            
            current_url = driver.current_url
            if "dashboard" in current_url or "pwd" in current_url:
                return True, "تم التسجيل بنجاح"
            else:
                return False, "فشل التسجيل"
                
        except Exception as e:
            return False, f"خطأ في التسجيل: {str(e)}"

    def fast_file_processing(self, driver, file_path):
        """معالجة سريعة للملف"""
        try:
            logger.info("🔄 معالجة سريعة للملف...")
            
            driver.get("https://devutils.myviewcloud.com:9100/#/pwd/resetpwd")
            time.sleep(1)
            
            file_inputs = driver.find_elements(By.XPATH, "//input[@type='file']")
            if not file_inputs:
                return False, "لم يتم العثور على حقل رفع الملف"
            
            file_input = file_inputs[0]
            
            # التأكد من أن الملف موجود قبل محاولة رفعه
            if not os.path.exists(file_path):
                return False, f"الملف غير موجود: {file_path}"
            
            file_input.send_keys(file_path)
            logger.info("✅ تم رفع الملف")
            
            buttons = driver.find_elements(By.TAG_NAME, "button")
            import_button = None
            
            for button in buttons:
                text = button.text.lower()
                if any(word in text for word in ['import', 'upload', 'submit']):
                    import_button = button
                    break
            
            if not import_button and buttons:
                import_button = buttons[0]
            
            if import_button:
                driver.execute_script("arguments[0].click();", import_button)
            else:
                driver.execute_script("""
                    var forms = document.querySelectorAll('form');
                    for (var form of forms) {
                        form.submit();
                    }
                """)
            
            logger.info("⏳ جاري المعالجة...")
            time.sleep(1)
            
            possible_selectors = [
                "//*[contains(@class, 'code')]",
                "//*[contains(text(), 'Security')]",
                "//*[contains(text(), 'Code')]",
                "//code",
                "//pre",
                "//i",
                "//span",
                "//div"
            ]
            
            for selector in possible_selectors:
                try:
                    elements = driver.find_elements(By.XPATH, selector)
                    for element in elements:
                        text = element.text.strip()
                        if text and len(text) >= 6 and any(c.isdigit() for c in text):
                            cleaned_text = text.replace('copy', '').replace('Copy', '').replace('COPY', '').strip()
                            logger.info(f"✅ تم العثور على الكود: {cleaned_text}")
                            return True, cleaned_text
                except:
                    continue
            
            body_text = driver.find_element(By.TAG_NAME, "body").text
            lines = [line.strip() for line in body_text.split('\n') if line.strip()]
            
            for line in lines:
                if len(line) >= 6 and any(c.isdigit() for c in line):
                    cleaned_line = line.replace('copy', '').replace('Copy', '').replace('COPY', '').strip()
                    logger.info(f"✅ تم العثور على الكود في النص: {cleaned_line}")
                    return True, cleaned_line
            
            return False, "لم يتم العثور على Security Code"
            
        except Exception as e:
            return False, f"خطأ في المعالجة: {str(e)}"

    def send_message_safely(self, chat_id, text, message_id=None, parse_mode=None):
        """إرسال رسالة آمنة بدون أي أسماء"""
        try:
            if message_id:
                self.bot.send_message(
                    chat_id, 
                    text, 
                    reply_to_message_id=message_id,
                    parse_mode=parse_mode
                )
            else:
                self.bot.send_message(
                    chat_id, 
                    text, 
                    parse_mode=parse_mode
                )
        except Exception as e:
            logger.error(f"خطأ في إرسال الرسالة: {e}")

    def edit_message_safely(self, chat_id, message_id, text, parse_mode=None):
        """تعديل رسالة بأمان"""
        try:
            self.bot.edit_message_text(
                chat_id=chat_id,
                message_id=message_id,
                text=text,
                parse_mode=parse_mode
            )
            return True
        except Exception as e:
            logger.error(f"خطأ في تعديل الرسالة: {e}")
            return False

    def process_reset_code(self, chat_id, user_id, runtime, processing_message_id, telegram_user_id):
        """معالجة استخراج كود الإعادة"""
        try:
            logger.info(f"بدء المعالجة في الخلفية للمستخدم {chat_id}")
            
            driver = self.setup_driver(fast_mode=False)
            
            try:
                login_success = self.login_to_site(driver)
                
                if not login_success:
                    self.edit_message_safely(
                        chat_id, 
                        processing_message_id,
                        "❌ فشل في التسجيل إلى الموقع"
                    )
                    return
                
                reset_code = self.extract_reset_code(driver, user_id, runtime)
                
                if reset_code:
                    # تحويل رسالة المعالجة إلى الكود
                    response = f"`{reset_code}`"
                    self.edit_message_safely(
                        chat_id, 
                        processing_message_id,
                        response,
                        'Markdown'
                    )
                else:
                    self.edit_message_safely(
                        chat_id, 
                        processing_message_id,
                        "❌ لم أتمكن من العثور على الكود"
                    )
                    
            except Exception as e:
                logger.error(f"خطأ في المعالجة: {e}")
                self.edit_message_safely(
                    chat_id, 
                    processing_message_id,
                    f"❌ حدث خطأ أثناء المعالجة"
                )
                
            finally:
                if driver:
                    driver.quit()
                if chat_id in self.user_data:
                    del self.user_data[chat_id]
                    
        except Exception as e:
            logger.error(f"خطأ عام في الخيط: {e}")
            if chat_id in self.user_data:
                del self.user_data[chat_id]

    def process_dat_file(self, file_path, chat_id, processing_message_id):
        """معالجة ملف DAT"""
        driver = None
        
        try:
            logger.info("⚡ بدء المعالجة السريعة...")
            
            # التأكد من أن الملف موجود قبل البدء
            if not os.path.exists(file_path):
                self.edit_message_safely(chat_id, processing_message_id, "❌ الملف المؤقت غير موجود")
                return
            
            driver = self.setup_driver(fast_mode=True)
            if not driver:
                self.edit_message_safely(chat_id, processing_message_id, "❌ فشل في إعداد المتصفح")
                return
            
            login_success, login_msg = self.quick_login(driver)
            if not login_success:
                self.edit_message_safely(chat_id, processing_message_id, f"❌ {login_msg}")
                return
            
            logger.info("✅ تم التسجيل بنجاح")
            
            process_success, result = self.fast_file_processing(driver, file_path)
            
            if process_success:
                # تحويل رسالة المعالجة إلى الكود
                self.edit_message_safely(chat_id, processing_message_id, f"`{result}`", 'Markdown')
            else:
                self.edit_message_safely(chat_id, processing_message_id, f"❌ {result}")
                
        except Exception as e:
            error_msg = f"❌ خطأ غير متوقع: {str(e)}"
            logger.error(error_msg)
            self.edit_message_safely(chat_id, processing_message_id, error_msg)
            
        finally:
            if driver:
                try:
                    driver.quit()
                    logger.info("🔴 تم إغلاق المتصفح")
                except:
                    pass
            
            # حذف الملف المؤقت بعد الانتهاء من المعالجة
            try:
                if os.path.exists(file_path):
                    os.remove(file_path)
                    logger.info(f"🗑️ تم حذف الملف المؤقت: {file_path}")
            except Exception as e:
                logger.error(f"خطأ في حذف الملف المؤقت: {e}")

    def handle_document_processing(self, file_path, chat_id, processing_message_id):
        """معالجة الملف في thread منفصل"""
        try:
            self.process_dat_file(file_path, chat_id, processing_message_id)
        except Exception as e:
            logger.error(f"خطأ في معالجة الملف: {e}")
            self.edit_message_safely(chat_id, processing_message_id, f"❌ خطأ في معالجة الملف: {str(e)}")

    def setup_handlers(self):
        """إعداد معالجات الرسائل"""
        
        @self.bot.message_handler(commands=['start', 'help'])
        def send_welcome(message):
            """ترحيب بالمستخدم"""
            welcome_text = """🔄 بوت استعادة كود الإعادة الموحد

🎯 **الخيارات المتاحة:**

1️⃣ **استخراج كود الإعادة:**
   • أرسل ID متبوعة بالأرقام (مثال: ID0000801941341161210645)

2️⃣ **معالجة ملفات DAT:**
   • أرسل ملف `.dat` مباشرة

⚡ **مميزات البوت:**
   • إعادة تشغيل تلقائي عند التعطل
   • معالجة سريعة للملفات
   • دعم متعدد المستخدمين"""
            
            self.send_message_safely(
                message.chat.id,
                welcome_text,
                message.message_id
            )

        @self.bot.message_handler(commands=['id'])
        def ask_for_id(message):
            """طلب ID من المستخدم - تم تعديله ليعمل كبديل"""
            welcome_text = """🔄 **للاستخدام الصحيح للبوت:**

1️⃣ **استخراج كود الإعادة:**
   • أرسل ID متبوعة بالأرقام مباشرة
   • مثال: `ID0000801941341161210645`

2️⃣ **معالجة ملفات DAT:**
   • أرسل ملف `.dat` مباشرة

📌 **ملاحظة:** لا يتم الرد على أي رسائل نصية أخرى"""
            
            self.send_message_safely(
                message.chat.id,
                welcome_text,
                message.message_id,
                'Markdown'
            )

        def is_valid_id_format(text):
            """التحقق من صحة تنسيق ID"""
            pattern = r'^ID\d+$'
            return re.match(pattern, text.upper()) is not None

        @self.bot.message_handler(func=lambda message: is_valid_id_format(message.text.strip()))
        def handle_id_with_numbers(message):
            """معالجة ID متبوعة بأرقام"""
            chat_id = message.chat.id
            text = message.text.strip()
            
            id_numbers = text[2:]  # إزالة 'ID' من البداية
            
            self.user_data[chat_id] = {
                'step': 'waiting_for_runtime',
                'id': id_numbers,
                'original_message_id': message.message_id
            }
            
            self.send_message_safely(
                chat_id,
                f"✅ تم حفظ الـ ID: {id_numbers}\n\n⏱️ أرسل الـ Runtime:",
                message.message_id
            )

        @self.bot.message_handler(content_types=['document'])
        def handle_document(message):
            """معالجة سريعة للملفات"""
            try:
                # التحقق من امتداد الملف
                if not message.document.file_name.endswith('.dat'):
                    self.send_message_safely(
                        message.chat.id,
                        "❌ يرجى إرسال ملف بامتداد .dat فقط",
                        message.message_id
                    )
                    return
                
                # إرسال رسالة المعالجة
                processing_msg = self.bot.reply_to(
                    message, 
                    "⏳ جاري المعالجة... سيتم إعلامك عند اكتمال العملية"
                )
                
                file_info = self.bot.get_file(message.document.file_id)
                downloaded_file = self.bot.download_file(file_info.file_path)
                
                file_name = f"temp_{int(time.time())}_{message.document.file_id}.dat"
                file_path = os.path.abspath(file_name)
                
                # حفظ الملف
                with open(file_path, 'wb') as f:
                    f.write(downloaded_file)
                
                logger.info(f"✅ تم حفظ الملف المؤقت: {file_path}")
                
                # معالجة الملف في thread منفصل
                thread = threading.Thread(
                    target=self.handle_document_processing,
                    args=(file_path, message.chat.id, processing_msg.message_id)
                )
                thread.daemon = True
                thread.start()
                    
            except Exception as e:
                self.send_message_safely(
                    message.chat.id,
                    f"❌ خطأ: {str(e)}",
                    message.message_id
                )
                logger.error(traceback.format_exc())

        @self.bot.message_handler(commands=['status'])
        def show_status(message):
            """عرض حالة البوت"""
            status_msg = f"🤖 **حالة البوت الموحد:**\n"
            status_msg += f"• عدد إعادة التشغيل: {self.restart_count}\n"
            status_msg += f"• آخر تشغيل: {self.last_restart_time or 'N/A'}\n"
            status_msg += f"• الحالة: 🟢 يعمل"
            self.send_message_safely(
                message.chat.id,
                status_msg,
                message.message_id,
                'Markdown'
            )
            
        @self.bot.message_handler(commands=['myid'])
        def show_user_id(message):
            """عرض معرف المستخدم"""
            user_id = message.from_user.id
            
            user_info = f"""📋 **معلومات المستخدم:**
• المعرف: `{user_id}`
• اليوزر: @{message.from_user.username or 'N/A'}

✅ البوت متاح للجميع للاستخدام"""
            self.send_message_safely(
                message.chat.id,
                user_info,
                message.message_id,
                'Markdown'
            )

        # معالج عام لجميع الرسائل النصية - لن يرد إلا على ID أو ملفات فقط
        @self.bot.message_handler(func=lambda message: True)
        def handle_message(message):
            """معالجة الرسائل الواردة - لن يرد إلا على ID متبوعة بأرقام"""
            chat_id = message.chat.id
            text = message.text.strip()
            
            # التحقق مما إذا كانت الرسالة في سياق انتظار Runtime
            if chat_id in self.user_data:
                user_state = self.user_data[chat_id]
                
                if user_state['step'] == 'waiting_for_runtime':
                    user_state['runtime'] = text
                    user_state['step'] = 'processing'
                    
                    user_id = user_state['id']
                    runtime = user_state['runtime']
                    original_message_id = user_state.get('original_message_id', message.message_id)
                    
                    # إرسال رسالة المعالجة
                    processing_msg = self.bot.send_message(
                        chat_id,
                        "⏳ جاري المعالجة... سيتم إعلامك عند اكتمال العملية",
                        reply_to_message_id=original_message_id
                    )
                    
                    thread = threading.Thread(
                        target=self.process_reset_code,
                        args=(chat_id, user_id, runtime, processing_msg.message_id, message.from_user.id)
                    )
                    thread.daemon = True
                    thread.start()
                    
                    logger.info(f"بدأ thread جديد للمستخدم {chat_id}")
                    return
            
            # إذا لم يكن ID صحيح ولم يكن في سياق معالجة، لا تفعل شيئاً
            # لن يتم الرد على أي رسالة نصية عادية

    def run_with_auto_restart(self):
        """تشغيل البوت مع إعادة التشغيل التلقائي"""
        
        while self.restart_count < self.max_restarts:
            try:
                self.restart_count += 1
                self.last_restart_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
                
                current_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
                logger.info(f"🚀 [{current_time}] Starting UNIFIED bot - Attempt #{self.restart_count}")
                print(f"🔄 [{current_time}] Starting UNIFIED bot - Attempt #{self.restart_count}")
                
                self.initialize_bot()
                
                logger.info("Unified bot started successfully, beginning polling...")
                self.bot.polling(none_stop=True, timeout=60, long_polling_timeout=60)
                
                logger.info("Unified bot stopped gracefully")
                break
                
            except KeyboardInterrupt:
                logger.info("Unified bot stopped by user (Ctrl+C)")
                print("🛑 Unified bot stopped by user")
                break
                
            except Exception as e:
                logger.error(f"❌ Unified bot crashed: {str(e)}")
                print(f"❌ Unified bot crashed: {str(e)}")
                
                if self.restart_count < self.max_restarts:
                    logger.info(f"🔄 Restarting in {self.restart_delay} seconds...")
                    print(f"🔄 Restarting in {self.restart_delay} seconds...")
                    time.sleep(self.restart_delay)
                else:
                    logger.error("🚫 Maximum restart attempts reached")
                    print("🚫 Maximum restart attempts reached")
                    break

def main():
    """الدالة الرئيسية"""
    print("=" * 60)
    print("🔄 Unified Reset Bot - Auto Restart System")
    print("🎯 Features: ID Processing + DAT File Processing")
    print("⚡ Fast Mode + Normal Mode Combined")
    print("🌍 Public Access Mode")
    print("🚫 No User Names Anywhere")
    print("✅ البوت الموحد يعمل الآن")
    print("📌 البوت يرد فقط على:")
    print("   • ملفات .dat")
    print("   • ID متبوعة بأرقام (مثل: ID0000801941341161210645)")
    print("=" * 60)
    
    bot_manager = UnifiedResetBot(TOKEN)
    bot_manager.run_with_auto_restart()
    
    print("🔚 Unified bot process ended")

if __name__ == "__main__":
    main()
