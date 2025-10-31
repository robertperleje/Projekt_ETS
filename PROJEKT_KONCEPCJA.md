# Projekt: Jenkins Pipeline - Restart Serwerów JBoss z integracją F5

**Data utworzenia:** 2025-10-31  
**Autor:** Robert Perleje  
**Asystent:** Claude (Anthropic)

---

## 📋 Spis treści

1. [Cel projektu](#cel-projektu)
2. [Parametry Jenkins Pipeline](#parametry-jenkins-pipeline)
3. [Mapowania](#mapowania)
4. [Scenariusze](#scenariusze)
5. [Szczegółowe kroki (Stages)](#szczegółowe-kroki-stages)
6. [Technologie](#technologie)

---

## 🎯 Cel projektu

Stworzenie Jenkins Job (Pipeline), który automatyzuje proces restartowania serwerów JBoss dla wybranych obszarów BOS z opcjonalnym wypięciem z load balancera F5.

**Główne funkcje:**
- Restart wybranego obszaru BOS (np. BOS Kernel, BOS Prowizje)
- Wybór serwera/serwerów do restartu (Server 1, Server 2, lub oba)
- Opcjonalne wypięcie z F5 podczas restartu (graceful shutdown)
- Weryfikacja poprawności restartu
- Rolling restart (jeden serwer po drugim przy wyborze obu)

---

## ⚙️ Parametry Jenkins Pipeline

| # | Nazwa parametru | Typ | Opis | Wartości |
|---|-----------------|-----|------|----------|
| 1 | **Obszar BOS** | Choice | Wybór obszaru do restartu | - BOS Kernel<br>- BOS Prowizje<br>- BOS Produkty<br>- BOS Roszczenia<br>- BOS Rezerwy<br>- BOS Sieć sprzedaży<br>- BOS Polisy<br>- BOS.Szkody |
| 2 | **Server 1** | Boolean (checkbox) | Czy restartować Server 1 | true/false |
| 3 | **Server 2** | Boolean (checkbox) | Czy restartować Server 2 | true/false |
| 4 | **Wypięcie z F5** | Boolean (checkbox) | Czy wypiąć z F5 przed restartem | true/false |

---

## 🗺️ Mapowania

### 1. Obszary BOS → Kody → Serwery JBoss

| Kod obszaru | Skrót | Nazwa obszaru | Serwery JBoss |
|-------------|-------|---------------|---------------|
| **BOS.KRN** | KR | BOS Kernel | jboss 01 i 02 |
| **BOS.PRO** | CS | BOS Prowizje | jboss 01 i 02 |
| **BOS.PTY** | PM | BOS Produkty | jboss 01 i 02 |
| **BOS.ORO** | CM | BOS Roszczenia | jboss 01 i 02 |
| **BOS.REZ** | RS | BOS Rezerwy | jboss 01 i 02 |
| **BOS.OSS** | SN | BOS Sieć sprzedaży | jboss 01 i 02 |
| **BOS.EPO** | PR | BOS Polisy | jboss 03 i 04 |
| **BOS.OSZ** | CL | BOS.Szkody | jboss 03 i 04 |

**Mapping fizyczny:**
- **jboss 01** = 10.26.166.25
- **jboss 02** = 10.26.166.26
- **jboss 03** = 10.26.166.53
- **jboss 04** = 10.26.166.54

### 2. Obszary BOS → Pule F5 → IP:Port

| Obszar | Pula F5 | Server 1 (IP:Port) | Server 2 (IP:Port) |
|--------|---------|-------------------|-------------------|
| **BOS.Szkody** | prod-bosp-jboss-8585-pool | 10.26.166.53:8585 | 10.26.166.54:8585 |
| **BOS Roszczenia** | prod-bosp-jboss-8582-pool | 10.26.166.25:8582 | 10.26.166.26:8582 |
| **BOS Prowizje** | prod-bosp-jboss-8586-pool | 10.26.166.25:8586 | 10.26.166.26:8586 |
| **BOS Kernel** | prod-bosp-jboss-8588-pool | 10.26.166.25:8588 | 10.26.166.26:8588 |
| **BOS Produkty** | prod-bosp-jboss-8581-pool | 10.26.166.25:8581 | 10.26.166.26:8581 |
| **BOS Polisy** | prod-bosp-jboss-8584-pool | 10.26.166.53:8584 | 10.26.166.54:8584 |
| **BOS Rezerwy** | prod-bosp-jboss-8587-pool | 10.26.166.25:8587 | 10.26.166.26:8587 |
| **BOS Sieć sprzedaży** | prod-bosp-jboss-8583-pool | 10.26.166.25:8583 | 10.26.166.26:8583 |

### 3. Server Groups → Serwery JBoss (EAP)

| Server Group | Server 1 | Server 2 | Obszar |
|--------------|----------|----------|---------|
| **prod-CM-grp-8582** | prod-CM-srv-8582-01 | prod-CM-srv-8582-02 | BOS Roszczenia |
| **prod-CS-grp-8586** | prod-CS-srv-8586-01 | prod-CS-srv-8586-02 | BOS Prowizje |
| **prod-KR-grp-8588** | prod-KR-srv-8588-01 | prod-KR-srv-8588-02 | BOS Kernel |
| **prod-PM-grp-8581** | prod-PM-srv-8581-01 | prod-PM-srv-8581-02 | BOS Produkty |
| **prod-PR-grp-8584** | prod-PR-srv-8584-03 | prod-PR-srv-8584-04 | BOS Polisy |
| **prod-RS-grp-8587** | prod-RS-srv-8587-01 | prod-RS-srv-8587-02 | BOS Rezerwy |
| **prod-SN-grp-8583** | prod-SN-srv-8583-01 | prod-SN-srv-8583-02 | BOS Sieć sprzedaży |
| **prod-CL-grp-8585** | prod-CL-srv-8585-03 | prod-CL-srv-8585-04 | BOS.Szkody |

**Notacja serwerów:**
- `-01` / `-02` = jboss 01/02 (10.26.166.25/26)
- `-03` / `-04` = jboss 03/04 (10.26.166.53/54)

---

## 🔄 Scenariusze

### Scenariusz 1: Server 1 ✓, Server 2 ✓, Wypięcie z F5 ✓
**Pełny rolling restart z F5 - oba serwery**

**Stages:**
1. Wypięcie Server 1 z F5 (disable)
2. Weryfikacja połączeń F5 dla Server 1 (Current ≤ 2)
3. Restart JBoss Server 1
4. Weryfikacja restartu Server 1 (status: running/started)
5. Wpięcie Server 1 do F5 (enable)
6. Wypięcie Server 2 z F5 (disable)
7. Weryfikacja połączeń F5 dla Server 2 (Current ≤ 2)
8. Restart JBoss Server 2
9. Weryfikacja restartu Server 2 (status: running/started)
10. Wpięcie Server 2 do F5 (enable)

---

### Scenariusz 2: Tylko Server 1 ✓, Server 2 ✗, Wypięcie z F5 ✓
**Restart tylko Server 1 z F5**

**Stages:**
1. Wypięcie Server 1 z F5 (disable)
2. Weryfikacja połączeń F5 dla Server 1 (Current ≤ 2)
3. Restart JBoss Server 1
4. Weryfikacja restartu Server 1 (status: running/started)
5. Wpięcie Server 1 do F5 (enable)

---

### Scenariusz 3: Server 1 ✗, Tylko Server 2 ✓, Wypięcie z F5 ✓
**Restart tylko Server 2 z F5**

**Stages:**
1. Wypięcie Server 2 z F5 (disable)
2. Weryfikacja połączeń F5 dla Server 2 (Current ≤ 2)
3. Restart JBoss Server 2
4. Weryfikacja restartu Server 2 (status: running/started)
5. Wpięcie Server 2 do F5 (enable)

---

### Scenariusz 4: Tylko Server 1 ✓, Server 2 ✗, Wypięcie z F5 ✗
**Restart tylko Server 1 bez F5**

**Stages:**
1. Restart JBoss Server 1
2. Weryfikacja restartu Server 1 (status: running/started)

---

### Scenariusz 5: Server 1 ✗, Tylko Server 2 ✓, Wypięcie z F5 ✗
**Restart tylko Server 2 bez F5**

**Stages:**
1. Restart JBoss Server 2
2. Weryfikacja restartu Server 2 (status: running/started)

---

### Scenariusz 6: Server 1 ✓, Server 2 ✓, Wypięcie z F5 ✗
**Rolling restart bez F5 - oba serwery**

**Stages:**
1. Restart JBoss Server 1
2. Weryfikacja restartu Server 1 (status: running/started)
3. Restart JBoss Server 2
4. Weryfikacja restartu Server 2 (status: running/started)

---

## 📝 Szczegółowe kroki (Stages)

### Przykład dla obszaru: **BOS Sieć sprzedaży**

#### Stage 1: Wypięcie Server 1 z F5

**Lokalizacja F5:**
- Path: `Local Traffic → Pools → Pool List → prod-bosp-jboss-8583-pool`
- Member: `10.26.166.25:8583`

**Akcja:**
- **Disable** member (graceful shutdown)
- Operacja przez F5 REST API lub CLI (tmsh)

**Endpoint REST API:**
```
PATCH /mgmt/tm/ltm/pool/prod-bosp-jboss-8583-pool/members/10.26.166.25:8583
Body: {"session": "user-disabled"}
```

---

#### Stage 2: Weryfikacja połączeń F5 dla Server 1

**Lokalizacja F5:**
- Path: `Statistics → Module Statistics: Local Traffic → Pools`
- Pool: `prod-bosp-jboss-8583-pool`
- Member: `10.26.166.25:8583`

**Warunek:**
- Monitorowanie kolumny **"Current Connections"**
- Czekanie aż: **Current ≤ 2**
- Polling co 5-10 sekund

**Endpoint REST API:**
```
GET /mgmt/tm/ltm/pool/prod-bosp-jboss-8583-pool/members/10.26.166.25:8583/stats
```

**Timeout:** maksymalnie 5 minut (konfigurowalne)

---

#### Stage 3: Restart JBoss Server 1

**Lokalizacja JBoss:**
- Server Group: `prod-SN-grp-8583`
- Server: `prod-SN-srv-8583-01`
- Host: `jboss01` (10.26.166.25)

**Akcja:**
- Restart serwera przez JBoss CLI lub REST API

**CLI (jboss-cli.sh):**
```bash
/host=jboss01/server-config=prod-SN-srv-8583-01:restart(blocking=true)
```

**REST API:**
```
POST /management/host/jboss01/server-config/prod-SN-srv-8583-01?operation=restart
```

---

#### Stage 4: Weryfikacja restartu Server 1

**Warunek:**
- Sprawdzenie statusu serwera: `running` lub `started`
- Polling co 10-15 sekund

**CLI:**
```bash
/host=jboss01/server-config=prod-SN-srv-8583-01:read-attribute(name=status)
```

**REST API:**
```
GET /management/host/jboss01/server-config/prod-SN-srv-8583-01?operation=attribute&name=status
```

**Timeout:** maksymalnie 10 minut (konfigurowalne)

---

#### Stage 5: Wpięcie Server 1 do F5

**Lokalizacja F5:**
- Path: `Local Traffic → Pools → Pool List → prod-bosp-jboss-8583-pool`
- Member: `10.26.166.25:8583`

**Akcja:**
- **Enable** member

**Endpoint REST API:**
```
PATCH /mgmt/tm/ltm/pool/prod-bosp-jboss-8583-pool/members/10.26.166.25:8583
Body: {"session": "user-enabled"}
```

---

## 🛠️ Technologie

### Jenkins Pipeline
- **Typ:** Declarative Pipeline (Jenkinsfile)
- **Parametry:** Choice, Boolean
- **Stages:** Dynamiczne w zależności od parametrów

### F5 BIG-IP
- **API:** iControl REST API
- **Endpoint:** `https://<F5_IP>/mgmt/tm/ltm/pool/`
- **Autentykacja:** Basic Auth lub Token
- **Operacje:** 
  - Disable member: `{"session": "user-disabled"}`
  - Enable member: `{"session": "user-enabled"}`
  - Stats: `/members/<IP:PORT>/stats`

### JBoss EAP (Red Hat)
- **API:** JBoss Management REST API / CLI
- **Endpoint:** `http://<JBOSS_IP>:9990/management`
- **CLI:** `jboss-cli.sh`
- **Operacje:**
  - Restart: `/host=<HOST>/server-config=<SERVER>:restart`
  - Status: `/host=<HOST>/server-config=<SERVER>:read-attribute(name=status)`

### Python (opcjonalnie)
- Biblioteki: `requests`, `time`
- Skrypty pomocnicze do interakcji z F5/JBoss API

---

## 📊 Przepływ danych

```
Parametry Jenkins
    ↓
[Mapowanie obszar → pula F5 + server group JBoss]
    ↓
Scenariusz (if Server1 && F5 → pełny flow)
    ↓
Stages wykonywane sekwencyjnie
    ↓
Logi + Status (SUCCESS/FAILURE)
```

---

## 🔐 Wymagania

### Dostępy:
- [ ] Credentials do F5 (REST API)
- [ ] Credentials do JBoss (Management API)
- [ ] Jenkins z zainstalowanymi pluginami:
  - Pipeline
  - HTTP Request Plugin
  - Credentials Plugin

### Konfiguracja:
- [ ] F5 Management IP + Port (443)
- [ ] JBoss Management IP + Port (9990)
- [ ] Timeout'y (konfigurowalne w pipeline)
- [ ] Retry policy dla API calls

---

## 📈 Metryki do monitorowania

- Czas trwania całego procesu
- Czas oczekiwania na spadek połączeń F5
- Czas restartu JBoss
- Liczba nieudanych prób (retry count)
- Status końcowy (SUCCESS/FAILURE/UNSTABLE)

---

## 🚀 Następne kroki

1. **Implementacja Jenkinsfile** (w repo: `Jenkinsfile`)
2. **Skrypty pomocnicze:**
   - `scripts/f5_operations.py` - obsługa F5 API
   - `scripts/jboss_operations.py` - obsługa JBoss API
3. **Testy:**
   - Dry-run mode (bez faktycznych restartów)
   - Test na środowisku deweloperskim
4. **Dokumentacja użytkownika** (jak uruchomić job)

---

## 📞 Kontakt

**Projekt:** Projekt_ETS  
**Repository:** https://github.com/robertperleje/Projekt_ETS  
**Twórca:** Robert Perleje  

---

**Wersja dokumentu:** 1.0  
**Ostatnia aktualizacja:** 2025-10-31
