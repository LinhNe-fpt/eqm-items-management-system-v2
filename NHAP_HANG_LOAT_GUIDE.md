# 📋 HƯỚNG DẪN: NHẬP HÀNG LOẠT JIG

## 🎯 Tính Năng
Nhập nhiều mã JIG cùng lúc với 4 cách:
1. **📦 Select từ Bảng JIG** - Chọn JIG → Nhập Hàng Loạt
2. **📋 Dán Danh Sách** - Copy-paste mã từ Excel/Text
3. **🔍 Quét QR** - Quét QR code tuần tự (Enter để lưu)
4. **📁 Upload File** - Tải lên file CSV/TXT

## 🚀 Cách Sử Dụng

### **Cách 1: Select từ Bảng JIG (Nhanh nhất)**
```
1. Vào menu "Quản lý JIG/Tool" → "Danh mục JIG/Tool"
2. Checkbox chọn các JIG cần nhập
3. Nhấp button "📦 Nhập Hàng Loạt" (xuất hiện khi select)
4. Dữ liệu JIG sẽ tự động populate
5. Chỉnh cấu hình batch (Loại nhập, Ngày, v.v.)
6. Nhấp "💾 Nhập Vào CSDL"
```

### **Cách 2: Dán Danh Sách**
```
1. Dán mã vào text area (mỗi mã 1 dòng)
   JIG001
   JIG002
   JIG003
2. Nhấp "Kiểm Tra Mã"
3. Xem kết quả validate
4. Nhấp "Nhập Vào CSDL"
```

### **Cách 3: Quét QR**
```
1. Click vào input, quét QR code
2. Sau mỗi mã, nhấn ENTER
3. Mã sẽ thêm vào danh sách
4. Sau khi quét xong, nhấp "Kiểm Tra Mã"
5. Nhấp "Nhập Vào CSDL"
```

### **Cách 4: Upload File**
```
1. Nhấp vùng upload hoặc kéo file vào
2. Chọn file CSV/TXT (format: 1 mã/dòng)
3. Nhấp "Kiểm Tra Mã"
4. Nhấp "Nhập Vào CSDL"
```

## ⚙️ Cấu Hình Batch (Bên phải)
- **Loại Nhập**: Nhập Kho JIG / Nhập Line / Mượn
- **Loại Hàng**: OK / NG
- **Ngày Nhập**: Mặc định hôm nay
- **Người Chuyển**: Tên người
- **Bên Chuyển**: Tên bên (YS, EQM, etc.)
- **Ghi Chú**: Thêm ghi chú (tuỳ chọn)

## ✅ Kết Quả
- **Thành công**: Mã được lưu vào bảng JIG_NHAP
- **Cập nhật kho**: Tồn kho JIG được cập nhật tự động
- **Báo cáo**: Hiển thị số lượng thành công/lỗi

## 📊 API Được Sử Dụng

### POST `/api/jig-nhap-bulk/validate-codes`
Validate danh sách mã có tồn tại trong hệ thống
```json
{
  "codes": ["JIG001", "JIG002"]
}
```

### POST `/api/jig-nhap-bulk/import`
Nhập hàng loạt vào CSDL
```json
{
  "jigs": [
    {
      "Ma_so_quan_ly": "JIG001",
      "Ten_JIG_thuc_te": "Tên JIG",
      "So_luong": 1,
      "Loai_nhap": "nhap_kho_jig",
      "hang_type": "ok"
    }
  ],
  "batch_config": {
    "Nguoi_chuyen_hang": "Tên",
    "Ben_Chuyen": "YS"
  }
}
```

## 🎨 Giao Diện
- **Gradient**: Blue → Cyan (chuyên nghiệp)
- **Responsive**: Hỗ trợ mobile, tablet, desktop
- **Dark Mode**: Tự động theo hệ thống
- **Animations**: Smooth transitions, framer-motion

## 💡 Mẹo
1. **Select JIG nhanh**: Dùng "Chọn tất cả" ở bảng JIG
2. Download template file CSV nếu cần
3. Validate trước khi nhập để tránh lỗi
4. Quét QR nhanh chóng cho 50+ mã
5. Xem chi tiết lỗi nếu validate không thành công
6. Có thể nhập tiếp sau khi xong

## 🐛 Xử Lý Lỗi
- **Mã không tồn tại**: Kiểm tra spelling
- **Validate thất bại**: Refresh page, thử lại
- **Import thất bại**: Xem chi tiết lỗi ở kết quả
- **Auto-populate không hoạt động**: Reload browser

## 📝 Cấu Trúc File Upload (CSV/TXT)
```
Ma_JIG
JIG001
JIG002
JIG003
```

## ⚡ Performance
- Hỗ trợ tối đa 100 mã cùng lúc
- Validate nhanh < 2 giây
- Import nhanh < 5 giây
- Auto-populate từ bảng JIG: Tức thì

---
**Phiên bản**: v1.1 | **Cập nhật**: 2026-01-06
**Cải tiến**: Thêm auto-populate từ select bảng JIG

