# 🔥 Firewall Setup cho Network XAMPP

## 🚀 Bước 1: Mở PowerShell (Admin)

Nhấn **Win + X** → chọn **Windows PowerShell (Admin)**

---

## ✅ Bước 2: Chạy lệnh sau

```powershell
# Thêm rule cho port 3000 (Node.js Backend)
netsh advfirewall firewall add rule name="Node.js Port 3000" `
  dir=in action=allow protocol=tcp localport=3000 remoteip=LocalSubnet

# Thêm rule cho port 5173 (Vite Frontend)
netsh advfirewall firewall add rule name="Vite Port 5173" `
  dir=in action=allow protocol=tcp localport=5173 remoteip=LocalSubnet

# Thêm rule cho port 3001 (nếu dùng)
netsh advfirewall firewall add rule name="Node.js Port 3001" `
  dir=in action=allow protocol=tcp localport=3001 remoteip=LocalSubnet

# Thêm rule cho port 80 & 443 (Apache)
netsh advfirewall firewall add rule name="Apache HTTP" `
  dir=in action=allow protocol=tcp localport=80 remoteip=LocalSubnet

netsh advfirewall firewall add rule name="Apache HTTPS" `
  dir=in action=allow protocol=tcp localport=443 remoteip=LocalSubnet
```

---

## ✔️ Bước 3: Xác nhận quy tắc đã thêm

```powershell
# Xem tất cả quy tắc
netsh advfirewall firewall show rule name=all

# Hoặc lọc:
netsh advfirewall firewall show rule name="Node*"
netsh advfirewall firewall show rule name="Vite*"
netsh advfirewall firewall show rule name="Apache*"
```

---

## 🌐 Bước 4: Lấy IP máy server

```powershell
# Chạy lệnh:
ipconfig

# Tìm dòng "IPv4 Address: 192.168.x.x"
# Ví dụ: 192.168.1.100
```

---

## 💻 Bước 5: Truy cập từ máy khác (cùng WiFi)

### **Frontend (Vite):**
```
http://192.168.1.100:5173
```

### **Backend (Node.js):**
```
http://192.168.1.100:3000/api/test
```

### **Test API:**
```
http://192.168.1.100:3000/api/bao-cao/nhap-kho
```

---

## 🗑️ Xóa quy tắc (nếu cần)

```powershell
# Xóa rule Port 3000
netsh advfirewall firewall delete rule name="Node.js Port 3000"

# Xóa rule Port 5173
netsh advfirewall firewall delete rule name="Vite Port 5173"
```

---

## 📋 Kiểm tra Connection

### **Từ máy khác (PowerShell):**

```powershell
# Test ping
ping 192.168.1.100

# Test port 3000
Test-NetConnection -ComputerName 192.168.1.100 -Port 3000

# Test port 5173
Test-NetConnection -ComputerName 192.168.1.100 -Port 5173
```

### **Từ Chrome:**

```
F12 → Network → Kiểm tra latency
```

---

## ⚠️ Lưu ý

✅ **Quy tắc chỉ allow LocalSubnet** (cùng mạng LAN)  
✅ **Port 3000 & 5173 chỉ mở trên LAN, không thể truy cập từ Internet**  
✅ **Khỏi lo bảo mật** - firewall chỉ allow máy cùng mạng  

---

## 🔍 Troubleshoot

### **Không thể kết nối?**

1. **Kiểm tra IP đúng:**
   ```powershell
   ipconfig  # Xem IPv4 Address
   ```

2. **Kiểm tra firewall rules:**
   ```powershell
   netsh advfirewall firewall show rule name=all
   ```

3. **Kiểm tra Node.js chạy:**
   ```powershell
   netstat -an | find "3000"  # Nếu có output = đang chạy
   ```

4. **Restart services:**
   ```powershell
   # Stop Node
   Stop-Process -Name "node" -Force
   
   # Restart
   cd C:\Users\EQM\Inventory\A\backend
   node server.js
   ```

---

## 📱 Mobile (Android/iOS)

Cùng cách, nhập IP + Port trong browser:

```
http://192.168.1.100:5173
```

---

**✅ Xong! Giờ máy khác đã có thể truy cập app của bạn!**
