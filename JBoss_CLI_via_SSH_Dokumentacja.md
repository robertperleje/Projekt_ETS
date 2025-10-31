# JBoss CLI via SSH - Dokumentacja

> Biblioteka Python do bezpiecznego wykonywania komend JBoss CLI przez SSH. Prostsza alternatywa dla tuneli SSH - wszystko działa przez port 22 bez potrzeby otwierania dodatkowych portów.

## 📋 Spis treści

- [Wprowadzenie](#wprowadzenie)
- [Jak działa SSH + JBoss CLI](#jak-działa-ssh--jboss-cli)
- [Instalacja](#instalacja)
- [Funkcja Python - JBossSSH](#funkcja-python---jbossssh)
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

### Rozwiązanie

Bezpośrednie SSH do wykonywania komend JBoss CLI:

- ✅ Bezpieczny dostęp przez **jeden** port (22)
- ✅ Szyfrowanie całej komunikacji przez SSH
- ✅ Brak potrzeby otwierania dodatkowych portów (9990/19990)
- ✅ Proste wykonywanie komend CLI przez Python
- ✅ JBoss CLI działa lokalnie na serwerze (tam gdzie JBoss)

---

## 🔐 Jak działa SSH + JBoss CLI

### Podstawowa koncepcja

Zamiast tworzyć tunel SSH i przekierowywać porty, używamy SSH do **bezpośredniego wykonywania komend** JBoss CLI na serwerze zdalnym. JBoss CLI działa lokalnie na serwerze (gdzie ma dostęp do localhost:9990), a my tylko wysyłamy komendy przez SSH.

### Wizualizacja przepływu danych

```
┌─────────────────────────────────────────────────────────────────────────┐
│              ARCHITEKTURA - BEZPOŚREDNIE SSH DO JBOSS CLI                │
└─────────────────────────────────────────────────────────────────────────┘

Klient (Jenkins)                              Serwer (jboss01)
─────────────────                             ─────────────────

┌──────────────┐                            ┌──────────────────────────┐
│              │                            │                          │
│  Python      │  1. SSH Connection         │   SSH Daemon (port 22)  │
│  Script      ├───────────────────────────>│                          │
│              │  Komenda: "uruchom CLI"    │                          │
└──────────────┘                            └───────────┬──────────────┘
                                                        │
                                                        │ 2. SSH uruchamia
                                                        │    jboss-cli.sh
                                                        ▼
                                            ┌──────────────────────────┐
                                            │  jboss-cli.sh            │
                                            │  (działa LOKALNIE)       │
                                            └───────────┬──────────────┘
                                                        │
                                                        │ 3. Łączy się
                                                        │    localhost:9990
                                                        ▼
                                            ┌──────────────────────────┐
                                            │   JBoss Server           │
                                            │   Management CLI: 9990   │
                                            │   (dostępny lokalnie)    │
                                            └──────────────────────────┘

═══════════════════════════════════════════════════════════════════════════
Cała komunikacja ZASZYFROWANA przez SSH (AES-256, ChaCha20)
Port 9990/19990 NIE musi być otwarty w firewallu!
═══════════════════════════════════════════════════════════════════════════
```

### Krok po kroku

1. **Wywołanie komendy przez SSH:**
   ```bash
   ssh user@jboss01 '/opt/jboss/bin/jboss-cli.sh --connect --controller=localhost:9990 --command=":read-attribute(name=server-state)"'
   ```

2. **Co się dzieje:**
   - Jenkins nawiązuje połączenie SSH na port `22` serwera jboss01
   - SSH uruchamia `jboss-cli.sh` **na serwerze jboss01**
   - `jboss-cli.sh` łączy się z **localhost:9990** (lokalnie na serwerze)
   - JBoss Management Interface odpowiada z danymi
   - Wynik wraca przez SSH do Jenkins
   - Całość jest szyfrowana przez SSH

3. **Rezultat:**
   - Nie ma potrzeby tunelowania portów
   - Nie trzeba instalować jboss-cli.sh na Jenkins
   - Port 9990/19990 pozostaje zamknięty w firewallu
   - Cała komunikacja jest szyfrowana
   - Prostsze i szybsze niż tunelowanie

### Porównanie z innymi metodami

#### Bezpośrednie połączenie z JBoss CLI (NIEBEZPIECZNE):

```
Firewall musi przepuścić:
├── Port 22 (SSH)       ✅
└── Port 9990 (CLI)     ✅ ← dodatkowy port, potencjalne zagrożenie!

Bezpieczeństwo: ⚠️ niskie
- JBoss Management Interface eksponowany publicznie
- Dwa punkty wejścia
- Komunikacja z CLI może być nieszyfrowana
```

#### SSH + zdalne wykonanie CLI (ZALECANE):

```
Firewall musi przepuścić:
└── Port 22 (SSH)       ✅ TYLKO TEN!

Port 9990: ❌ NIE musi być otwarty - dostępny tylko lokalnie!

Bezpieczeństwo: ✅ wysokie
- Jeden punkt wejścia (SSH)
- JBoss CLI niewidoczny z zewnątrz
- Cała komunikacja szyfrowana przez SSH
- Standardowe mechanizmy uwierzytelniania SSH
```

### Szyfrowanie SSH

SSH używa nowoczesnych algorytmów:

- **Wymiana kluczy:** Diffie-Hellman, ECDH, Curve25519
- **Szyfrowanie:** AES-256-GCM, ChaCha20-Poly1305
- **Integralność:** HMAC-SHA2-256, HMAC-SHA2-512
- **Autentykacja:** RSA-4096, Ed25519, ECDSA

```bash
# Sprawdź szczegóły połączenia:
ssh -v user@host

# Przykładowy output:
# debug1: kex: server->client cipher: aes256-gcm@openssh.com
# debug1: kex: client->server cipher: aes256-gcm@openssh.com
```

### Dlaczego to lepsze od tuneli?

| Aspekt | Tunel SSH | Bezpośrednie SSH |
|--------|-----------|------------------|
| **Złożoność** | ⚠️ Średnia (zarządzanie tunelem) | ✅ Niska (jedna komenda) |
| **Latencja** | ⚠️ Wyższa (2 hopy) | ✅ Niższa (1 hop) |
| **Punkty awarii** | ⚠️ Więcej (tunel + CLI) | ✅ Mniej (tylko SSH) |
| **Gdzie działa CLI** | Na kliencie (Jenkins) | ✅ Na serwerze (lokalnie) |
| **Wymagania** | jboss-cli.sh na Jenkins | ✅ jboss-cli.sh tylko na serwerze |
| **Niezawodność** | ⚠️ Średnia | ✅ Wysoka |

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

## 🐍 Funkcja Python - JBossSSH

### Pełny kod źródłowy

```python
"""
JBoss CLI via SSH
=================

Biblioteka do bezpośredniego wykonywania komend JBoss CLI przez SSH.
Prostsza i bardziej niezawodna alternatywa dla tuneli SSH.

Autor: DevOps Team
Wersja: 2.0.0
Licencja: MIT
"""

import subprocess
import sys
import time


class JBossSSH:
    """
    Wykonuje komendy JBoss CLI przez SSH bez potrzeby tunelowania portów.
    
    CLI działa lokalnie na serwerze JBoss, przez co:
    - Port 9990/19990 nie musi być otwarty w firewallu
    - Cała komunikacja jest szyfrowana przez SSH
    - Nie ma potrzeby instalowania jboss-cli.sh na kliencie
    - Prostsze i bardziej niezawodne niż tunelowanie
    
    Przykład użycia:
        jboss = JBossSSH('jboss01')
        if jboss.check_status():
            print("JBoss działa!")
    """
    
    def __init__(self, host, ssh_user='jenkins', cli_path='/opt/jboss/bin/jboss-cli.sh', cli_port=9990):
        """
        Inicjalizuje połączenie SSH do JBoss.
        
        Args:
            host (str): Nazwa hosta lub IP serwera JBoss
            ssh_user (str, optional): Użytkownik SSH. Domyślnie 'jenkins'
            cli_path (str, optional): Ścieżka do jboss-cli.sh na serwerze
            cli_port (int, optional): Port JBoss Management CLI. Domyślnie 9990
        
        Przykład:
            jboss = JBossSSH('jboss01', ssh_user='admin', cli_port=19990)
        """
        self.host = host
        self.ssh_user = ssh_user
        self.cli_path = cli_path
        self.cli_port = cli_port
    
    def execute_cli_command(self, command, timeout=60):
        """
        Wykonuje komendę JBoss CLI przez SSH.
        
        Args:
            command (str): Komenda JBoss CLI (np. ':read-attribute(name=server-state)')
            timeout (int, optional): Timeout w sekundach. Domyślnie 60
        
        Returns:
            tuple: (return_code, stdout, stderr)
            
        Przykład:
            rc, out, err = jboss.execute_cli_command(':read-attribute(name=server-state)')
            if rc == 0:
                print(f"Status: {out}")
        """
        # Komenda SSH z zagnieżdżoną komendą JBoss CLI
        ssh_command = [
            'ssh',
            '-o', 'ConnectTimeout=10',
            '-o', 'BatchMode=yes',  # Nie pytaj o hasło (użyj kluczy)
            '-o', 'StrictHostKeyChecking=no',  # W produkcji zmień na 'yes'
            f'{self.ssh_user}@{self.host}',
            f'{self.cli_path} --connect --controller=localhost:{self.cli_port} --command="{command}"'
        ]
        
        print(f"🔧 Wykonuję komendę: {command}")
        print(f"📡 Serwer: {self.host}:{self.cli_port}")
        
        try:
            result = subprocess.run(
                ssh_command,
                capture_output=True,
                text=True,
                timeout=timeout
            )
            
            return (result.returncode, result.stdout, result.stderr)
            
        except subprocess.TimeoutExpired:
            return (1, "", f"Timeout - polecenie przekroczyło limit {timeout}s")
        except FileNotFoundError:
            return (1, "", "SSH nie jest zainstalowane lub niedostępne")
        except Exception as e:
            return (1, "", f"Błąd wykonania: {str(e)}")
    
    def check_status(self):
        """
        Sprawdza status serwera JBoss.
        
        Returns:
            bool: True jeśli JBoss działa, False w przeciwnym wypadku
            
        Przykład:
            if jboss.check_status():
                print("✓ JBoss działa poprawnie")
        """
        print(f"\n{'='*60}")
        print(f"Sprawdzam status JBoss: {self.host}")
        print(f"{'='*60}\n")
        
        rc, out, err = self.execute_cli_command(':read-attribute(name=server-state)')
        
        if rc == 0 and 'running' in out.lower():
            print(f"✓ {self.host}: RUNNING")
            print(out)
            print(f"{'='*60}\n")
            return True
        else:
            print(f"✗ {self.host}: NOT RUNNING")
            if err:
                print(f"Error: {err}")
            print(f"{'='*60}\n")
            return False
    
    def get_version(self):
        """
        Pobiera wersję JBoss.
        
        Returns:
            str: Wersja JBoss lub None w przypadku błędu
            
        Przykład:
            version = jboss.get_version()
            print(f"Wersja: {version}")
        """
        rc, out, err = self.execute_cli_command(':read-attribute(name=release-version)')
        
        if rc == 0:
            # Parsuj output - zwykle format: "result" => "X.Y.Z"
            for line in out.split('\n'):
                if '=>' in line and 'result' in line:
                    return line.split('=>')[1].strip().strip('"')
            return out.strip()
        return None
    
    def list_deployments(self):
        """
        Pobiera listę deploymentów.
        
        Returns:
            str: Lista deploymentów lub None w przypadku błędu
            
        Przykład:
            deployments = jboss.list_deployments()
            print(deployments)
        """
        rc, out, err = self.execute_cli_command('deployment-info')
        return out if rc == 0 else None
    
    def get_deployment_status(self, deployment_name):
        """
        Sprawdza status konkretnego deploymentu.
        
        Args:
            deployment_name (str): Nazwa deploymentu
            
        Returns:
            bool: True jeśli deployment jest włączony, False w przeciwnym wypadku
        """
        command = f'/deployment={deployment_name}:read-attribute(name=enabled)'
        rc, out, err = self.execute_cli_command(command)
        
        if rc == 0 and 'true' in out.lower():
            return True
        return False
    
    def read_datasources(self):
        """
        Pobiera informacje o źródłach danych (datasources).
        
        Returns:
            str: Informacje o datasources lub None w przypadku błędu
        """
        rc, out, err = self.execute_cli_command('/subsystem=datasources:read-resource')
        return out if rc == 0 else None
    
    def execute_multiple_commands(self, commands, stop_on_error=True):
        """
        Wykonuje wiele komend CLI.
        
        Args:
            commands (list): Lista komend do wykonania
            stop_on_error (bool): Czy zatrzymać przy pierwszym błędzie
            
        Returns:
            list: Lista tupli (command, return_code, stdout, stderr)
            
        Przykład:
            commands = [
                ':read-attribute(name=server-state)',
                ':read-attribute(name=release-version)',
                'deployment-info'
            ]
            results = jboss.execute_multiple_commands(commands)
        """
        results = []
        
        for command in commands:
            rc, out, err = self.execute_cli_command(command)
            results.append((command, rc, out, err))
            
            if stop_on_error and rc != 0:
                print(f"⚠️ Zatrzymano wykonywanie - błąd w komendzie: {command}")
                break
        
        return results
    
    def test_connection(self):
        """
        Testuje połączenie SSH (bez uruchamiania JBoss CLI).
        
        Returns:
            bool: True jeśli połączenie działa, False w przeciwnym wypadku
        """
        ssh_test = [
            'ssh',
            '-o', 'ConnectTimeout=5',
            '-o', 'BatchMode=yes',
            f'{self.ssh_user}@{self.host}',
            'echo "SSH OK"'
        ]
        
        try:
            result = subprocess.run(ssh_test, capture_output=True, text=True, timeout=10)
            return result.returncode == 0 and 'SSH OK' in result.stdout
        except:
            return False


def check_multiple_hosts(hosts, cli_port=9990, ssh_user='jenkins'):
    """
    Sprawdza status JBoss na wielu hostach.
    
    Args:
        hosts (list): Lista nazw hostów
        cli_port (int): Port JBoss CLI
        ssh_user (str): Użytkownik SSH
        
    Returns:
        dict: Słownik {host: status} gdzie status to bool
        
    Przykład:
        results = check_multiple_hosts(['jboss01', 'jboss02', 'jboss03'])
        print(f"Działających: {sum(results.values())}/{len(results)}")
    """
    results = {}
    
    print(f"\n{'='*70}")
    print(f"Sprawdzam {len(hosts)} serwerów JBoss")
    print(f"{'='*70}\n")
    
    for host in hosts:
        jboss = JBossSSH(host, ssh_user=ssh_user, cli_port=cli_port)
        results[host] = jboss.check_status()
    
    # Podsumowanie
    print(f"\n{'='*70}")
    print("PODSUMOWANIE")
    print(f"{'='*70}")
    
    success_count = sum(results.values())
    for host, status in results.items():
        icon = '✓' if status else '✗'
        status_text = 'RUNNING' if status else 'NOT RUNNING'
        print(f"{icon} {host:30} {status_text}")
    
    print(f"\n{success_count}/{len(results)} serwerów działa poprawnie")
    print(f"{'='*70}\n")
    
    return results


if __name__ == "__main__":
    """
    Przykład użycia z linii poleceń.
    
    Użycie:
        python jboss_ssh.py <hostname> [cli_port]
        
    Przykład:
        python jboss_ssh.py jboss01 9990
        python jboss_ssh.py jboss01  # domyślnie port 9990
    """
    if len(sys.argv) < 2:
        print("Użycie: python jboss_ssh.py <hostname> [cli_port]")
        print("\nPrzykład:")
        print("  python jboss_ssh.py jboss01")
        print("  python jboss_ssh.py jboss01 19990")
        sys.exit(1)
    
    hostname = sys.argv[1]
    cli_port = int(sys.argv[2]) if len(sys.argv) > 2 else 9990
    
    jboss = JBossSSH(hostname, cli_port=cli_port)
    
    # Test połączenia SSH
    if not jboss.test_connection():
        print("✗ Błąd połączenia SSH!")
        sys.exit(1)
    
    # Sprawdź status
    success = jboss.check_status()
    
    if success:
        # Pokaż dodatkowe informacje
        version = jboss.get_version()
        if version:
            print(f"Wersja JBoss: {version}\n")
    
    sys.exit(0 if success else 1)
```

### API Reference

#### Klasa JBossSSH

| Metoda | Parametry | Zwraca | Opis |
|--------|-----------|--------|------|
| `__init__()` | `host`, `ssh_user='jenkins'`, `cli_path`, `cli_port=9990` | - | Inicjalizuje połączenie |
| `execute_cli_command()` | `command`, `timeout=60` | `(int, str, str)` | Wykonuje komendę CLI |
| `check_status()` | - | `bool` | Sprawdza status JBoss |
| `get_version()` | - | `str` | Pobiera wersję JBoss |
| `list_deployments()` | - | `str` | Lista deploymentów |
| `get_deployment_status()` | `deployment_name` | `bool` | Status deploymentu |
| `read_datasources()` | - | `str` | Informacje o datasources |
| `execute_multiple_commands()` | `commands`, `stop_on_error=True` | `list` | Wykonuje wiele komend |
| `test_connection()` | - | `bool` | Testuje połączenie SSH |

#### Funkcje pomocnicze

| Funkcja | Parametry | Zwraca | Opis |
|---------|-----------|--------|------|
| `check_multiple_hosts()` | `hosts`, `cli_port=9990`, `ssh_user` | `dict` | Sprawdza wiele hostów |

---

## 💡 Przykłady użycia

### 1. Podstawowe użycie - sprawdzenie statusu

```python
from jboss_ssh import JBossSSH

# Sprawdź status JBoss
jboss = JBossSSH('jboss01')

if jboss.check_status():
    print("✓ JBoss działa poprawnie")
else:
    print("✗ JBoss nie odpowiada")
```

### 2. Pobieranie informacji o JBoss

```python
from jboss_ssh import JBossSSH

jboss = JBossSSH('jboss01', cli_port=19990)

# Sprawdź wersję
version = jboss.get_version()
print(f"Wersja JBoss: {version}")

# Lista deploymentów
deployments = jboss.list_deployments()
print(f"Deploymenty:\n{deployments}")

# Informacje o datasources
datasources = jboss.read_datasources()
print(f"Datasources:\n{datasources}")
```

### 3. Własne komendy CLI

```python
from jboss_ssh import JBossSSH

jboss = JBossSSH('jboss01')

# Wykonaj własną komendę
rc, out, err = jboss.execute_cli_command(':read-attribute(name=launch-type)')

if rc == 0:
    print(f"Launch type: {out}")
else:
    print(f"Błąd: {err}")
```

### 4. Sprawdzanie statusu deploymentu

```python
from jboss_ssh import JBossSSH

jboss = JBossSSH('jboss01')

app_name = 'myapp.war'

if jboss.get_deployment_status(app_name):
    print(f"✓ {app_name} jest wdrożony i włączony")
else:
    print(f"✗ {app_name} nie jest dostępny")
```

### 5. Wykonywanie wielu komend

```python
from jboss_ssh import JBossSSH

jboss = JBossSSH('jboss01')

commands = [
    ':read-attribute(name=server-state)',
    ':read-attribute(name=release-version)',
    ':read-attribute(name=launch-type)',
    'deployment-info'
]

results = jboss.execute_multiple_commands(commands)

for cmd, rc, out, err in results:
    if rc == 0:
        print(f"✓ {cmd}")
        print(f"  {out[:100]}...")  # Pierwsze 100 znaków
    else:
        print(f"✗ {cmd}: {err}")
```

### 6. Monitoring wielu serwerów

```python
from jboss_ssh import check_multiple_hosts

hosts = ['jboss01', 'jboss02', 'jboss03']

results = check_multiple_hosts(hosts, cli_port=9990)

# Sprawdź czy wszystkie działają
if all(results.values()):
    print("✓ Wszystkie serwery działają poprawnie")
else:
    failed = [host for host, status in results.items() if not status]
    print(f"✗ Problemy z serwerami: {', '.join(failed)}")
```

### 7. Test połączenia SSH przed użyciem CLI

```python
from jboss_ssh import JBossSSH

jboss = JBossSSH('jboss01')

# Najpierw test połączenia SSH
if not jboss.test_connection():
    print("✗ Brak połączenia SSH - sprawdź dostęp")
    exit(1)

# Teraz możesz bezpiecznie używać CLI
if jboss.check_status():
    version = jboss.get_version()
    print(f"✓ JBoss {version} działa poprawnie")
```

### 8. Obsługa błędów i retry

```python
from jboss_ssh import JBossSSH
import time

def check_with_retry(hostname, retries=3, delay=5):
    """Sprawdź status JBoss z retry"""
    
    jboss = JBossSSH(hostname)
    
    for attempt in range(1, retries + 1):
        print(f"Próba {attempt}/{retries}...")
        
        if jboss.check_status():
            return True
        
        if attempt < retries:
            print(f"Czekam {delay}s przed następną próbą...")
            time.sleep(delay)
    
    return False

# Użycie
if check_with_retry('jboss01', retries=3):
    print("✓ JBoss działa")
else:
    print("✗ JBoss nie odpowiada po 3 próbach")
```

### 9. Różne porty CLI na różnych środowiskach

```python
from jboss_ssh import JBossSSH

environments = {
    'dev': {'host': 'jboss-dev', 'port': 9990},
    'test': {'host': 'jboss-test', 'port': 9990},
    'prod': {'host': 'jboss-prod', 'port': 19990}
}

for env_name, config in environments.items():
    print(f"\n=== Środowisko: {env_name} ===")
    
    jboss = JBossSSH(
        config['host'],
        cli_port=config['port']
    )
    
    if jboss.check_status():
        version = jboss.get_version()
        print(f"✓ {env_name}: JBoss {version}")
    else:
        print(f"✗ {env_name}: Błąd")
```

### 10. Zbieranie metryk do monitoringu

```python
from jboss_ssh import JBossSSH
import json
from datetime import datetime

def collect_jboss_metrics(hostname):
    """Zbiera metryki JBoss do systemu monitoringu"""
    
    jboss = JBossSSH(hostname)
    
    metrics = {
        'timestamp': datetime.now().isoformat(),
        'hostname': hostname,
        'status': 'unknown',
        'version': None,
        'deployments_count': 0
    }
    
    # Status serwera
    if jboss.check_status():
        metrics['status'] = 'running'
    else:
        metrics['status'] = 'down'
        return metrics
    
    # Wersja
    metrics['version'] = jboss.get_version()
    
    # Liczba deploymentów
    deployments = jboss.list_deployments()
    if deployments:
        # Policz linie z deployment info (uproszczone)
        metrics['deployments_count'] = deployments.count('NAME')
    
    return metrics

# Użycie
metrics = collect_jboss_metrics('jboss01')
print(json.dumps(metrics, indent=2))

# Wyślij do systemu monitoringu (Prometheus, Grafana, etc.)
# ...
```

---

## 🔧 Integracja z Jenkins

### Pipeline - podstawowy

```groovy
pipeline {
    agent {
        docker {
            image 'python:3.11-slim'
        }
    }
    
    parameters {
        string(name: 'JBOSS_HOST', defaultValue: 'jboss01', description: 'Hostname JBoss')
        string(name: 'CLI_PORT', defaultValue: '9990', description: 'Port JBoss CLI')
        string(name: 'SSH_USER', defaultValue: 'jenkins', description: 'Użytkownik SSH')
    }
    
    stages {
        stage('Prepare Script') {
            steps {
                script {
                    // Pobierz bibliotekę z repo lub użyj wbudowanej
                    sh 'curl -O https://raw.githubusercontent.com/your-repo/jboss_ssh.py || true'
                }
            }
        }
        
        stage('Check JBoss Status') {
            steps {
                sh """
                    python jboss_ssh.py ${params.JBOSS_HOST} ${params.CLI_PORT}
                """
            }
        }
    }
    
    post {
        success {
            echo '✓ JBoss działa poprawnie'
        }
        failure {
            echo '✗ Problem z JBoss'
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
            args '-v ~/.ssh:/root/.ssh:ro'
        }
    }
    
    parameters {
        text(
            name: 'JBOSS_HOSTS', 
            defaultValue: 'jboss01\njboss02\njboss03',
            description: 'Lista hostów JBoss (jeden na linię)'
        )
        string(name: 'CLI_PORT', defaultValue: '9990', description: 'Port JBoss CLI')
        choice(
            name: 'COMMAND',
            choices: ['status', 'version', 'deployments', 'datasources'],
            description: 'Polecenie do wykonania'
        )
        string(name: 'SSH_USER', defaultValue: 'jenkins', description: 'Użytkownik SSH')
    }
    
    stages {
        stage('Prepare') {
            steps {
                script {
                    writeFile file: 'jboss_ssh.py', text: libraryResource('jboss_ssh.py')
                    
                    writeFile file: 'check_all_hosts.py', text: """
from jboss_ssh import JBossSSH
import sys

hosts = '''${params.JBOSS_HOSTS}'''.strip().split('\\n')
cli_port = ${params.CLI_PORT}
ssh_user = '${params.SSH_USER}'
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
        jboss = JBossSSH(host, ssh_user=ssh_user, cli_port=cli_port)
        
        rc, out, err = jboss.execute_cli_command(jboss_command, timeout=30)
        
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
        text(name: 'HOSTS', defaultValue: 'jboss01\njboss02\njboss03', description: 'Lista hostów')
    }
    
    stages {
        stage('Verify Deployment') {
            steps {
                script {
                    writeFile file: 'verify_deployment.py', text: """
from jboss_ssh import JBossSSH
import sys

app_name = '${params.APP_NAME}'
expected_version = '${params.EXPECTED_VERSION}'
hosts = '''${params.HOSTS}'''.strip().split('\\n')

all_ok = True

print(f"\\n{'='*70}")
print(f"Weryfikacja deploymentu: {app_name}")
print(f"Oczekiwana wersja: {expected_version}")
print(f"{'='*70}\\n")

for host in hosts:
    host = host.strip()
    if not host:
        continue
    
    print(f"\\nSprawdzam {host}...")
    
    jboss = JBossSSH(host)
    
    # Sprawdź czy deployment istnieje i jest włączony
    if not jboss.get_deployment_status(app_name):
        print(f"✗ {host}: Deployment nie istnieje lub jest wyłączony")
        all_ok = False
        continue
    
    # Sprawdź wersję (jeśli jest w nazwie pliku)
    cmd = f'/deployment={app_name}:read-attribute(name=runtime-name)'
    rc, out, err = jboss.execute_cli_command(cmd)
    
    if rc == 0:
        if expected_version in out:
            print(f"✓ {host}: Wersja {expected_version} wdrożona poprawnie")
        else:
            print(f"✗ {host}: Nieprawidłowa wersja!")
            print(f"  Znaleziono: {out}")
            all_ok = False
    else:
        print(f"✗ {host}: Błąd sprawdzania: {err}")
        all_ok = False

print(f"\\n{'='*70}")
if all_ok:
    print("✓ Deployment zweryfikowany pomyślnie na wszystkich serwerach")
    sys.exit(0)
else:
    print("✗ Deployment nie jest poprawny na wszystkich serwerach")
    sys.exit(1)
"""
                    sh 'python verify_deployment.py'
                }
            }
        }
    }
    
    post {
        success {
            echo '✓ Weryfikacja deploymentu zakończona sukcesem'
        }
        failure {
            echo '✗ Problemy z deploymentem'
        }
    }
}
```

### Shared Library Jenkins

Możesz też stworzyć Jenkins Shared Library:

```groovy
// vars/jbossSSH.groovy
def checkStatus(String host, int port = 9990) {
    sh """
        python -c "
from jboss_ssh import JBossSSH
import sys
jboss = JBossSSH('${host}', cli_port=${port})
sys.exit(0 if jboss.check_status() else 1)
        "
    """
}

def executeCommand(String host, String command, int port = 9990) {
    return sh(
        script: """
            python -c "
from jboss_ssh import JBossSSH
jboss = JBossSSH('${host}', cli_port=${port})
rc, out, err = jboss.execute_cli_command('${command}')
print(out)
exit(rc)
            "
        """,
        returnStdout: true
    ).trim()
}

def getVersion(String host, int port = 9990) {
    return sh(
        script: """
            python -c "
from jboss_ssh import JBossSSH
jboss = JBossSSH('${host}', cli_port=${port})
version = jboss.get_version()
print(version if version else 'unknown')
            "
        """,
        returnStdout: true
    ).trim()
}
```

Użycie w Pipeline:

```groovy
@Library('shared-lib') _

pipeline {
    agent any
    stages {
        stage('Check JBoss') {
            steps {
                script {
                    jbossSSH.checkStatus('jboss01', 9990)
                    
                    def version = jbossSSH.getVersion('jboss01')
                    echo "Wersja JBoss: ${version}"
                }
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

# Skopiuj klucz na serwer JBoss
ssh-copy-id jenkins@jboss01
```

#### 2. Ogranicz dostęp SSH

W `~/.ssh/authorized_keys` na serwerze JBoss:

```bash
# Ogranicz do konkretnych komend (najbezpieczniejsze)
command="/opt/scripts/jboss-cli-wrapper.sh" ssh-ed25519 AAAA...

# Ogranicz do konkretnych IP
from="10.20.30.0/24" ssh-ed25519 AAAA...

# Kombinacja
command="/opt/scripts/jboss-cli-wrapper.sh",from="10.20.30.40" ssh-ed25519 AAAA...
```

Przykładowy wrapper (`/opt/scripts/jboss-cli-wrapper.sh`):
```bash
#!/bin/bash
# Pozwala tylko na wykonywanie jboss-cli.sh
case "$SSH_ORIGINAL_COMMAND" in
    "/opt/jboss/bin/jboss-cli.sh"*)
        $SSH_ORIGINAL_COMMAND
        ;;
    *)
        echo "Tylko komendy jboss-cli.sh są dozwolone"
        exit 1
        ;;
esac
```

#### 3. Konfiguracja SSH dla produkcji

Edytuj kod, usuń niebezpieczne opcje:

```python
ssh_cmd = [
    'ssh',
    '-o', 'ConnectTimeout=10',
    '-o', 'BatchMode=yes',
    # W PRODUKCJI:
    '-o', 'StrictHostKeyChecking=yes',  # ← WAŻNE!
    '-o', 'PasswordAuthentication=no',  # Wymuszaj klucze
    '-o', 'PubkeyAuthentication=yes',
    f'{self.ssh_user}@{self.remote_host}',
    # komenda...
]
```

#### 4. Zasada najmniejszych uprawnień

```bash
# Dedykowany użytkownik dla CLI
sudo useradd -m -s /bin/bash jboss-monitor
sudo usermod -G jboss jboss-monitor  # Dostęp do plików JBoss

# Generuj klucze dla tego użytkownika
sudo su - jboss-monitor
ssh-keygen -t ed25519

# Ogranicz sudo tylko do potrzebnych komend (jeśli konieczne)
# /etc/sudoers.d/jboss-monitor
jboss-monitor ALL=(jboss) NOPASSWD: /opt/jboss/bin/jboss-cli.sh
```

#### 5. Monitoring i audyt

```python
import logging
from datetime import datetime

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('/var/log/jboss-ssh.log'),
        logging.StreamHandler()
    ]
)

class JBossSSH:
    def execute_cli_command(self, command, timeout=60):
        logging.info(f"User: {self.ssh_user}, Host: {self.host}, Command: {command}")
        # ... reszta kodu
        logging.info(f"Result: rc={result.returncode}, duration={duration}s")
```

Monitoring nieautoryzowanych prób na serwerze:
```bash
# /etc/ssh/sshd_config
LogLevel VERBOSE

# Monitoruj logi
tail -f /var/log/auth.log | grep -i "failed\|error\|invalid"
```

#### 6. Secrets management

Nigdy nie hardcoduj credentials w kodzie:

```python
import os

# Pobieraj z zmiennych środowiskowych
SSH_USER = os.getenv('JBOSS_SSH_USER', 'jenkins')
SSH_KEY_PATH = os.getenv('SSH_KEY_PATH', '~/.ssh/id_ed25519')

# W Jenkins: withCredentials([sshUserPrivateKey(...)])
jboss = JBossSSH(
    host='jboss01',
    ssh_user=SSH_USER
)
```

W Jenkins Pipeline:
```groovy
pipeline {
    stages {
        stage('Check JBoss') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'jboss-ssh-key',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh """
                        export SSH_AUTH_SOCK=""
                        ssh-add ${SSH_KEY}
                        python jboss_ssh.py jboss01
                    """
                }
            }
        }
    }
}
```

#### 7. Network segmentation

```
┌──────────────────────────────────────────────────────────┐
│                     DMZ / Public                          │
│  ┌────────────┐                                          │
│  │  Jenkins   │  SSH (22) tylko do specific hosts        │
│  └─────┬──────┘                                          │
└────────┼──────────────────────────────────────────────────┘
         │ SSH (zaszyfrowane)
         │ - Autentykacja kluczami
         │ - Wszystkie komendy logowane
         │
┌────────▼──────────────────────────────────────────────────┐
│              Internal Network / Private                    │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐      │
│  │  jboss01   │    │  jboss02   │    │  jboss03   │      │
│  │  CLI:9990  │    │  CLI:9990  │    │  CLI:9990  │      │
│  └────────────┘    └────────────┘    └────────────┘      │
│  (CLI dostępny TYLKO lokalnie - 127.0.0.1)                │
└──────────────────────────────────────────────────────────┘
```

#### 8. Rate limiting i brute force protection

Na serwerze JBoss (`/etc/ssh/sshd_config`):
```bash
# Maksymalnie 3 próby logowania
MaxAuthTries 3

# Timeout dla nieuwierzytelnionych połączeń
LoginGraceTime 30

# Maksymalna liczba równoczesnych nieuwierzytelnionych połączeń
MaxStartups 3:50:10
```

Fail2ban dla dodatkowej ochrony:
```bash
# /etc/fail2ban/jail.local
[sshd]
enabled = true
maxretry = 3
bantime = 3600
```

### Checklist bezpieczeństwa

- [ ] Używasz kluczy SSH zamiast haseł
- [ ] `StrictHostKeyChecking` ustawione na `yes` w produkcji
- [ ] Klucze SSH mają odpowiednie uprawnienia (600)
- [ ] Używasz dedykowanego użytkownika dla dostępu do JBoss CLI
- [ ] Włączony logging wszystkich połączeń i komend
- [ ] Ograniczony dostęp przez `authorized_keys` (command=, from=)
- [ ] Secrets są w zmiennych środowiskowych lub vault, nie w kodzie
- [ ] Regularnie rotowane klucze SSH (np. co 90 dni)
- [ ] Monitoring nieautoryzowanych prób połączeń
- [ ] Rate limiting na serwerze SSH
- [ ] JBoss CLI port (9990) NIE jest dostępny publicznie
- [ ] Firewall przepuszcza tylko SSH (port 22)
- [ ] Backup kluczy SSH w bezpiecznym miejscu
- [ ] Dokumentacja procedur dla incydentów bezpieczeństwa

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

# 4. Sprawdź czy serwer SSH działa (na serwerze docelowym)
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

### Problem: "jboss-cli.sh: command not found"

```
✗ bash: /opt/jboss/bin/jboss-cli.sh: No such file or directory
```

**Rozwiązania:**

```bash
# 1. Sprawdź gdzie jest jboss-cli.sh na serwerze
ssh user@jboss01 'find / -name jboss-cli.sh 2>/dev/null'

# 2. Zaktualizuj ścieżkę w kodzie
jboss = JBossSSH('jboss01', cli_path='/prawidlowa/sciezka/jboss-cli.sh')

# 3. Sprawdź zmienne środowiskowe JBoss
ssh user@jboss01 'echo $JBOSS_HOME'
```

### Problem: "Controller localhost:9990 not found"

```
✗ The controller is not available at localhost:9990
```

**Rozwiązania:**

```bash
# 1. Sprawdź czy JBoss działa (na serwerze)
ssh user@jboss01 'ps aux | grep jboss'
ssh user@jboss01 'systemctl status jboss'

# 2. Sprawdź czy port CLI jest otwarty lokalnie (na serwerze)
ssh user@jboss01 'netstat -tlnp | grep 9990'
ssh user@jboss01 'ss -tlnp | grep 9990'

# 3. Sprawdź port w konfiguracji JBoss
ssh user@jboss01 'grep management-native /opt/jboss/standalone/configuration/standalone.xml'

# 4. Użyj właściwego portu
# Czasem może być 9990, 19990, lub inny
jboss = JBossSSH('jboss01', cli_port=19990)
```

### Problem: Timeout podczas wykonywania polecenia

```
✗ Timeout - polecenie przekroczyło limit 60s
```

**Rozwiązania:**

```python
# Zwiększ timeout
rc, out, err = jboss.execute_cli_command(command, timeout=300)  # 5 minut

# Lub sprawdź czy JBoss nie jest przeciążony
jboss.execute_cli_command(':read-attribute(name=server-state)', timeout=10)
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

# 3. UWAGA: W produkcji NIE używaj StrictHostKeyChecking=no!
# Zmień w kodzie na 'yes' i użyj known_hosts
```

### Problem: SSH działa, ale CLI zwraca błędy

```
✓ SSH: OK
✗ CLI: WBFL000019: The connection was closed as the management request did not complete within 5000 milliseconds
```

**Rozwiązania:**

```bash
# 1. Zwiększ timeout w JBoss CLI
# Edytuj jboss-cli.xml na serwerze
<default-connect-timeout>30000</default-connect-timeout>

# 2. Lub użyj krótkiego timeout w Pythonie
jboss.execute_cli_command(':read-attribute(name=server-state)', timeout=5)

# 3. Sprawdź obciążenie serwera JBoss
ssh user@jboss01 'top -bn1 | grep java'
```

### Problem: Brak odpowiedzi, ale brak błędów

```python
rc, out, err = jboss.execute_cli_command(':read-attribute(name=server-state)')
# rc = 0, ale out jest pusty
```

**Rozwiązania:**

```bash
# 1. Test ręczny
ssh user@jboss01 '/opt/jboss/bin/jboss-cli.sh --connect --controller=localhost:9990 --command=":read-attribute(name=server-state)"'

# 2. Sprawdź czy CLI wymaga uwierzytelnienia
# Możesz potrzebować dodać: --user=admin --password=admin

# 3. Sprawdź logi JBoss
ssh user@jboss01 'tail -100 /opt/jboss/standalone/log/server.log'
```

### Debug Mode

Włącz szczegółowy logging:

```python
from jboss_ssh import JBossSSH

jboss = JBossSSH('jboss01')

# Test połączenia SSH
if not jboss.test_connection():
    print("✗ SSH connection failed!")
    exit(1)

# Wykonaj komendę z verbose
command = ':read-attribute(name=server-state)'
print(f"Executing: {command}")

rc, out, err = jboss.execute_cli_command(command)

print(f"Return code: {rc}")
print(f"STDOUT:\n{out}")
print(f"STDERR:\n{err}")
```

### Często sprawdzane komendy

```bash
# Test połączenia SSH
ssh user@jboss01 'echo "SSH OK"'

# Test czy JBoss działa
ssh user@jboss01 'ps aux | grep "[j]boss"'

# Test CLI manualnie
ssh user@jboss01 '/opt/jboss/bin/jboss-cli.sh --connect --controller=localhost:9990 --command=":read-attribute(name=server-state)"'

# Sprawdź port CLI
ssh user@jboss01 'netstat -tlnp | grep 9990'

# Sprawdź logi JBoss
ssh user@jboss01 'tail -50 /opt/jboss/standalone/log/server.log'
```

### Weryfikacja konfiguracji

```python
from jboss_ssh import JBossSSH

def verify_setup(hostname):
    """Kompleksowa weryfikacja konfiguracji"""
    
    print(f"\n{'='*60}")
    print(f"Weryfikacja konfiguracji: {hostname}")
    print(f"{'='*60}\n")
    
    jboss = JBossSSH(hostname)
    
    # 1. Test SSH
    print("1. Test połączenia SSH...")
    if jboss.test_connection():
        print("   ✓ SSH działa")
    else:
        print("   ✗ SSH nie działa - sprawdź klucze i dostęp")
        return False
    
    # 2. Test JBoss CLI
    print("\n2. Test JBoss CLI...")
    if jboss.check_status():
        print("   ✓ JBoss CLI działa")
    else:
        print("   ✗ JBoss CLI nie działa - sprawdź ścieżkę i port")
        return False
    
    # 3. Pobierz dodatkowe info
    print("\n3. Informacje o JBoss:")
    version = jboss.get_version()
    print(f"   Wersja: {version}")
    
    print("\n✓ Wszystko działa poprawnie!")
    return True

# Użycie
verify_setup('jboss01')
```

---

## ❓ FAQ

### Q: Czy SSH do wykonywania komend jest bezpieczny?

**A:** Tak! SSH używa silnego szyfrowania (AES-256, ChaCha20) i jest standardem w branży. Wszystkie dane, w tym komendy i odpowiedzi JBoss CLI, są zaszyfrowane end-to-end.

### Q: Czy port 9990/19990 musi być otwarty w firewallu?

**A:** **NIE!** To główna zaleta tego podejścia. Port JBoss CLI pozostaje dostępny tylko lokalnie na serwerze. Jedyny otwarty port to SSH (22).

### Q: Gdzie musi być zainstalowany jboss-cli.sh?

**A:** Tylko na serwerze JBoss. Nie ma potrzeby instalowania go na kliencie (Jenkins), ponieważ CLI wykonuje się zdalnie przez SSH.

### Q: Jak to się ma do tuneli SSH?

**A:** To prostsze rozwiązanie:
- **Tunel SSH:** Tworzy lokalny port → przekierowuje przez SSH → uruchamia CLI lokalnie na kliencie
- **Bezpośrednie SSH:** Wysyła komendę przez SSH → uruchamia CLI zdalnie na serwerze → zwraca wynik

Bezpośrednie SSH jest:
- Prostsze (mniej kodu)
- Szybsze (mniej hopów)
- Bardziej niezawodne (mniej punktów awarii)

### Q: Co jeśli muszę wykonać wiele komend?

**A:** Możesz:

1. Wykonać każdą komendę osobno:
```python
jboss = JBossSSH('jboss01')
jboss.execute_cli_command('command1')
jboss.execute_cli_command('command2')
```

2. Lub użyć `execute_multiple_commands()`:
```python
commands = ['command1', 'command2', 'command3']
results = jboss.execute_multiple_commands(commands)
```

Każda komenda używa tego samego połączenia SSH (multiplexing).

### Q: Jak wpływa to na wydajność?

**A:** Minimalny narzut:
- Każde wywołanie: ~50-200ms (nawiązanie SSH + wykonanie CLI)
- Dla porównania tunel: ~20-50ms per komenda + czas setup tunelu (1-2s)
- Dla częstych wywołań (>10/s) różnica jest większa, ale w typowych scenariuszach (monitoring, deploymenty) jest niezauważalna

### Q: Co jeśli JBoss CLI wymaga uwierzytelnienia?

**A:** Dodaj credentials do komendy:

```python
command = ':read-attribute(name=server-state) --user=admin --password=secret'
rc, out, err = jboss.execute_cli_command(command)
```

Lepiej: skonfiguruj JBoss CLI auth na serwerze w `jboss-cli.xml`.

### Q: Czy mogę używać tego z kontenerami Docker?

**A:** Tak! Upewnij się, że:

1. SSH client jest zainstalowany w kontenerze:
   ```dockerfile
   RUN apt-get update && apt-get install -y openssh-client
   ```

2. Klucze SSH są dostępne:
   ```bash
   docker run -v ~/.ssh:/root/.ssh:ro myimage
   ```

3. Kontener ma dostęp sieciowy do hosta JBoss.

### Q: Jak obsłużyć różne porty CLI na różnych środowiskach?

**A:** Użyj konfiguracji lub zmiennych:

```python
# Z dict
envs = {
    'dev': {'host': 'jboss-dev', 'port': 9990},
    'prod': {'host': 'jboss-prod', 'port': 19990}
}

for env, config in envs.items():
    jboss = JBossSSH(config['host'], cli_port=config['port'])
    jboss.check_status()
```

### Q: Co jeśli muszę używać jump host / bastion?

**A:** SSH obsługuje ProxyJump:

```python
# Najpierw skonfiguruj w ~/.ssh/config
# Host jboss01
#     ProxyJump jumphost

# Potem normalnie:
jboss = JBossSSH('jboss01')
jboss.check_status()
```

Lub bezpośrednio w komendzie SSH (modyfikacja w kodzie):
```python
ssh_command = [
    'ssh',
    '-J', 'jumphost',  # ProxyJump
    f'{self.ssh_user}@{self.host}',
    # ... reszta
]
```

### Q: Jak testować lokalnie bez prawdziwego JBoss?

**A:** Możesz:

1. Testować samo SSH:
```python
jboss = JBossSSH('localhost')
if jboss.test_connection():
    print("SSH działa")
```

2. Mock w unit testach:
```python
from unittest.mock import patch, MagicMock

@patch('subprocess.run')
def test_check_status(mock_run):
    mock_run.return_value = MagicMock(
        returncode=0,
        stdout='{"result" => "running"}'
    )
    
    jboss = JBossSSH('testhost')
    assert jboss.check_status() == True
```

### Q: Co z logowaniem i audytem?

**A:** Dodaj logging:

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

class JBossSSH:
    def execute_cli_command(self, command, timeout=60):
        logging.info(f"Executing on {self.host}: {command}")
        # ... existing code
        logging.info(f"Result: rc={result.returncode}")
```

### Q: Czy mogę wykonać inne komendy przez SSH (nie tylko CLI)?

**A:** Tak! Możesz łatwo rozszerzyć:

```python
def execute_shell_command(self, command):
    """Wykonaj dowolną komendę shell na serwerze"""
    ssh_command = [
        'ssh',
        f'{self.ssh_user}@{self.host}',
        command
    ]
    result = subprocess.run(ssh_command, capture_output=True, text=True)
    return result.returncode, result.stdout, result.stderr

# Użycie
jboss = JBossSSH('jboss01')
rc, out, err = jboss.execute_shell_command('ps aux | grep jboss')
```

### Q: Jak zintegrować z systemem monitoringu (Prometheus, Grafana)?

**A:** Przykład:

```python
from prometheus_client import Gauge, start_http_server
from jboss_ssh import JBossSSH
import time

# Metryki Prometheus
jboss_status = Gauge('jboss_status', 'JBoss status', ['host'])
jboss_deployments = Gauge('jboss_deployments', 'Number of deployments', ['host'])

def collect_metrics():
    hosts = ['jboss01', 'jboss02', 'jboss03']
    
    for host in hosts:
        jboss = JBossSSH(host)
        
        # Status (1 = running, 0 = down)
        status = 1 if jboss.check_status() else 0
        jboss_status.labels(host=host).set(status)
        
        # Liczba deploymentów
        deployments = jboss.list_deployments()
        if deployments:
            count = deployments.count('NAME')
            jboss_deployments.labels(host=host).set(count)

# Start HTTP server dla Prometheus
start_http_server(8000)

# Zbieraj metryki co 60s
while True:
    collect_metrics()
    time.sleep(60)
```

### Q: Czy to działa na Windows?

**A:** Tak, jeśli masz SSH client:

1. Windows 10/11 - wbudowany OpenSSH
2. Git Bash - zawiera SSH
3. WSL - pełne SSH capabilities

Kod Python działa identycznie na wszystkich platformach.

---

## 📚 Dodatkowe zasoby

### Dokumentacja

- [OpenSSH Manual](https://www.openssh.com/manual.html)
- [JBoss CLI Guide](https://docs.jboss.org/author/display/WFLY/CLI+Recipes)
- [Python subprocess](https://docs.python.org/3/library/subprocess.html)
- [SSH Command Execution](https://www.ssh.com/academy/ssh/command)

### Bezpieczeństwo

- [SSH Best Practices](https://www.ssh.com/academy/ssh/security)
- [NIST SSH Guidelines](https://nvlpubs.nist.gov/nistpubs/ir/2015/NIST.IR.7966.pdf)
- [SSH Key Management](https://www.ssh.com/academy/ssh/keygen)

### JBoss / WildFly

- [JBoss CLI Documentation](https://docs.wildfly.org/26/Admin_Guide.html#Command_Line_Interface)
- [Management API Guide](https://docs.wildfly.org/26/Admin_Guide.html#Management_API_reference)

### Narzędzia

- [Ansible](https://www.ansible.com/) - Automation using SSH
- [Fabric](https://www.fabfile.org/) - Python library for SSH tasks
- [Paramiko](https://www.paramiko.org/) - Python SSH library (alternatywa)

---

## 📝 Changelog

### v2.0.0 (2025-10-31)

- ✨ Przepisane na bezpośrednie SSH zamiast tuneli
- 🚀 Prostsze API - klasa JBossSSH
- ⚡ Szybsze - mniej hopów sieciowych
- 🔒 Bezpieczniejsze - mniej punktów awarii
- 📊 Dodano `check_multiple_hosts()`
- 🧪 Dodano `test_connection()`
- 📝 Pełna dokumentacja i przykłady
- 🔧 Integracja z Jenkins Pipeline

---

## 📄 Licencja

MIT License - możesz swobodnie używać w projektach komercyjnych i open source.

---

## 👥 Autorzy

Utworzone dla projektów DevOps/SRE wymagających bezpiecznego dostępu do JBoss Management CLI bez potrzeby otwierania dodatkowych portów w firewallu.

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

**Happy JBoss Monitoring! 🚀**
