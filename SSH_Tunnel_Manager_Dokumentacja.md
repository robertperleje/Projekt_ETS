# SSH Tunnel Manager - Dokumentacja

> Biblioteka Python do zarządzania tunelami SSH z automatycznym lifecycle management. Idealna do bezpiecznego dostępu do portów niedostępnych publicznie, szczególnie dla JBoss CLI, baz danych i innych serwisów wewnętrznych.

## 📋 Spis treści

- [Wprowadzenie](#wprowadzenie)
- [Jak działa tunel SSH](#jak-działa-tunel-ssh)
- [Instalacja](#instalacja)
- [Funkcja Python - SSHTunnel](#funkcja-python---sshtunnel)
- [Przykłady użycia](#przykłady-użycia)
- [Integracja z Jenkins](#integracja-z-jenkins)
- [Bezpieczeństwo](#bezpieczeństwo)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)

---

## 🎯 Wprowadzenie

### Problem

W środowiskach korporacyjnych często mamy serwisy dostępne tylko lokalnie (np. JBoss CLI na porcie 9990/19990), które nie są eksponowane przez firewall. Bezpośredni dostęp wymagałby otwarcia dodatkowych portów, co:

- ❌ Zwiększa powierzchnię ataku
- ❌ Wymaga dodatkowych zmian w firewallu
- ❌ Komplikuje zarządzanie bezpieczeństwem
- ❌ Nie szyfruje komunikacji

### Rozwiązanie

Tunel SSH pozwala na:

- ✅ Bezpieczny dostęp przez **jeden** port (22)
- ✅ Szyfrowanie całej komunikacji
- ✅ Brak potrzeby otwierania dodatkowych portów
- ✅ Prosty lifecycle management przez Python

---

## 🔐 Jak działa tunel SSH

### Podstawowa koncepcja

SSH Tunnel (Port Forwarding) to mechanizm, który pozwala przekierować ruch sieciowy z lokalnego portu przez szyfrowane połączenie SSH do portu na zdalnym serwerze.

### Wizualizacja przepływu danych

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         ARCHITEKTURA TUNELU SSH                          │
└─────────────────────────────────────────────────────────────────────────┘

Klient (Jenkins)                              Serwer (jboss01)
─────────────────                             ─────────────────

localhost:19990                               localhost:19990
     │                                              ▲
     │ 1. Aplikacja łączy się                       │
     │    z localhost:19990                         │ 4. SSH przekierowuje
     │                                              │    do lokalnego portu
     ▼                                              │
┌──────────────┐                            ┌──────────────┐
│              │  2. Dane są szyfrowane     │              │
│ Tunel SSH    │     i wysyłane przez       │  SSH Daemon  │
│ (port 22)    │═══════ INTERNET ══════════>│  (port 22)   │
│              │     (BEZPIECZNY KANAŁ)     │              │
└──────────────┘                            └──────────────┘
                                                    │
                                                    │ 3. SSH odbiera
                                                    │    i rozpakowuje
                                                    ▼
                                             ┌──────────────┐
                                             │  JBoss CLI   │
                                             │  :19990      │
                                             │ (LOKALNIE)   │
                                             └──────────────┘
```

### Krok po kroku

1. **Utworzenie tunelu:**
   ```bash
   ssh -L 19990:localhost:19990 user@jboss01
   ```

2. **Co się dzieje:**
   - SSH tworzy lokalny port `19990` na kliencie
   - Wszystkie dane wysłane na `localhost:19990` są przechwytywane
   - Dane są szyfrowane algorytmami SSH (AES-256, ChaCha20)
   - Dane wędrują tunelem SSH na port `22` serwera
   - Na serwerze SSH rozpakowuje dane i przekierowuje do `localhost:19990`

3. **Rezultat:**
   - Aplikacja myśli, że łączy się lokalnie z `localhost:19990`
   - W rzeczywistości komunikuje się ze zdalnym serwerem
   - Cała komunikacja jest szyfrowana
   - Nie ma potrzeby otwierania portu `19990` w firewallu

### Porównanie z bezpośrednim połączeniem

#### Bez tunelu SSH (wymaga 2 otwarte porty):

```
Firewall musi przepuścić:
├── Port 22 (SSH)      ✅
└── Port 19990 (CLI)   ✅ ← dodatkowy port do otwarcia!

Bezpieczeństwo: ⚠️ średnie
- Dwa punkty wejścia
- JBoss CLI eksponowany publicznie
- Potencjalnie nieszyfrowana komunikacja z CLI
```

#### Z tunelem SSH (tylko 1 otwarty port):

```
Firewall musi przepuścić:
└── Port 22 (SSH)      ✅ TYLKO TEN!

Port 19990: ❌ NIE MUSI być otwarty - dostępny tylko lokalnie!

Bezpieczeństwo: ✅ wysokie
- Jeden punkt wejścia
- JBoss CLI niewidoczny z zewnątrz
- Cała komunikacja szyfrowana przez SSH
```

### Algorytmy szyfrowania SSH

SSH używa nowoczesnych algorytmów:

- **Wymiana kluczy:** Diffie-Hellman, ECDH, Curve25519
- **Szyfrowanie:** AES-256-GCM, ChaCha20-Poly1305
- **Integralność:** HMAC-SHA2-256, HMAC-SHA2-512
- **Klucze:** RSA-4096, Ed25519, ECDSA

```bash
# Sprawdź szczegóły połączenia:
ssh -v user@host

# Przykładowy output:
# debug1: kex: server->client cipher: aes256-gcm@openssh.com
# debug1: kex: client->server cipher: aes256-gcm@openssh.com
```

---

## 📦 Instalacja

### Wymagania

- Python 3.7+
- OpenSSH client
- Dostęp SSH do serwera docelowego

### Instalacja zależności

```bash
# Standardowa biblioteka Python - brak dodatkowych zależności!
# Używamy tylko modułów built-in:
# - subprocess
# - socket
# - time
# - sys
```

### Konfiguracja SSH

#### 1. Generowanie kluczy SSH (zalecane)

```bash
# Generuj parę kluczy Ed25519 (najnowszy standard)
ssh-keygen -t ed25519 -C "jenkins@company.com"

# Skopiuj klucz publiczny na serwer
ssh-copy-id user@jboss01

# Sprawdź połączenie bez hasła
ssh user@jboss01
```

#### 2. Konfiguracja SSH config (opcjonalne)

Utwórz lub edytuj `~/.ssh/config`:

```bash
Host jboss01
    HostName 10.10.10.25
    User jenkins
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 60
    ServerAliveCountMax 3
    StrictHostKeyChecking no
```

Teraz możesz łączyć się po prostu: `ssh jboss01`

---

## 🐍 Funkcja Python - SSHTunnel

### Pełny kod źródłowy

```python
"""
SSH Tunnel Manager
==================

Biblioteka do zarządzania tunelami SSH z automatycznym lifecycle management.

Autor: DevOps Team
Wersja: 1.0.0
Licencja: MIT
"""

import subprocess
import time
import socket
import sys
import signal
import os


class SSHTunnel:
    """
    Zarządza tunelem SSH z automatycznym lifecycle management.
    
    Obsługuje:
    - Tworzenie tuneli SSH (Local Port Forwarding)
    - Sprawdzanie czy tunel już działa
    - Automatyczne zamykanie tuneli
    - Wykonywanie poleceń przez tunel
    - Context manager dla bezpiecznego zarządzania
    
    Przykład użycia:
        with SSHTunnel('jboss01', 19990) as tunnel:
            # Wykonaj polecenia przez tunel
            tunnel.execute_command(['curl', 'localhost:19990'])
    """
    
    def __init__(self, remote_host, remote_port, local_port=None, ssh_user='jenkins'):
        """
        Inicjalizuje tunel SSH.
        
        Args:
            remote_host (str): Nazwa hosta lub IP docelowego serwera
            remote_port (int): Port na zdalnym hoście (np. 19990 dla JBoss CLI)
            local_port (int, optional): Port lokalny. Domyślnie taki sam jak remote_port
            ssh_user (str, optional): Użytkownik SSH. Domyślnie 'jenkins'
        
        Przykład:
            tunnel = SSHTunnel('jboss01', 19990, local_port=9990, ssh_user='admin')
        """
        self.remote_host = remote_host
        self.remote_port = remote_port
        self.local_port = local_port if local_port else remote_port
        self.ssh_user = ssh_user
        self.tunnel_process = None
        
    def is_port_open(self, host, port):
        """
        Sprawdza czy port jest dostępny.
        
        Args:
            host (str): Nazwa hosta lub IP
            port (int): Numer portu
            
        Returns:
            bool: True jeśli port jest otwarty, False w przeciwnym wypadku
        """
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(2)
            result = sock.connect_ex((host, port))
            sock.close()
            return result == 0
        except Exception as e:
            return False
    
    def is_tunnel_running(self):
        """
        Sprawdza czy tunel SSH już działa.
        
        Używa pgrep do wyszukania procesu SSH z odpowiednimi parametrami.
        
        Returns:
            bool: True jeśli tunel działa, False w przeciwnym wypadku
        """
        cmd = f"pgrep -f 'ssh.*{self.local_port}:localhost:{self.remote_port}.*{self.remote_host}'"
        result = subprocess.run(cmd, shell=True, capture_output=True)
        return result.returncode == 0
    
    def start(self):
        """
        Uruchamia tunel SSH.
        
        Jeśli tunel już działa, nie tworzy nowego.
        Po utworzeniu tunelu czeka 2 sekundy i sprawdza czy port jest dostępny.
        
        Returns:
            bool: True jeśli tunel działa, False w przypadku błędu
            
        Przykład:
            tunnel = SSHTunnel('jboss01', 19990)
            if tunnel.start():
                print("Tunel gotowy!")
        """
        if self.is_tunnel_running():
            print(f"✓ Tunel SSH już działa dla {self.remote_host}:{self.remote_port}")
            return True
        
        print(f"🔧 Tworzę tunel SSH: localhost:{self.local_port} -> {self.remote_host}:{self.remote_port}")
        
        # Komenda SSH do utworzenia tunelu
        ssh_cmd = [
            'ssh',
            '-f',  # Uruchom w tle (fork do background)
            '-N',  # Nie wykonuj poleceń zdalnych (No remote command)
            '-L', f'{self.local_port}:localhost:{self.remote_port}',  # Local port forwarding
            f'{self.ssh_user}@{self.remote_host}',
            '-o', 'StrictHostKeyChecking=no',  # Nie pytaj o fingerprint (produkcja: usuń!)
            '-o', 'ServerAliveInterval=60',    # Ping co 60s
            '-o', 'ServerAliveCountMax=3'      # Zamknij po 3 nieudanych pingach
        ]
        
        try:
            subprocess.run(ssh_cmd, check=True, capture_output=True, text=True)
            time.sleep(2)  # Poczekaj na nawiązanie połączenia
            
            # Sprawdź czy tunel działa
            if self.is_port_open('localhost', self.local_port):
                print(f"✓ Tunel SSH uruchomiony pomyślnie!")
                return True
            else:
                print(f"✗ Tunel SSH nie odpowiada na porcie {self.local_port}")
                return False
                
        except subprocess.CalledProcessError as e:
            print(f"✗ Błąd przy tworzeniu tunelu SSH:")
            print(f"   Stderr: {e.stderr}")
            return False
        except Exception as e:
            print(f"✗ Nieoczekiwany błąd: {e}")
            return False
    
    def stop(self):
        """
        Zatrzymuje tunel SSH.
        
        Używa pkill do zabicia procesu SSH z odpowiednimi parametrami.
        """
        print(f"🔧 Zamykam tunel SSH dla {self.remote_host}:{self.remote_port}")
        cmd = f"pkill -f 'ssh.*{self.local_port}:localhost:{self.remote_port}.*{self.remote_host}'"
        subprocess.run(cmd, shell=True, stderr=subprocess.DEVNULL)
        time.sleep(1)
        print("✓ Tunel SSH zamknięty")
    
    def execute_command(self, command, timeout=60):
        """
        Wykonuje polecenie przez tunel SSH.
        
        Automatycznie uruchamia tunel jeśli nie działa.
        
        Args:
            command (list): Lista z poleceniem i argumentami
            timeout (int): Timeout w sekundach (domyślnie 60)
            
        Returns:
            tuple: (return_code, stdout, stderr)
            
        Przykład:
            tunnel = SSHTunnel('jboss01', 19990)
            rc, out, err = tunnel.execute_command(['curl', 'localhost:19990/health'])
            if rc == 0:
                print(f"Odpowiedź: {out}")
        """
        if not self.start():
            return (1, "", "Nie można uruchomić tunelu SSH")
        
        print(f"▶ Wykonuję polecenie przez tunel...")
        
        try:
            result = subprocess.run(
                command,
                capture_output=True,
                text=True,
                timeout=timeout
            )
            
            return (result.returncode, result.stdout, result.stderr)
            
        except subprocess.TimeoutExpired:
            return (1, "", f"Timeout - polecenie przekroczyło limit {timeout}s")
        except Exception as e:
            return (1, "", f"Błąd wykonania: {str(e)}")
    
    def __enter__(self):
        """
        Context manager - wejście.
        
        Automatycznie uruchamia tunel przy wejściu do bloku 'with'.
        """
        self.start()
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        """
        Context manager - wyjście.
        
        Automatycznie zamyka tunel przy wyjściu z bloku 'with',
        nawet jeśli wystąpił błąd.
        """
        self.stop()
        return False  # Nie przytłumiaj wyjątków


def check_jboss_status(hostname, cli_port=19990, jboss_cli_path='/opt/jboss/bin/jboss-cli.sh'):
    """
    Sprawdza status JBoss przez CLI używając tunelu SSH.
    
    Args:
        hostname (str): Nazwa hosta JBoss
        cli_port (int): Port JBoss CLI (domyślnie 19990)
        jboss_cli_path (str): Ścieżka do jboss-cli.sh
        
    Returns:
        bool: True jeśli JBoss działa, False w przeciwnym wypadku
        
    Przykład:
        if check_jboss_status('jboss01'):
            print("JBoss działa poprawnie!")
    """
    print(f"\n{'='*60}")
    print(f"Sprawdzam status JBoss na: {hostname}:{cli_port}")
    print(f"{'='*60}\n")
    
    # Użyj context managera - tunel zostanie automatycznie zamknięty
    with SSHTunnel(hostname, cli_port) as tunnel:
        # Komenda JBoss CLI
        command = [
            jboss_cli_path,
            '--connect',
            f'--controller=localhost:{cli_port}',
            '--command=:read-attribute(name=server-state)'
        ]
        
        return_code, stdout, stderr = tunnel.execute_command(command)
        
        print(f"\n{'='*60}")
        if return_code == 0:
            print("✓ Status JBoss:")
            print(stdout)
            print(f"{'='*60}\n")
            return True
        else:
            print("✗ Błąd sprawdzania statusu:")
            print(stderr)
            print(f"{'='*60}\n")
            return False


if __name__ == "__main__":
    """
    Przykład użycia z linii poleceń.
    
    Użycie:
        python ssh_tunnel.py <hostname> [cli_port]
        
    Przykład:
        python ssh_tunnel.py jboss01 19990
    """
    if len(sys.argv) < 2:
        print("Użycie: python ssh_tunnel.py <hostname> [cli_port]")
        print("\nPrzykład:")
        print("  python ssh_tunnel.py jboss01 19990")
        sys.exit(1)
    
    hostname = sys.argv[1]
    cli_port = int(sys.argv[2]) if len(sys.argv) > 2 else 19990
    
    success = check_jboss_status(hostname, cli_port)
    sys.exit(0 if success else 1)
```

### API Reference

#### Klasa SSHTunnel

| Metoda | Parametry | Zwraca | Opis |
|--------|-----------|--------|------|
| `__init__()` | `remote_host`, `remote_port`, `local_port=None`, `ssh_user='jenkins'` | - | Inicjalizuje tunel |
| `start()` | - | `bool` | Uruchamia tunel SSH |
| `stop()` | - | - | Zatrzymuje tunel SSH |
| `is_tunnel_running()` | - | `bool` | Sprawdza czy tunel działa |
| `is_port_open()` | `host`, `port` | `bool` | Sprawdza dostępność portu |
| `execute_command()` | `command`, `timeout=60` | `(int, str, str)` | Wykonuje polecenie przez tunel |

#### Funkcje pomocnicze

| Funkcja | Parametry | Zwraca | Opis |
|---------|-----------|--------|------|
| `check_jboss_status()` | `hostname`, `cli_port=19990`, `jboss_cli_path` | `bool` | Sprawdza status JBoss |

---

## 💡 Przykłady użycia

### 1. Podstawowe użycie - Context Manager

```python
from ssh_tunnel import SSHTunnel

# Automatyczne zarządzanie lifecycle
with SSHTunnel('jboss01', 19990) as tunnel:
    # Tunel jest otwarty
    command = ['curl', 'http://localhost:19990/health']
    rc, out, err = tunnel.execute_command(command)
    
    if rc == 0:
        print(f"Status: {out}")
# Tunel zostaje automatycznie zamknięty
```

### 2. Ręczne zarządzanie tunelem

```python
from ssh_tunnel import SSHTunnel

tunnel = SSHTunnel('jboss01', 19990, ssh_user='admin')

try:
    if tunnel.start():
        # Wykonaj operacje
        rc, out, err = tunnel.execute_command(['curl', 'localhost:19990'])
        print(out)
finally:
    tunnel.stop()  # Zawsze zamknij tunel
```

### 3. JBoss CLI - sprawdzanie statusu

```python
from ssh_tunnel import check_jboss_status

# Prosta funkcja do sprawdzania statusu
if check_jboss_status('jboss01', cli_port=19990):
    print("✓ JBoss działa poprawnie")
else:
    print("✗ JBoss nie odpowiada")
```

### 4. Własne polecenia JBoss CLI

```python
from ssh_tunnel import SSHTunnel

with SSHTunnel('jboss01', 19990) as tunnel:
    commands = [
        ':read-attribute(name=server-state)',
        ':read-attribute(name=release-version)',
        'deployment-info',
        '/subsystem=datasources:read-resource'
    ]
    
    for cmd in commands:
        jboss_cmd = [
            '/opt/jboss/bin/jboss-cli.sh',
            '--connect',
            '--controller=localhost:19990',
            f'--command={cmd}'
        ]
        
        rc, out, err = tunnel.execute_command(jboss_cmd)
        print(f"\n{cmd}:\n{out}")
```

### 5. Monitoring wielu serwerów

```python
from ssh_tunnel import SSHTunnel

hosts = ['jboss01', 'jboss02', 'jboss03']
results = {}

for host in hosts:
    print(f"\n{'='*60}")
    print(f"Sprawdzam: {host}")
    print(f"{'='*60}")
    
    try:
        with SSHTunnel(host, 19990) as tunnel:
            command = [
                '/opt/jboss/bin/jboss-cli.sh',
                '--connect',
                '--controller=localhost:19990',
                '--command=:read-attribute(name=server-state)'
            ]
            
            rc, out, err = tunnel.execute_command(command)
            
            if rc == 0 and 'running' in out.lower():
                results[host] = 'RUNNING ✓'
            else:
                results[host] = 'NOT RUNNING ✗'
                
    except Exception as e:
        results[host] = f'ERROR: {e} ✗'

# Podsumowanie
print(f"\n{'='*60}")
print("PODSUMOWANIE")
print(f"{'='*60}")
for host, status in results.items():
    print(f"{host:20} {status}")
```

### 6. Forwarding bazy danych

```python
from ssh_tunnel import SSHTunnel
import psycopg2

# Tunel do PostgreSQL
with SSHTunnel('db-server', 5432, local_port=5433) as tunnel:
    # Połącz się z bazą przez tunel
    conn = psycopg2.connect(
        host='localhost',
        port=5433,
        database='mydb',
        user='user',
        password='pass'
    )
    
    cursor = conn.cursor()
    cursor.execute("SELECT version();")
    print(cursor.fetchone())
    
    conn.close()
```

### 7. Multi-port forwarding

```python
from ssh_tunnel import SSHTunnel

# Otwórz wiele tuneli jednocześnie
tunnels = [
    SSHTunnel('jboss01', 19990, local_port=19990),  # JBoss CLI
    SSHTunnel('jboss01', 8080, local_port=8080),    # HTTP
    SSHTunnel('jboss01', 9990, local_port=9990),    # Management
]

try:
    # Uruchom wszystkie tunele
    for tunnel in tunnels:
        tunnel.start()
    
    # Wykonaj operacje przez tunele
    # ...
    
finally:
    # Zamknij wszystkie tunele
    for tunnel in tunnels:
        tunnel.stop()
```

### 8. Z timeoutem i retry

```python
from ssh_tunnel import SSHTunnel
import time

def execute_with_retry(tunnel, command, retries=3):
    """Wykonaj polecenie z retry"""
    for attempt in range(retries):
        rc, out, err = tunnel.execute_command(command, timeout=30)
        
        if rc == 0:
            return out
        
        print(f"Próba {attempt + 1}/{retries} nieudana, ponawiam...")
        time.sleep(5)
    
    raise Exception(f"Nie udało się wykonać polecenia po {retries} próbach")

# Użycie
with SSHTunnel('jboss01', 19990) as tunnel:
    result = execute_with_retry(tunnel, ['curl', 'localhost:19990/health'])
    print(result)
```

---

## 🔧 Integracja z Jenkins

### Pipeline - podstawowy

```groovy
pipeline {
    agent {
        docker {
            image 'python:3.11-slim'
            args '-v /opt/jboss:/opt/jboss:ro'
        }
    }
    
    parameters {
        string(name: 'JBOSS_HOST', defaultValue: 'jboss01', description: 'Hostname JBoss')
        string(name: 'CLI_PORT', defaultValue: '19990', description: 'Port JBoss CLI')
    }
    
    stages {
        stage('Prepare SSH Tunnel') {
            steps {
                script {
                    // Pobierz bibliotekę z repo
                    sh 'curl -O https://raw.githubusercontent.com/your-repo/ssh_tunnel.py'
                    
                    // Lub skopiuj z workspace
                    // sh 'cp ${WORKSPACE}/ssh_tunnel.py .'
                }
            }
        }
        
        stage('Check JBoss Status') {
            steps {
                sh """
                    python ssh_tunnel.py ${params.JBOSS_HOST} ${params.CLI_PORT}
                """
            }
        }
    }
    
    post {
        always {
            echo 'Tunele SSH zostały automatycznie zamknięte'
        }
    }
}
```

### Pipeline - zaawansowany z wieloma hostami

```groovy
pipeline {
    agent {
        docker {
            image 'python:3.11-slim'
            args '-v /opt/jboss:/opt/jboss:ro -v ~/.ssh:/root/.ssh:ro'
        }
    }
    
    parameters {
        text(
            name: 'JBOSS_HOSTS', 
            defaultValue: 'jboss01\njboss02\njboss03',
            description: 'Lista hostów JBoss (jeden na linię)'
        )
        string(name: 'CLI_PORT', defaultValue: '19990', description: 'Port JBoss CLI')
        choice(
            name: 'COMMAND',
            choices: ['status', 'version', 'deployments', 'datasources'],
            description: 'Polecenie do wykonania'
        )
    }
    
    stages {
        stage('Prepare') {
            steps {
                script {
                    writeFile file: 'ssh_tunnel.py', text: libraryResource('ssh_tunnel.py')
                    
                    writeFile file: 'check_all_hosts.py', text: """
from ssh_tunnel import SSHTunnel
import sys

hosts = '''${params.JBOSS_HOSTS}'''.strip().split('\\n')
cli_port = ${params.CLI_PORT}
command_type = '${params.COMMAND}'

commands = {
    'status': ':read-attribute(name=server-state)',
    'version': ':read-attribute(name=release-version)',
    'deployments': 'deployment-info',
    'datasources': '/subsystem=datasources:read-resource'
}

jboss_command = commands.get(command_type, commands['status'])
results = {}

print(f"\\n{'='*70}")
print(f"Sprawdzam {len(hosts)} serwerów JBoss - Polecenie: {command_type}")
print(f"{'='*70}\\n")

for host in hosts:
    host = host.strip()
    if not host:
        continue
        
    print(f"\\n{'─'*70}")
    print(f"Host: {host}")
    print(f"{'─'*70}")
    
    try:
        with SSHTunnel(host, cli_port) as tunnel:
            cmd = [
                '/opt/jboss/bin/jboss-cli.sh',
                '--connect',
                f'--controller=localhost:{cli_port}',
                f'--command={jboss_command}'
            ]
            
            rc, out, err = tunnel.execute_command(cmd, timeout=30)
            
            if rc == 0:
                print(f"✓ {host}: SUCCESS")
                print(out)
                results[host] = 'SUCCESS'
            else:
                print(f"✗ {host}: FAILED")
                print(err)
                results[host] = 'FAILED'
                
    except Exception as e:
        print(f"✗ {host}: ERROR - {e}")
        results[host] = 'ERROR'

# Podsumowanie
print(f"\\n{'='*70}")
print("PODSUMOWANIE")
print(f"{'='*70}")
success_count = sum(1 for v in results.values() if v == 'SUCCESS')
total_count = len(results)

for host, status in results.items():
    icon = '✓' if status == 'SUCCESS' else '✗'
    print(f"{icon} {host:30} {status}")

print(f"\\n{success_count}/{total_count} serwerów działa poprawnie")
print(f"{'='*70}\\n")

sys.exit(0 if success_count == total_count else 1)
"""
                }
            }
        }
        
        stage('Execute on All Hosts') {
            steps {
                sh 'python check_all_hosts.py'
            }
        }
    }
    
    post {
        success {
            echo '✓ Wszystkie serwery sprawdzone pomyślnie'
        }
        failure {
            echo '✗ Niektóre serwery nie odpowiadają'
        }
        always {
            echo '🧹 Czyszczenie tuneli SSH'
        }
    }
}
```

### Pipeline - deployment verification

```groovy
pipeline {
    agent any
    
    parameters {
        string(name: 'APP_NAME', description: 'Nazwa aplikacji do sprawdzenia')
        string(name: 'EXPECTED_VERSION', description: 'Oczekiwana wersja')
    }
    
    stages {
        stage('Verify Deployment') {
            steps {
                script {
                    writeFile file: 'verify_deployment.py', text: """
from ssh_tunnel import SSHTunnel

app_name = '${params.APP_NAME}'
expected_version = '${params.EXPECTED_VERSION}'
hosts = ['jboss01', 'jboss02', 'jboss03']

all_ok = True

for host in hosts:
    with SSHTunnel(host, 19990) as tunnel:
        cmd = [
            '/opt/jboss/bin/jboss-cli.sh',
            '--connect',
            '--controller=localhost:19990',
            f'--command=/deployment={app_name}:read-attribute(name=runtime-name)'
        ]
        
        rc, out, err = tunnel.execute_command(cmd)
        
        if expected_version in out:
            print(f"✓ {host}: Wersja {expected_version} wdrożona poprawnie")
        else:
            print(f"✗ {host}: Nieprawidłowa wersja!")
            all_ok = False

exit(0 if all_ok else 1)
"""
                    sh 'python verify_deployment.py'
                }
            }
        }
    }
}
```

### Shared Library Jenkins

Możesz też stworzyć Jenkins Shared Library:

```groovy
// vars/sshTunnel.groovy
def check(String host, int port = 19990) {
    sh """
        python -c "
from ssh_tunnel import SSHTunnel, check_jboss_status
import sys
sys.exit(0 if check_jboss_status('${host}', ${port}) else 1)
        "
    """
}

def execute(String host, int port, List command) {
    sh """
        python -c "
from ssh_tunnel import SSHTunnel
with SSHTunnel('${host}', ${port}) as t:
    rc, out, _ = t.execute_command(${command})
    print(out)
    exit(rc)
        "
    """
}
```

Użycie w Pipeline:

```groovy
@Library('shared-lib') _

pipeline {
    agent any
    stages {
        stage('Check') {
            steps {
                sshTunnel.check('jboss01', 19990)
            }
        }
    }
}
```

---

## 🔒 Bezpieczeństwo

### Best Practices

#### 1. Używaj kluczy SSH zamiast haseł

```bash
# Generuj silny klucz
ssh-keygen -t ed25519 -C "jenkins@company.com"

# LUB RSA 4096-bit
ssh-keygen -t rsa -b 4096 -C "jenkins@company.com"

# Ustaw odpowiednie uprawnienia
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

#### 2. Ogranicz dostęp SSH

W `~/.ssh/authorized_keys` na serwerze:

```bash
# Ogranicz do konkretnych komend
command="/opt/scripts/allowed-commands.sh" ssh-ed25519 AAAA...

# Ogranicz do konkretnych IP
from="10.20.30.0/24" ssh-ed25519 AAAA...

# Wyłącz port forwarding (jeśli nie potrzebny)
no-port-forwarding,no-agent-forwarding,no-X11-forwarding ssh-ed25519 AAAA...
```

#### 3. Konfiguracja SSH dla produkcji

Edytuj kod, usuń niebezpieczne opcje:

```python
ssh_cmd = [
    'ssh',
    '-f', '-N',
    '-L', f'{self.local_port}:localhost:{self.remote_port}',
    f'{self.ssh_user}@{self.remote_host}',
    # USUŃ TO W PRODUKCJI:
    # '-o', 'StrictHostKeyChecking=no',  
    # DODAJ TO:
    '-o', 'StrictHostKeyChecking=yes',
    '-o', 'ServerAliveInterval=60',
    '-o', 'ServerAliveCountMax=3',
    '-o', 'ConnectTimeout=10',
    '-o', 'PasswordAuthentication=no'  # Wymuszaj klucze
]
```

#### 4. Zasada najmniejszych uprawnień

```bash
# Dedykowany użytkownik dla tuneli
sudo useradd -m -s /bin/bash jenkins-tunnel
sudo su - jenkins-tunnel

# Generuj klucze dla tego użytkownika
ssh-keygen -t ed25519

# Ogranicz sudo tylko do potrzebnych komend
# /etc/sudoers.d/jenkins-tunnel
jenkins-tunnel ALL=(ALL) NOPASSWD: /usr/bin/systemctl status jboss
```

#### 5. Monitoring i audyt

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('/var/log/ssh-tunnels.log'),
        logging.StreamHandler()
    ]
)

class SSHTunnel:
    def start(self):
        logging.info(f"Creating tunnel: {self.ssh_user}@{self.remote_host}:{self.remote_port}")
        # ... reszta kodu
    
    def stop(self):
        logging.info(f"Closing tunnel: {self.remote_host}:{self.remote_port}")
        # ... reszta kodu
```

#### 6. Secrets management

Nigdy nie hardcoduj credentials w kodzie:

```python
import os

# Pobieraj z zmiennych środowiskowych
SSH_USER = os.getenv('SSH_USER', 'jenkins')
SSH_KEY_PATH = os.getenv('SSH_KEY_PATH', '~/.ssh/id_ed25519')

# Lub z Jenkins Credentials
# W Pipeline: withCredentials([sshUserPrivateKey(...)]) { }
```

#### 7. Network segmentation

```
┌──────────────────────────────────────────────────────────┐
│                     DMZ / Public                          │
│  ┌────────────┐                                          │
│  │  Jenkins   │  SSH (22) tylko do specific hosts        │
│  └─────┬──────┘                                          │
└────────┼──────────────────────────────────────────────────┘
         │ SSH Tunnel
         │ (enkapsuluje wszystkie porty)
         │
┌────────▼──────────────────────────────────────────────────┐
│              Internal Network / Private                    │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐      │
│  │  jboss01   │    │  jboss02   │    │  database  │      │
│  │  :19990    │    │  :19990    │    │  :5432     │      │
│  └────────────┘    └────────────┘    └────────────┘      │
│  (porty dostępne TYLKO lokalnie)                          │
└──────────────────────────────────────────────────────────┘
```

### Checklist bezpieczeństwa

- [ ] Używasz kluczy SSH zamiast haseł
- [ ] `StrictHostKeyChecking` ustawione na `yes` w produkcji
- [ ] Klucze SSH mają odpowiednie uprawnienia (600)
- [ ] Używasz dedykowanego użytkownika dla tuneli
- [ ] Włączony logging wszystkich połączeń
- [ ] Tunele mają timeout i automatycznie się zamykają
- [ ] Secrets są w zmiennych środowiskowych, nie w kodzie
- [ ] Ograniczony dostęp przez `authorized_keys`
- [ ] Regularnie rotowane klucze SSH
- [ ] Monitoring nieautoryzowanych prób połączeń

---

## 🔍 Troubleshooting

### Problem: "Connection refused"

```
✗ ssh: connect to host jboss01 port 22: Connection refused
```

**Rozwiązania:**

```bash
# 1. Sprawdź czy SSH działa na serwerze
ssh -v user@jboss01

# 2. Sprawdź czy port 22 jest otwarty
nc -zv jboss01 22
telnet jboss01 22

# 3. Sprawdź firewall na kliencie
sudo iptables -L -n | grep 22

# 4. Sprawdź czy serwer SSH działa
# (na serwerze docelowym)
sudo systemctl status sshd
sudo systemctl start sshd
```

### Problem: "Permission denied (publickey)"

```
✗ Permission denied (publickey,password).
```

**Rozwiązania:**

```bash
# 1. Sprawdź czy klucz jest dodany
ssh-add -l

# 2. Dodaj klucz do ssh-agent
eval $(ssh-agent)
ssh-add ~/.ssh/id_ed25519

# 3. Skopiuj klucz na serwer
ssh-copy-id user@jboss01

# 4. Sprawdź uprawnienia
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub

# 5. Debug
ssh -vvv user@jboss01
```

### Problem: Tunel się tworzy, ale port nie odpowiada

```
✓ Tunel SSH uruchomiony pomyślnie!
✗ Tunel SSH nie odpowiada na porcie 19990
```

**Rozwiązania:**

```bash
# 1. Sprawdź czy usługa działa na zdalnym serwerze
# (na serwerze jboss01)
netstat -tlnp | grep 19990
ss -tlnp | grep 19990

# 2. Sprawdź czy tunel faktycznie działa
ps aux | grep ssh | grep 19990

# 3. Sprawdź czy lokalny port jest wolny
netstat -tlnp | grep 19990  # na kliencie

# 4. Test manualny
ssh -L 19990:localhost:19990 user@jboss01
# W innym terminalu:
curl localhost:19990

# 5. Sprawdź logi SSH
tail -f /var/log/auth.log  # na serwerze
```

### Problem: Tunel się rozłącza

```
✗ Broken pipe
```

**Rozwiązania:**

```python
# Dodaj keep-alive w kodzie
ssh_cmd = [
    'ssh', '-f', '-N',
    '-L', f'{self.local_port}:localhost:{self.remote_port}',
    f'{self.ssh_user}@{self.remote_host}',
    '-o', 'ServerAliveInterval=30',     # ping co 30s
    '-o', 'ServerAliveCountMax=5',      # 5 prób
    '-o', 'TCPKeepAlive=yes'            # TCP keep-alive
]
```

```bash
# Lub w ~/.ssh/config
Host jboss01
    ServerAliveInterval 30
    ServerAliveCountMax 5
    TCPKeepAlive yes
```

### Problem: "Address already in use"

```
✗ bind: Address already in use
```

**Rozwiązania:**

```bash
# 1. Znajdź proces używający portu
lsof -i :19990
netstat -tlnp | grep 19990

# 2. Zabij proces
kill <PID>

# 3. Lub użyj innego lokalnego portu
tunnel = SSHTunnel('jboss01', 19990, local_port=19991)
```

### Problem: Timeout podczas wykonywania polecenia

```
✗ Timeout - polecenie przekroczyło limit 60s
```

**Rozwiązania:**

```python
# Zwiększ timeout
rc, out, err = tunnel.execute_command(command, timeout=300)  # 5 minut

# Lub podziel na mniejsze operacje
# Zamiast jednego dużego zapytania, zrób kilka małych
```

### Problem: "Host key verification failed"

```
✗ Host key verification failed.
```

**Rozwiązania:**

```bash
# 1. Dodaj host do known_hosts
ssh-keyscan jboss01 >> ~/.ssh/known_hosts

# 2. Lub akceptuj przy pierwszym połączeniu
ssh user@jboss01  # wpisz 'yes'

# 3. W produkcji NIE używaj StrictHostKeyChecking=no!
```

### Debug Mode

Włącz szczegółowy logging:

```python
import subprocess
import sys

# Dodaj verbose do SSH
ssh_cmd = [
    'ssh', '-v', '-v', '-v',  # Triple verbose
    '-f', '-N',
    '-L', f'{local_port}:localhost:{remote_port}',
    f'{ssh_user}@{remote_host}'
]

result = subprocess.run(ssh_cmd, capture_output=True, text=True)
print("STDOUT:", result.stdout)
print("STDERR:", result.stderr)
```

### Często sprawdzane komendy

```bash
# Lista wszystkich tuneli SSH
ps aux | grep 'ssh.*-L'

# Zabij wszystkie tunele
pkill -f 'ssh.*-L'

# Sprawdź otwarte połączenia
netstat -tn | grep :22

# Test połączenia bez tunelu
nc -zv jboss01 22

# Test połączenia przez tunel
nc -zv localhost 19990
```

---

## ❓ FAQ

### Q: Czy tunel SSH jest bezpieczny?

**A:** Tak! SSH używa silnego szyfrowania (AES-256, ChaCha20) i jest standardem w branży. Wszystkie dane przesyłane przez tunel są zaszyfrowane end-to-end.

### Q: Czy mogę używać jednego tunelu dla wielu połączeń?

**A:** Tak! Jeden tunel SSH może obsługiwać wiele równoległych połączeń do tego samego portu. SSH multiplexuje połączenia.

```python
with SSHTunnel('jboss01', 19990) as tunnel:
    # Możesz wykonać wiele operacji równolegle
    thread1 = threading.Thread(target=operation1)
    thread2 = threading.Thread(target=operation2)
    thread1.start()
    thread2.start()
```

### Q: Jak wpływa to na wydajność?

**A:** Minimalne narzuty:
- Szyfrowanie: ~1-5% CPU
- Latencja: +1-10ms (przez sieć)
- Throughput: 90-95% natywnej przepustowości

Dla większości zastosowań (CLI, API) różnica jest niezauważalna.

### Q: Co jeśli muszę forwardować wiele portów?

**A:** Możesz utworzyć wiele tuneli:

```python
tunnels = [
    SSHTunnel('jboss01', 19990),  # CLI
    SSHTunnel('jboss01', 8080),   # HTTP
    SSHTunnel('jboss01', 9990),   # Management
]

for tunnel in tunnels:
    tunnel.start()
```

Lub użyć jednego połączenia SSH z wieloma forwardingami:

```bash
ssh -L 19990:localhost:19990 \
    -L 8080:localhost:8080 \
    -L 9990:localhost:9990 \
    user@jboss01
```

### Q: Jak obsłużyć sytuację, gdy serwer ma wiele interfejsów?

**A:** Możesz forwardować do konkretnego interfejsu:

```python
# Forward do konkretnego IP na zdalnym hoście
ssh_cmd = [
    'ssh', '-f', '-N',
    '-L', f'{local_port}:192.168.1.100:{remote_port}',  # konkretny IP
    f'{ssh_user}@{remote_host}'
]
```

### Q: Czy mogę używać tego w kontenerach Docker?

**A:** Tak! Upewnij się, że:

1. Masz zainstalowany SSH client w kontenerze:
   ```dockerfile
   RUN apt-get update && apt-get install -y openssh-client
   ```

2. Klucze SSH są dostępne:
   ```bash
   docker run -v ~/.ssh:/root/.ssh:ro myimage
   ```

3. Sieć kontenera ma dostęp do hosta docelowego.

### Q: Jak długo może działać tunel?

**A:** Tunel może działać w nieskończoność, jeśli:
- Włączony ServerAliveInterval (ping co X sekund)
- Sieć jest stabilna
- Nie ma timeoutów na firewallu

```python
ssh_cmd = [
    'ssh', '-f', '-N',
    '-o', 'ServerAliveInterval=60',    # Ping co 60s
    '-o', 'ServerAliveCountMax=3',     # 3 nieudane = disconnect
    '-o', 'ExitOnForwardFailure=yes',  # Wyjdź przy błędzie forwarding
    # ... rest
]
```

### Q: Co z proxy/jump hosts?

**A:** SSH obsługuje ProxyJump:

```python
ssh_cmd = [
    'ssh', '-f', '-N',
    '-J', 'jumphost',  # Połącz przez jumphost
    '-L', f'{local_port}:localhost:{remote_port}',
    f'{ssh_user}@{remote_host}'
]
```

Lub w `~/.ssh/config`:
```
Host jboss01
    ProxyJump jumphost
```

### Q: Jak monitorować tunele w produkcji?

**A:** Dodaj monitoring:

```python
import prometheus_client
from prometheus_client import Counter, Gauge

tunnel_created = Counter('ssh_tunnel_created', 'Tunnels created')
tunnel_active = Gauge('ssh_tunnel_active', 'Active tunnels')
tunnel_errors = Counter('ssh_tunnel_errors', 'Tunnel errors')

class SSHTunnel:
    def start(self):
        tunnel_created.inc()
        tunnel_active.inc()
        # ... existing code
        
    def stop(self):
        tunnel_active.dec()
        # ... existing code
```

### Q: Czy można używać z Windows?

**A:** Tak, jeśli masz SSH client:

1. Windows 10/11 - wbudowany OpenSSH:
   ```powershell
   ssh -L 19990:localhost:19990 user@jboss01
   ```

2. Git Bash - zawiera SSH
3. WSL - pełne SSH capabilities

Kod Python działa identycznie.

### Q: Jak testować lokalnie?

**A:** Użyj localhost:

```python
# Testuj bez prawdziwego SSH
tunnel = SSHTunnel('localhost', 19990, ssh_user='yourusername')
tunnel.start()
```

Lub mock SSH w testach:

```python
import unittest
from unittest.mock import patch, MagicMock

class TestSSHTunnel(unittest.TestCase):
    @patch('subprocess.run')
    def test_start_tunnel(self, mock_run):
        mock_run.return_value = MagicMock(returncode=0)
        
        tunnel = SSHTunnel('testhost', 19990)
        result = tunnel.start()
        
        self.assertTrue(result)
        mock_run.assert_called_once()
```

---

## 📚 Dodatkowe zasoby

### Dokumentacja

- [OpenSSH Manual](https://www.openssh.com/manual.html)
- [SSH Port Forwarding](https://www.ssh.com/academy/ssh/tunneling/example)
- [Python subprocess](https://docs.python.org/3/library/subprocess.html)

### Bezpieczeństwo

- [SSH Best Practices](https://www.ssh.com/academy/ssh/security)
- [NIST SSH Guidelines](https://nvlpubs.nist.gov/nistpubs/ir/2015/NIST.IR.7966.pdf)

### Narzędzia

- [autossh](https://www.harding.motd.ca/autossh/) - Auto-restart SSH tunnels
- [sshuttle](https://github.com/sshuttle/sshuttle) - VPN over SSH
- [mosh](https://mosh.org/) - Mobile Shell (alternative to SSH)

---

## 📝 Changelog

### v1.0.0 (2025-10-31)

- ✨ Pierwsza wersja
- 🔐 Context manager dla automatycznego lifecycle
- 📊 Sprawdzanie statusu tunelu
- ⏱️ Timeout dla poleceń
- 📝 Pełna dokumentacja
- 🧪 Przykłady użycia
- 🔧 Integracja z Jenkins

---

## 📄 Licencja

MIT License - możesz swobodnie używać w projektach komercyjnych i open source.

---

## 👥 Autorzy

Utworzone dla projektów DevOps/SRE wymagających bezpiecznego dostępu do serwisów wewnętrznych.

---

## 🤝 Wkład

Zgłaszaj issues i pull requesty na GitHubie!

### Jak pomóc:

1. Fork repozytorium
2. Utwórz branch (`git checkout -b feature/amazing-feature`)
3. Commit zmian (`git commit -m 'Add amazing feature'`)
4. Push do brancha (`git push origin feature/amazing-feature`)
5. Otwórz Pull Request

---

## 📞 Kontakt

Pytania? Problemy? Otwórz issue na GitHubie!

---

**Happy Tunneling! 🚀**
