# 🎯 Latihan 1: Hello World MCP
## Step-by-Step Tutorial

> **Objective**: Buat endpoint n8n yang menerima request dan return response standar MCP.
> **Difficulty**: Beginner
> **Estimated Time**: 15-20 menit

---

## 📋 Apa yang Akan Kita Buat?

Kita akan membuat **MCP Endpoint** sederhana yang:
1. Menerima request POST dengan nama user
2. Memproses data
3. Mengembalikan response dalam format MCP standar

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Request    │────▶│    n8n       │────▶│   Response   │
│              │     │   Process    │     │              │
│ POST /hello  │     │              │     │  MCP Format  │
│ {"name":"X"} │     │              │     │              │
└──────────────┘     └──────────────┘     └──────────────┘
```

---

## 🔧 Prerequisites

Sebelum mulai, pastikan:
- [ ] n8n sudah terinstall dan berjalan
- [ ] Bisa akses n8n di browser (biasanya `http://localhost:5678`)
- [ ] Familiar dengan membuat workflow baru

---

## 📝 Step 1: Buat Workflow Baru

### 1.1 Buka n8n

Akses n8n di browser:
```
http://localhost:5678
```
atau URL n8n instance Anda.

### 1.2 Create New Workflow

1. Klik **"+ Add Workflow"** atau **"Create new workflow"**
2. Beri nama: **"Latihan 1 - Hello World MCP"**
3. Klik **Save** (Ctrl+S)

```
📁 Workflow: Latihan 1 - Hello World MCP
   Status: Not active (Draft)
```

---

## 📝 Step 2: Tambah Webhook Trigger

### 2.1 Add Node

1. Klik tombol **"+"** di canvas
2. Search: **"Webhook"**
3. Klik **"Webhook"** (bukan Respond to Webhook)

### 2.2 Configure Webhook

Isi parameter berikut:

| Parameter | Value | Keterangan |
|-----------|-------|------------|
| **HTTP Method** | POST | Karena kita kirim data |
| **Path** | `mcp-hello` | URL endpoint kita |
| **Response Mode** | Using 'Respond to Webhook' node | Kita kontrol response-nya |

```
⚙️ Webhook Configuration:
┌─────────────────────────────────┐
│ HTTP Method: [POST ▾]           │
│ Path: mcp-hello                 │
│ Response Mode: [Using 'Respond  │
│                to Webhook' ▾]   │
└─────────────────────────────────┘
```

### 2.3 Hasil

Setelah save, n8n akan generate 2 URL:
- **Test URL**: `http://localhost:5678/webhook-test/mcp-hello`
- **Production URL**: `http://localhost:5678/webhook/mcp-hello`

> ⚠️ **Penting**: Test URL hanya aktif saat workflow dalam mode "Test". Production URL aktif saat workflow di-activate.

---

## 📝 Step 3: Tambah Code Node (Processing)

### 3.1 Add Node

1. Klik **"+"** setelah Webhook node
2. Search: **"Code"**
3. Klik **"Code"**

### 3.2 Configure Code Node

**Node Name**: `Process Request`

**Code** (paste ini):

```javascript
// Step 1: Ambil data dari request
const inputData = $input.first().json;

// Step 2: Extract nama dari body (handling berbagai format)
// Request bisa datang sebagai:
// - { body: { name: "Ahmad" } }  → dari Postman/curl
// - { name: "Ahmad" }            → langsung
const body = inputData.body || inputData;
const name = body.name || "Guest";

// Step 3: Validasi - nama tidak boleh kosong
if (!name || name.trim() === "") {
  throw new Error("Name parameter is required");
}

// Step 4: Buat response dalam format MCP standar
const mcpResponse = {
  // Status keseluruhan
  success: true,
  
  // Data utama
  message: `Hello, ${name}! 👋 MCP is working perfectly.`,
  
  // Metadata (informasi tambahan)
  metadata: {
    tool: "mcp-hello",
    version: "1.0.0",
    processedAt: new Date().toISOString(),
    inputReceived: {
      name: name
    }
  },
  
  // Tips (untuk learning purpose)
  tips: [
    "Ini adalah response standar MCP",
    "Field 'success' menandakan apakah request berhasil",
    "Field 'metadata' berisi informasi tentang processing"
  ]
};

// Step 5: Return response
return [{
  json: mcpResponse
}];
```

### 3.3 Penjelasan Code

```
┌─────────────────────────────────────────────────────────────┐
│                    ALUR PROCESSING                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  INPUT                    PROCESS                OUTPUT     │
│  ─────                    ───────                ──────     │
│                                                             │
│  { "name": "Ahmad" }  →  Extract name    →  {               │
│                          Validate             success: true │
│                          Format MCP           message: "..."│
│                                               metadata: {}  │
│                                             }               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 📝 Step 4: Tambah Respond to Webhook

### 4.1 Add Node

1. Klik **"+"** setelah Code node
2. Search: **"Respond"**
3. Klik **"Respond to Webhook"**

### 4.2 Configure Response

| Parameter | Value |
|-----------|-------|
| **Respond With** | JSON |
| **Response Body** | `{{ $json }}` |

### 4.3 (Optional) Add Custom Headers

Klik **"Add Option"** → **"Response Headers"**

Add headers:
```
X-MCP-Version: 1.0
X-Powered-By: n8n-mcp-bridge
```

---

## 📝 Step 5: Connect All Nodes

Pastikan nodes terhubung dengan benar:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Webhook    │────▶│   Process    │────▶│   Respond    │
│   Trigger    │     │   Request    │     │  to Webhook  │
└──────────────┘     └──────────────┘     └──────────────┘
```

**Cara connect:**
1. Klik titik kecil di kanan node (output)
2. Drag ke titik kecil di kiri node berikutnya (input)

---

## 📝 Step 6: Test Workflow

### 6.1 Activate Test Mode

1. Klik **"Test workflow"** (atau tekan Ctrl+Shift+Enter)
2. n8n akan menunggu request masuk

```
🔄 Waiting for webhook call...
   Test URL: http://localhost:5678/webhook-test/mcp-hello
```

### 6.2 Kirim Test Request

Buka terminal/PowerShell baru dan jalankan:

**PowerShell:**
```powershell
$body = @{ name = "Ahmad" } | ConvertTo-Json
Invoke-RestMethod -Uri "http://localhost:5678/webhook-test/mcp-hello" -Method POST -Body $body -ContentType "application/json"
```

**CURL (jika ada):**
```bash
curl -X POST http://localhost:5678/webhook-test/mcp-hello \
  -H "Content-Type: application/json" \
  -d '{"name": "Ahmad"}'
```

**Postman:**
1. New Request → POST
2. URL: `http://localhost:5678/webhook-test/mcp-hello`
3. Body → raw → JSON
4. Content:
```json
{
  "name": "Ahmad"
}
```
5. Send!

### 6.3 Expected Response

```json
{
  "success": true,
  "message": "Hello, Ahmad! 👋 MCP is working perfectly.",
  "metadata": {
    "tool": "mcp-hello",
    "version": "1.0.0",
    "processedAt": "2026-01-16T13:15:00.000Z",
    "inputReceived": {
      "name": "Ahmad"
    }
  },
  "tips": [
    "Ini adalah response standar MCP",
    "Field 'success' menandakan apakah request berhasil",
    "Field 'metadata' berisi informasi tentang processing"
  ]
}
```

---

## 📝 Step 7: Activate untuk Production

Setelah test berhasil:

1. Klik toggle **"Active"** di pojok kanan atas
2. Status berubah dari Draft → Active
3. Sekarang Production URL aktif: `http://localhost:5678/webhook/mcp-hello`

```
✅ Workflow is now active!
   Production URL ready: http://localhost:5678/webhook/mcp-hello
```

---

## 🧪 Test Cases

Coba beberapa test case untuk memastikan workflow robust:

### Test Case 1: Normal Request
```json
Input:  { "name": "Ahmad" }
Output: { "success": true, "message": "Hello, Ahmad! 👋..." }
Status: ✅ PASS
```

### Test Case 2: Empty Name
```json
Input:  { "name": "" }
Output: Error - "Name parameter is required"
Status: ✅ Expected error
```

### Test Case 3: No Name Field
```json
Input:  { "email": "test@mail.com" }
Output: { "success": true, "message": "Hello, Guest! 👋..." }
Status: ✅ Uses default "Guest"
```

### Test Case 4: Indonesian Name
```json
Input:  { "name": "Budi Santoso" }
Output: { "success": true, "message": "Hello, Budi Santoso! 👋..." }
Status: ✅ PASS - handles spaces
```

---

## 📊 Workflow Diagram Final

```
┌─────────────────────────────────────────────────────────────────┐
│                LATIHAN 1: HELLO WORLD MCP                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐                                           │
│  │    WEBHOOK      │  POST /mcp-hello                          │
│  │    TRIGGER      │  Receives: { "name": "..." }              │
│  └────────┬────────┘                                           │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                           │
│  │     PROCESS     │  Extract name                             │
│  │     REQUEST     │  Validate input                           │
│  │     (Code)      │  Format MCP response                      │
│  └────────┬────────┘                                           │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                           │
│  │    RESPOND TO   │  Return JSON                              │
│  │     WEBHOOK     │  Headers: X-MCP-Version                   │
│  └─────────────────┘                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## ✅ Checklist Penyelesaian

Pastikan semua ini sudah tercapai:

- [ ] Webhook node ter-configure dengan benar (POST, mcp-hello)
- [ ] Code node memproses input dan format response
- [ ] Respond to Webhook mengirim response JSON
- [ ] Semua node terhubung
- [ ] Test dengan Postman/curl berhasil
- [ ] Response sesuai format MCP standar
- [ ] Workflow di-activate

---

## 🎓 Apa yang Sudah Dipelajari?

1. **Webhook sebagai Entry Point** — Cara membuat endpoint yang bisa menerima request dari luar
2. **Data Processing** — Cara extract dan validate input data
3. **MCP Response Format** — Struktur standar dengan success, data, dan metadata
4. **n8n Flow** — Cara menghubungkan nodes menjadi workflow

---

## 🚀 Challenge (Bonus)

Setelah selesai, coba modifikasi workflow:

1. **Add Timestamp Format Indonesia**
   - Ubah timestamp ke format: "16 Januari 2026, 20:00 WIB"

2. **Add Request Counter**
   - Hitung berapa kali endpoint dipanggil (hint: gunakan static data atau database)

3. **Add Input Validation**
   - Tolak nama yang mengandung angka
   - Tolak nama kurang dari 2 karakter

---

## 📁 Files

Template workflow (bisa diimport langsung): 
`exercises/Latihan1-HelloWorldMCP.json`

---

*Tutorial by AI Architect | Phase 1 - Latihan 1*
