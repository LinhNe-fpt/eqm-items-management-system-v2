# Backend - Nhập Hàng Loạt (Bulk Import)

## 📋 Tổng Quan

Backend API cho tính năng nhập hàng loạt nhiều mã JIG cùng một lúc với đầy đủ thông tin.

**File:** `/A/backend/dieu_hanh/jig_nhap_bulk.js`

---

## 🔧 Endpoints

### 1. POST `/api/jig-nhap-bulk/import`

**Nhập hàng loạt và lưu vào database**

#### Request Body:
```json
{
  "jigs": [
    {
      "Ma_so_quan_ly": "A013_01",
      "Ten_JIG_thuc_te": "A013_BATTERY_PRESS_JIG",
      "Code_final": "Z0000008-647957",
      "Ma_Model": "A013",
      "So_luong": 2,
      "hang_type": "ok",
      "vi_tri_kho": null
    }
  ],
  "batch_config": {
    "Loai_nhap": "nhap_kho_jig",
    "So_CRO": "CRO-2024-001",
    "Nguoi_chuyen_hang": "John Doe",
    "Ben_Chuyen": "Supplier A",
    "Ben_nhan": "You Sung",
    "Ngay_chuyen": "2024-01-06",
    "Trang_thai_CF_PSM": "OK",
    "So_RQ": "RQ-123",
    "Model": "A013",
    "Ghi_chu": "Import từ bulk"
  }
}
```

#### Response (Success):
```json
{
  "success": [
    {
      "index": 0,
      "Ma_so_quan_ly": "A013_01",
      "Ten_JIG_thuc_te": "A013_BATTERY_PRESS_JIG",
      "So_luong": 2,
      "message": "Nhập thành công"
    }
  ],
  "failed": [],
  "summary": {
    "total": 1,
    "success_count": 1,
    "failed_count": 0,
    "timestamp": "2024-01-06T...",
    "phieu_id": 123
  }
}
```

#### Response (Partial Failure):
```json
{
  "success": [...],
  "failed": [
    {
      "index": 1,
      "Ma_so_quan_ly": "A013_02",
      "error": "Mã JIG không tồn tại"
    }
  ],
  "summary": {...}
}
```

---

### 2. POST `/api/jig-nhap-bulk/validate-codes`

**Validate danh sách mã JIG trước khi import**

#### Request Body:
```json
{
  "codes": ["A013_01", "A013_02", "INVALID_CODE"]
}
```

#### Response:
```json
{
  "valid": [
    {
      "Ma_so_quan_ly": "A013_01",
      "Ten_JIG_thuc_te": "A013_BATTERY_PRESS_JIG",
      "Code_final": "Z0000008-647957",
      "Ma_Model": "A013"
    }
  ],
  "invalid": [
    {
      "code": "INVALID_CODE",
      "error": "Mã JIG không tồn tại"
    }
  ]
}
```

---

## 💾 Database Operations

### Bảng được cập nhật:

#### 1. **JIG_NHAP_PHIEU** (Phiếu nhập chính)
- **Khi nào:** Tạo 1 lần khi import
- **Lưu thông tin:**
  - `Loai_nhap`, `So_CRO`, `Nguoi_chuyen_hang`
  - `Ben_Chuyen`, `Ben_nhan`, `Ngay_chuyen`, `Gio_chuyen`
  - `Trang_thai_CF_PSM`, `So_RQ`, `Ghi_chu`
  - `Ten_nguoi_nhap`, `Ngay_nhap`, `Gio_nhap`
- **Output:** Trả về `Phieu_JD` (ID phiếu)

#### 2. **JIG_NHAP_PHIEU_LINE** (Chi tiết từng dòng nhập)
- **Khi nào:** Insert 1 record cho mỗi JIG trong danh sách
- **Lưu thông tin:**
  - `Phieu_JD` (ID phiếu cha)
  - `Ma_so_quan_ly`, `Ten_JIG_thuc_te`, `Code_final`
  - `Loai_nhap`, `So_CRO`, `Nguoi_chuyen_hang`, `Ngay_chuyen`
  - `Model`, `Tong_so_luong`, `Ton_line`
  - `hang_ok_ng` (OK/NG), `Trang_thai_CF_PSM`, `So_RQ`
  - `Ghi_chu`, `vi_tri_kho`

#### 3. **JIG_VI_TRI** (Vị trí tồn kho) - Nếu nhập line
- **Khi nào:** Cập nhật nếu `Loai_nhap === 'nhap_line'`
- **Cập nhật cột:** `OK` hoặc `NG` (tùy theo `hang_type`)
- **Công thức:** `Cột = Cột + So_luong`

#### 4. **JIG_MASTER** (Danh sách chủ JIG) - Nếu nhập line
- **Khi nào:** Cập nhật nếu `Loai_nhap === 'nhap_line'`
- **Cập nhật cột:**
  - `Ton_line`: `Ton_line + So_luong`
  - `Tong_so_luong`: `Tong_so_luong + So_luong`

---

## 🔄 Transaction & Error Handling

### Transaction Flow:
1. ✅ **Validate** tất cả JIG trước (kiểm tra tồn tại)
2. ✅ **Tạo phiếu** nhập chính (JIG_NHAP_PHIEU)
3. ✅ **Loop** từng JIG:
   - Insert vào JIG_NHAP_PHIEU_LINE
   - Update JIG_VI_TRI (nếu nhập line)
   - Update JIG_MASTER (nếu nhập line)
4. ✅ **Commit** nếu tất cả thành công

### Error Handling:
- ❌ **Tất cả lỗi?** → `ROLLBACK` (không lưu gì)
- ⚠️ **Một phần lỗi?** → `COMMIT` (lưu các mã thành công, báo lỗi chi tiết)
- ❌ **So_CRO missing?** → Error ngay (bắt buộc)

---

## ✅ Validation Rules

| Field | Required | Validation |
|-------|----------|-----------|
| So_CRO | ✅ Yes | Không được rỗng |
| Ma_so_quan_ly | ✅ Yes | Phải tồn tại trong JIG_MASTER |
| Ten_JIG_thuc_te | ✅ Yes | Không được rỗng |
| So_luong | ❌ No | Default = 1, Min = 1, Max = 9999 |
| Loai_nhap | ❌ No | Default = 'nhap_kho_jig' |
| hang_type | ❌ No | Default = 'ok' ('ok' hoặc 'ng') |

---

## 📊 Ví dụ thực tế

### Scenario: Import 3 JIG, 1 cái lỗi

**Request:**
```json
{
  "jigs": [
    {"Ma_so_quan_ly": "A013_01", "Ten_JIG_thuc_te": "JIG A", "Code_final": "CODE1", "So_luong": 2},
    {"Ma_so_quan_ly": "A013_02", "Ten_JIG_thuc_te": "JIG B", "Code_final": "CODE2", "So_luong": 1},
    {"Ma_so_quan_ly": "INVALID", "Ten_JIG_thuc_te": "Bad JIG", "Code_final": "CODE3", "So_luong": 1}
  ],
  "batch_config": {
    "So_CRO": "CRO-001",
    "Loai_nhap": "nhap_line",
    "Nguoi_chuyen_hang": "User1"
  }
}
```

**Response:**
```json
{
  "success": [
    {"index": 0, "Ma_so_quan_ly": "A013_01", "So_luong": 2, "message": "Nhập thành công"},
    {"index": 1, "Ma_so_quan_ly": "A013_02", "So_luong": 1, "message": "Nhập thành công"}
  ],
  "failed": [
    {"index": 2, "Ma_so_quan_ly": "INVALID", "error": "Mã JIG không tồn tại"}
  ],
  "summary": {
    "total": 3,
    "success_count": 2,
    "failed_count": 1,
    "phieu_id": 123
  }
}
```

**Database Result:**
- ✅ JIG_NHAP_PHIEU: 1 record với ID = 123
- ✅ JIG_NHAP_PHIEU_LINE: 2 records (A013_01 & A013_02)
- ✅ JIG_VI_TRI: Cập nhật OK += 3 (2+1)
- ✅ JIG_MASTER: Ton_line += 3, Tong_so_luong += 3

---

## 🚀 Integration

**File:** `A/backend/server.js`

```javascript
import jigNhapBulkRouter from './dieu_hanh/jig_nhap_bulk.js';
// ...
app.use('/api/jig-nhap-bulk', jigNhapBulkRouter);
```

**Frontend** gọi via `getApiUrl('/api/jig-nhap-bulk/import')`

---

## 📝 Logs & Monitoring

- **Success**: Chi tiết từng JIG thành công (Ma_so_quan_ly, So_luong)
- **Failed**: Chi tiết lỗi từng JIG (error message)
- **Timestamp**: Mỗi response có timestamp
- **Phieu_ID**: Trả về ID phiếu để track

---

## ⚠️ Lưu ý quan trọng

1. **So_CRO bắt buộc** - Không có sẽ error
2. **Transaction safety** - Nếu validation fail, không insert gì
3. **Partial success** - Commit dù có lỗi, để không mất dữ liệu tốt
4. **Tối đa 100 JIG/batch** - Giới hạn hiệu suất
5. **Ton_line update** - Chỉ update khi `Loai_nhap === 'nhap_line'`
6. **hang_type mapping** - 'ok' → OK column, 'ng' → NG column

---

**Version:** 1.0  
**Last Updated:** 2024-01-06
