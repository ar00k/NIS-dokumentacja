# README.md

Ten plik README opisuje krok po kroku konfigurację serwera NIS, klientów NIS oraz udostępnianie katalogów przez NFS w sieci lokalnej. Instrukcje są przygotowane pod Ubuntu/Debian.

---

## Spis treści

1. [Wymagania wstępne](#wymagania-wstępne)
2. [Przygotowanie środowiska (opcjonalne)](#przygotowanie-środowiska-opcjonalne)
3. [Konfiguracja serwera NIS](#konfiguracja-serwera-nis)
4. [Konfiguracja klienta NIS](#konfiguracja-klienta-nis)
5. [Konfiguracja serwera NFS](#konfiguracja-serwera-nfs)
6. [Konfiguracja klienta NFS](#konfiguracja-klienta-nfs)
7. [Testowanie i weryfikacja](#testowanie-i-weryfikacja)
8. [Rozwiązywanie problemów](#rozwiązywanie-problemów)

---

## Wymagania wstępne

* Działający Ubuntu/Debian (serwer i klienci)
* Sieć lokalna 192.168.10.0/24
* Uprawnienia `sudo`
* VirtualBox Additions (jeśli testujemy w VM)

---

## Przygotowanie środowiska (opcjonalne)

> Te kroki są przydatne przy pracy w środowisku graficznym VirtualBox, ale **nie są niezbędne** do funkcjonalności NIS/NFS.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install ubuntu-desktop   # instalacja GUI (opcjonalnie)
sudo systemctl set-default graphical.target

# Montowanie płyty z dodatkami VirtualBox
sudo mount /dev/cdrom /mnt
cd /mnt
sudo ./VBoxLinuxAdditions.run

# W VirtualBox: włączyć dwukierunkowe przeciąganie i współdzielony schowek
```

---

## Konfiguracja serwera NIS

### 1. Ustawienie nazwy domeny NIS

```bash
sudo nisdomainname nisArek
```

Aby nazwa domeny była trwała po restarcie, w pliku `/etc/default/nis` ustaw:

```ini
# /etc/default/nis
domain=nisArek
YPPWDDIR=/etc
YPCHANGEOK=chsh
NISSERVER=master        # KLUCZOWE: serwer główny NIS
NISMASTER=nisArek        # nazwa serwera (opcjonalnie przy wielu)
YPSERVARGS=""
```

### 2. Konfiguracja zabezpieczeń serwera NIS

#### Plik `/etc/ypserv.securenets`

```text
# Zawsze zezwalaj na localhost (IPv4 i IPv6)
255.0.0.0       127.0.0.0
host            ::1

# Dostęp tylko dla podsieci 192.168.10.0/24
255.255.255.0   192.168.10.0

# Pozostałe połączenia są odrzucane domyślnie
```

#### Plik `/etc/yp.conf`

```text
# domena NIS i adres serwera
domain nisArek server 192.168.10.10
```

### 3. Uruchomienie usług NIS przy starcie systemu

```bash
sudo systemctl enable ypserv yppasswdd ypxfrd
sudo reboot
```

### 4. Inicjalizacja map NIS

Po restarcie wykonaj:

```bash
sudo /usr/lib/yp/ypinit -m
```

Gdy pojawi się prośba o nazwę serwera, naciśnij <kbd>Ctrl+D</kbd> (domyślnie użyje `nisArek`).

---

## Konfiguracja klienta NIS

Wykonaj na każdym węźle-kliencie (node1: 192.168.10.20, node2: 192.168.10.30)

1. Instalacja klienta NIS:

   ```bash
   sudo apt update && sudo apt install nis -y
   ```
2. Ustawienie domeny NIS:

   ```bash
   sudo nisdomainname nisArek
   echo "nisArek" | sudo tee /etc/defaultdomain
   ```
3. Plik `/etc/yp.conf`:

   ```text
   domain nisArek server 192.168.10.10
   ```
4. Modyfikacja `/etc/nsswitch.conf` — uwzględnienie `nis` w odpowiednich liniach:

   ```diff
   passwd:         files systemd nis sss
   group:          files systemd nis sss
   shadow:         files systemd nis sss
   gshadow:        files systemd nis sss

   hosts:          files mdns4_minimal [NOTFOUND=return] dns
   netgroup:       nis [SUCCESS=return] sss
   automount:      sss
   ```
5. Restart usługi (lub reboot):

   ```bash
   sudo systemctl restart nis.service
   ```

---

## Konfiguracja serwera NFS

> Wykonaj na serwerze NFS (ten sam host co NIS lub inny)

1. Instalacja i uruchomienie:

   ```bash
   sudo apt update
   sudo apt install nfs-kernel-server -y
   sudo systemctl enable --now nfs-server rpcbind
   ```
2. Dodanie użytkowników NFS:

   ```bash
   sudo adduser --shell /bin/bash --home /home/nisuser1 nisuser1
   sudo adduser --shell /bin/bash --home /home/nisuser2 nisuser2

   sudo groupadd nisusers
   sudo usermod -aG nisusers nisuser1
   sudo usermod -aG nisusers nisuser2
   ```
3. Przygotowanie eksportowanych katalogów:

   ```bash
   sudo mkdir -p /srv/public /srv/restricted
   sudo chmod 777 /srv/public
   sudo chmod 775 /srv/restricted
   sudo chown nobody:nogroup /srv/public
   sudo chown root:nisusers /srv/restricted
   ```
4. Konfiguracja `/etc/exports`:

   ```text
   # Katalog publiczny dla całej podsieci
   /srv/public    192.168.10.0/24(rw,sync,no_subtree_check,no_root_squash)

   # Katalog restricted: node1 tylko do odczytu, node2 do zapisu
   /srv/restricted 192.168.10.20(ro,sync)
                  192.168.10.30(rw,sync)
   ```
5. Zastosowanie i restart usług:

   ```bash
   sudo exportfs -arv
   sudo systemctl restart nfs-server
   ```

---

## Konfiguracja klienta NFS

Na każdym kliencie:

1. Instalacja:

   ```bash
   sudo apt install nfs-common -y
   ```

2. Utworzenie punktów montowania i wpis w `/etc/fstab`:

   ```text
   192.168.10.10:/srv/public     /mnt/public     nfs  defaults,_netdev 0 0
   192.168.10.10:/srv/restricted /mnt/restricted nfs  defaults,_netdev 0 0
   ```

3. Utworzenie katalogów i montowanie:

   ```bash
   sudo mkdir -p /mnt/public /mnt/restricted
   sudo mount -a
   ```

4. Weryfikacja kont z NIS:

   ```bash
   ypcat passwd    # powinny się pojawić nisuser1 i nisuser2
   ```

---

## Testowanie i weryfikacja

* **NIS**: `ypcat passwd`, sprawdź czy widzisz użytkowników z serwera.
* **NFS**:

  * Węzeł node1: spróbuj zapisać plik w `/mnt/restricted` → powinien być błąd (tylko odczyt).
  * Węzeł node2: zapisz plik w `/mnt/restricted` → powinna zadziałać operacja.
  * Sprawdź uprawnienia w `/mnt/public` → odczyt/zapis dla wszystkich.

---

## Rozwiązywanie problemów

* **Status usług NIS**:

  ```bash
  sudo systemctl status ypserv yppasswdd ypxfrd
  ```

* **Reinicjalizacja map NIS** (jeśli mapy się nie tworzą poprawnie):

  ```bash
  sudo rm -rf /var/yp/nisArek/*    # usuń stare mapy
  sudo /usr/lib/yp/ypinit -m
  ```

* **Sprawdzenie domeny NIS**:

  ```bash
  cat /etc/defaultdomain     # powinna zawierać 'nisArek'
  ```

* Upewnij się, że pliki konfiguracyjne nie zawierają niepotrzebnych komentarzy ani pustych `YPSERVARGS`.

---

*Powodzenia!*
