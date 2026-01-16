# 🔌 Phase 1 Deep Dive: MCP Mastery
## The Connectivity Layer — Panduan Lengkap

> 📺 **Video Reference**: [Programmer Zaman Now - MCP](https://youtu.be/xNFeC4bJ4FY) [13:32]
> *"MCP adalah solusi agar LLM (Gemini, Claude, GPT) punya bahasa yang sama untuk bicara dengan aplikasi."*

---

## 📚 Daftar Isi

1. [Apa itu MCP? (Konsep Dasar)](#-apa-itu-mcp)
2. [Masalah yang Dipecahkan MCP](#-masalah-yang-dipecahkan-mcp)
3. [Arsitektur MCP](#-arsitektur-mcp)
4. [Komponen MCP Dijelaskan](#-komponen-mcp-dijelaskan)
5. [Tool-Aware Prompting](#-tool-aware-prompting)
6. [Implementasi Praktis dengan n8n](#-implementasi-praktis-dengan-n8n)
7. [Custom MCP Server untuk Legacy System](#-custom-mcp-server-untuk-legacy-system)
8. [Latihan Praktis](#-latihan-praktis)
9. [Troubleshooting & FAQ](#-troubleshooting--faq)

---

## 🤔 Apa itu MCP?

### Definisi Sederhana

**Model Context Protocol (MCP)** adalah standar komunikasi yang memungkinkan AI agents (seperti Claude, Gemini, GPT) untuk "berbicara" dengan aplikasi eksternal menggunakan bahasa yang sama.

### Analogi: USB-C untuk AI

```
SEBELUM USB-C (Kabel Berbeda untuk Setiap Device):
┌────────┐     ┌─────────────┐
│ iPhone │────▶│ Lightning   │
└────────┘     └─────────────┘
┌────────┐     ┌─────────────┐
│ Android│────▶│ Micro USB   │
└────────┘     └─────────────┘
┌────────┐     ┌─────────────┐
│ Laptop │────▶│ Barrel Jack │
└────────┘     └─────────────┘

SESUDAH USB-C (Satu Kabel untuk Semua):
┌────────┐     ┌─────────────┐
│ iPhone │────▶│             │
└────────┘     │             │
┌────────┐     │   USB-C     │
│ Android│────▶│             │
└────────┘     │             │
┌────────┐     │             │
│ Laptop │────▶│             │
└────────┘     └─────────────┘
```

**Sama seperti USB-C menyatukan berbagai konektor**, MCP menyatukan cara AI berkomunikasi dengan berbagai sistem:

```
SEBELUM MCP (Integrasi Custom untuk Setiap Sistem):
┌─────────┐     ┌───────────────────┐
│ Claude  │────▶│ Custom MySQL API  │
└─────────┘     └───────────────────┘
┌─────────┐     ┌───────────────────┐
│ Gemini  │────▶│ Custom Slack API  │
└─────────┘     └───────────────────┘
┌─────────┐     ┌───────────────────┐
│ GPT     │────▶│ Custom GitHub API │
└─────────┘     └───────────────────┘

SESUDAH MCP (Satu Protokol untuk Semua):
┌─────────┐     ┌───────────────────┐     ┌────────────┐
│ Claude  │     │                   │     │   MySQL    │
├─────────┤     │                   │     ├────────────┤
│ Gemini  │────▶│       MCP         │────▶│   Slack    │
├─────────┤     │                   │     ├────────────┤
│ GPT     │     │                   │     │   GitHub   │
└─────────┘     └───────────────────┘     └────────────┘
```

---

## 🎯 Masalah yang Dipecahkan MCP

### Masalah 1: Fragmentasi Integrasi

**Sebelum MCP:**
- Setiap LLM punya cara sendiri untuk call external tools
- Developer harus buat integrasi berbeda untuk Claude, GPT, Gemini
- Maintenance nightmare — update satu, harus update semua

**Sesudah MCP:**
- Satu standar protokol untuk semua LLM
- Buat sekali, pakai di mana-mana
- Update di satu tempat, semua LLM dapat manfaatnya

### Masalah 2: AI Tidak Punya Akses ke Data Real-time

**Sebelum MCP:**
```
User: "Berapa stok produk A di warehouse?"
AI: "Maaf, saya tidak punya akses ke sistem inventory Anda."
```

**Sesudah MCP:**
```
User: "Berapa stok produk A di warehouse?"
AI: [Calls MCP tool: get_inventory(product="A")]
AI: "Stok produk A saat ini adalah 150 unit di warehouse Jakarta."
```

### Masalah 3: Typist vs Architect Problem

| Typist (Tanpa MCP) | Architect (Dengan MCP) |
|--------------------|------------------------|
| User → AI → User → Manual Query → User | User → AI → MCP → Database → AI → User |
| 5 langkah, human in the loop | 1 langkah, fully automated |
| Rentan human error | Consistent & reliable |

---

## 🏗️ Arsitektur MCP

### Overview Arsitektur

```
┌─────────────────────────────────────────────────────────────────┐
│                         MCP ECOSYSTEM                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐         ┌──────────────┐                     │
│  │  MCP CLIENT  │◀───────▶│  MCP SERVER  │                     │
│  │  (AI Agent)  │   JSON  │  (Your App)  │                     │
│  └──────────────┘   RPC   └──────────────┘                     │
│         │                        │                              │
│         │                        │                              │
│         ▼                        ▼                              │
│  ┌──────────────┐         ┌──────────────┐                     │
│  │    LLM       │         │  External    │                     │
│  │ (Claude/GPT) │         │  Resources   │                     │
│  └──────────────┘         │  - Database  │                     │
│                           │  - APIs      │                     │
│                           │  - Files     │                     │
│                           └──────────────┘                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Alur Komunikasi

```
Step 1: User Request
┌────────┐
│  User  │──▶ "Tampilkan top 5 customer bulan ini"
└────────┘

Step 2: AI Memahami & Memilih Tool
┌────────┐
│   AI   │──▶ "Saya perlu tool query_database untuk menjawab ini"
└────────┘

Step 3: MCP Client Kirim Request ke MCP Server
┌────────────┐     ┌────────────┐
│ MCP Client │────▶│ MCP Server │
│            │     │            │
│ {          │     │ Receive:   │
│   "tool":  │     │ Parse JSON │
│   "query_db"     │ Execute    │
│   "params":│     │            │
│   {...}    │     │            │
│ }          │     │            │
└────────────┘     └────────────┘

Step 4: MCP Server Execute & Return
┌────────────┐     ┌────────────┐
│ MCP Server │────▶│    MySQL   │
│            │◀────│            │
│ Format     │     │ Raw Data   │
│ Response   │     │            │
└────────────┘     └────────────┘

Step 5: AI Generate Final Answer
┌────────┐
│   AI   │──▶ "Top 5 customer bulan ini adalah: 1. PT ABC (Rp 500jt)..."
└────────┘
```

---

## 🔧 Komponen MCP Dijelaskan

### 1. MCP Server

**Apa itu?**
MCP Server adalah aplikasi yang menyediakan "capabilities" (kemampuan) kepada AI agent. Ibarat restoran, MCP Server adalah menu + dapur.

**Apa yang disediakan MCP Server:**

| Capability | Deskripsi | Contoh |
|------------|-----------|--------|
| **Resources** | Data statis atau semi-statis | File configs, dokumentasi |
| **Tools** | Fungsi yang bisa dipanggil | `query_database()`, `send_email()` |
| **Prompts** | Template prompt yang reusable | "Summarize this data..." |

**Contoh MCP Server sederhana (Node.js):**

```javascript
// mcp-server.js
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

const server = new McpServer({
  name: "my-company-mcp",
  version: "1.0.0"
});

// Definisi Tool: Query Database
server.tool(
  "query_database",
  "Execute SQL query ke database MySQL perusahaan",
  {
    query: { type: "string", description: "SQL query to execute" }
  },
  async ({ query }) => {
    // Execute query ke MySQL
    const result = await mysql.query(query);
    return { content: [{ type: "text", text: JSON.stringify(result) }] };
  }
);

// Definisi Resource: Dokumentasi API
server.resource(
  "api-docs",
  "Dokumentasi API internal perusahaan",
  async () => {
    return { content: apiDocumentation };
  }
);

server.run();
```

### 2. MCP Client

**Apa itu?**
MCP Client adalah sisi AI agent yang "memanggil" capabilities dari MCP Server. Ibarat customer di restoran yang memesan dari menu.

**Siapa yang jadi MCP Client:**
- Claude (via Claude Desktop atau API)
- VS Code dengan extension tertentu
- Antigravity (Google's AI agent)
- Custom aplikasi yang mengimplementasikan MCP client

### 3. Transport Layer

**Apa itu?**
Cara komunikasi antara Client dan Server. Ada 2 opsi:

| Transport | Kapan Dipakai | Pros | Cons |
|-----------|---------------|------|------|
| **stdio** | Local processes | Simple, no network | Single machine only |
| **HTTP/SSE** | Remote/network | Distributed, scalable | More complex setup |

**Contoh stdio (Local):**
```bash
# MCP Server berjalan sebagai subprocess
claude --mcp-server "./my-mcp-server.js"
```

**Contoh HTTP/SSE (Remote):**
```javascript
// Server berjalan di URL
const server = new McpServer({ transport: "http", port: 3000 });

// Client connect via HTTP
const client = new McpClient("http://mcp-server.mycompany.com:3000");
```

---

## 💬 Tool-Aware Prompting

### Apa itu Tool-Aware Prompting?

Cara menulis prompt yang **membuat AI sadar** tentang tools yang tersedia dan **kapan harus menggunakannya**.

### Perbandingan Prompt

```markdown
# ❌ PROMPT BIASA (Typist)
"Berapa total penjualan bulan lalu?"

# AI akan menjawab: 
"Maaf, saya tidak punya akses ke data penjualan Anda."
```

```markdown
# ✅ TOOL-AWARE PROMPT (Architect)
"Kamu memiliki akses ke database penjualan melalui tool `query_sales_db`.
Gunakan tool tersebut untuk mengambil total penjualan bulan lalu.
Format hasil sebagai: 
- Total: Rp XXX
- Jumlah transaksi: XXX
- Rata-rata per transaksi: Rp XXX"

# AI akan:
1. Recognize ada tool yang bisa dipakai
2. Call tool: query_sales_db({ period: "last_month" })
3. Format hasil sesuai request
```

### Template Tool-Aware Prompt

```markdown
## Context
Kamu adalah [ROLE]. Kamu memiliki akses ke tools berikut:
- `tool_name_1`: [deskripsi singkat]
- `tool_name_2`: [deskripsi singkat]

## Task
[Jelaskan apa yang harus dilakukan]

## Constraints
- Gunakan tool `X` untuk [use case]
- Jangan gunakan tool `Y` kecuali [kondisi]
- Format output sebagai [format]

## Example
Input: [contoh input]
Tool call: [contoh tool call]
Output: [contoh output]
```

### Real-World Example

```markdown
## Context
Kamu adalah Sales Analyst AI. Kamu memiliki akses ke tools berikut:
- `query_bigquery`: Menjalankan SQL query ke BigQuery data warehouse
- `get_customer_info`: Mengambil detail customer berdasarkan ID
- `create_report`: Membuat PDF report

## Task
Analisis performa penjualan Q4 2025 dan identifikasi:
1. Top 10 customer by revenue
2. Produk dengan pertumbuhan tertinggi
3. Region dengan penurunan penjualan

## Constraints
- Gunakan `query_bigquery` untuk semua data retrieval
- Gunakan `create_report` hanya setelah analisis selesai
- Format angka dalam Rupiah dengan separator ribuan

## Example
Input: "Analisis Q4 2025"
Tool call: query_bigquery({ sql: "SELECT customer_name, SUM(amount) as revenue FROM sales WHERE quarter='Q4' AND year=2025 GROUP BY 1 ORDER BY 2 DESC LIMIT 10" })
Output: 
| Rank | Customer | Revenue |
|------|----------|---------|
| 1 | PT ABC | Rp 5.2M |
| 2 | CV XYZ | Rp 4.1M |
...
```

---

## 🛠️ Implementasi Praktis dengan n8n

### Konsep: n8n sebagai MCP Bridge

Karena n8n adalah automation platform, kita bisa menggunakannya sebagai **"MCP Bridge"** — jembatan antara AI agent dan sistem eksternal.

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ AI Agent     │────▶│     n8n      │────▶│  BigQuery    │
│ (Antigravity)│     │ (MCP Bridge) │     │  MySQL       │
│              │◀────│              │◀────│  APIs        │
└──────────────┘     └──────────────┘     └──────────────┘
      │                    │
      │   HTTP/Webhook     │
      └────────────────────┘
```

### Step-by-Step: Membuat MCP Bridge di n8n

#### Step 1: Buat Webhook Endpoint

Node ini akan menjadi "pintu masuk" untuk request dari AI agent.

```
Webhook Node:
- Path: /mcp-query
- Method: POST
- Response Mode: Response to Webhook
```

#### Step 2: Parse & Validate Request

```javascript
// Code Node: Parse MCP Request
const body = $input.first().json.body;

// Validate required fields
if (!body.tool) {
  throw new Error('Missing required field: tool');
}

// Map tool name to action
const toolActions = {
  'query_bigquery': 'bigquery',
  'get_customer': 'mysql',
  'send_notification': 'slack'
};

const action = toolActions[body.tool];
if (!action) {
  throw new Error(`Unknown tool: ${body.tool}`);
}

return [{
  json: {
    action: action,
    tool: body.tool,
    parameters: body.parameters || {},
    requestId: $now.toMillis()
  }
}];
```

#### Step 3: Route ke Service yang Tepat

```
Switch Node:
- Condition: action
- Routes:
  - "bigquery" → BigQuery Node
  - "mysql" → MySQL Node
  - "slack" → Slack Node
```

#### Step 4: Execute & Format Response

```javascript
// Code Node: Format MCP Response
const result = $input.first().json;
const request = $('Parse MCP Request').first().json;

return [{
  json: {
    success: true,
    tool: request.tool,
    requestId: request.requestId,
    result: {
      data: result,
      rowCount: Array.isArray(result) ? result.length : 1,
      executedAt: new Date().toISOString()
    }
  }
}];
```

#### Step 5: Return Response

```
Respond to Webhook Node:
- Response Body: ={{ $json }}
- Headers:
  - Content-Type: application/json
  - X-MCP-Version: 1.0
```

### Complete Workflow Diagram

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Webhook    │───▶│   Parse &   │───▶│   Switch    │
│  Trigger    │    │  Validate   │    │   (Route)   │
└─────────────┘    └─────────────┘    └──────┬──────┘
                                             │
                   ┌─────────────────────────┼─────────────────────────┐
                   │                         │                         │
                   ▼                         ▼                         ▼
            ┌─────────────┐          ┌─────────────┐          ┌─────────────┐
            │  BigQuery   │          │    MySQL    │          │    Slack    │
            └──────┬──────┘          └──────┬──────┘          └──────┬──────┘
                   │                         │                         │
                   └─────────────────────────┼─────────────────────────┘
                                             │
                                             ▼
                                      ┌─────────────┐
                                      │   Format    │
                                      │   Response  │
                                      └──────┬──────┘
                                             │
                                             ▼
                                      ┌─────────────┐
                                      │   Return    │
                                      │   Webhook   │
                                      └─────────────┘
```

---

## 🏭 Custom MCP Server untuk Legacy System

### Kenapa Perlu Custom MCP?

> **Video Insight [16:09]**: Perusahaan besar akan butuh MCP internal untuk menghubungkan data sensitif mereka (order management, member system) ke AI agent secara aman.

**Public MCP**: MySQL, GitHub, Slack — sudah ada
**Custom MCP**: Order Management, ERP Legacy, Internal CRM — harus dibuat sendiri

### Arsitektur Custom MCP

```
┌─────────────────────────────────────────────────────────────────┐
│                    CUSTOM MCP ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    LEGACY SYSTEMS                        │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │   │
│  │  │  ERP    │  │   CRM   │  │Inventory│  │   HRIS  │    │   │
│  │  │ (SOAP)  │  │ (REST)  │  │  (FTP)  │  │  (SQL)  │    │   │
│  │  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘    │   │
│  └───────┼────────────┼────────────┼────────────┼──────────┘   │
│          │            │            │            │               │
│          └────────────┴────────────┴────────────┘               │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                 n8n MCP BRIDGE LAYER                     │   │
│  │                                                          │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐              │   │
│  │  │ Protocol │  │  Data    │  │ Security │              │   │
│  │  │ Adapter  │  │Transform │  │  Layer   │              │   │
│  │  └──────────┘  └──────────┘  └──────────┘              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    MCP STANDARD API                      │   │
│  │              (Single Endpoint for AI Agent)              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│                       ┌─────────────┐                          │
│                       │  AI Agent   │                          │
│                       │(Antigravity)│                          │
│                       └─────────────┘                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Langkah Membungkus Legacy System

#### 1. Identifikasi Legacy APIs

```
| System    | Protocol | Authentication | Data Format |
|-----------|----------|----------------|-------------|
| ERP       | SOAP     | Basic Auth     | XML         |
| CRM       | REST     | API Key        | JSON        |
| Inventory | FTP      | Username/Pass  | CSV         |
| HRIS      | SQL      | Connection Str | Rows        |
```

#### 2. Buat Protocol Adapter

```javascript
// Code Node: Protocol Adapter
const request = $input.first().json;
const system = request.targetSystem;

async function callLegacy(system, params) {
  switch(system) {
    case 'erp':
      // SOAP call
      return await callSOAP({
        url: process.env.ERP_URL,
        action: params.action,
        body: buildSOAPEnvelope(params)
      });
    
    case 'crm':
      // REST call
      return await fetch(process.env.CRM_URL + params.endpoint, {
        headers: { 'X-API-Key': process.env.CRM_KEY }
      });
    
    case 'inventory':
      // FTP download + parse CSV
      const file = await ftpDownload(params.filename);
      return parseCSV(file);
    
    case 'hris':
      // Direct SQL
      return await pgClient.query(params.sql);
  }
}

const result = await callLegacy(system, request.params);
return [{ json: result }];
```

#### 3. Buat Data Transformer

```javascript
// Code Node: Data Transformer (Legacy → MCP Standard)
const legacyData = $input.first().json;
const sourceSystem = $('Route').first().json.system;

function transformToMCP(data, source) {
  // Standarisasi format output
  switch(source) {
    case 'erp':
      // XML → JSON
      return {
        orderId: data.OrderHeader.OrderID,
        customer: data.OrderHeader.CustomerName,
        items: data.OrderLines.map(l => ({
          sku: l.ItemCode,
          qty: parseInt(l.Quantity),
          price: parseFloat(l.UnitPrice)
        })),
        total: parseFloat(data.OrderHeader.TotalAmount)
      };
    
    case 'crm':
      // Already JSON, just restructure
      return {
        customerId: data.id,
        name: data.contact_name,
        email: data.email_address,
        segment: data.customer_segment
      };
    
    case 'inventory':
      // CSV rows → JSON array
      return data.map(row => ({
        sku: row[0],
        warehouse: row[1],
        quantity: parseInt(row[2]),
        lastUpdated: row[3]
      }));
  }
}

return [{ 
  json: {
    success: true,
    data: transformToMCP(legacyData, sourceSystem),
    metadata: {
      source: sourceSystem,
      transformedAt: new Date().toISOString()
    }
  }
}];
```

#### 4. Tambahkan Security Layer

```javascript
// Code Node: Security Layer
const request = $input.first().json;

// 1. Validate API Key
const apiKey = $input.first().headers['x-api-key'];
if (apiKey !== process.env.MCP_API_KEY) {
  throw new Error('Unauthorized: Invalid API key');
}

// 2. Check Rate Limit
const clientId = request.clientId;
const requestCount = await redis.incr(`rate:${clientId}`);
await redis.expire(`rate:${clientId}`, 60);

if (requestCount > 100) {
  throw new Error('Rate limit exceeded: 100 requests per minute');
}

// 3. Audit Log
await db.insert('mcp_audit_log', {
  clientId: clientId,
  tool: request.tool,
  timestamp: new Date(),
  ipAddress: $input.first().headers['x-forwarded-for']
});

// 4. Pass through if all checks pass
return $input.all();
```

---

## 📝 Latihan Praktis

### Latihan 1: Hello World MCP (Beginner)

**Objective**: Buat endpoint n8n yang menerima request dan return response standar MCP.

**Steps**:
1. Buat workflow baru di n8n
2. Tambah Webhook node (POST /mcp-hello)
3. Tambah Code node untuk format response
4. Tambah Respond to Webhook node

**Expected Output**:
```json
POST /webhook/mcp-hello
Body: { "name": "Ahmad" }

Response:
{
  "success": true,
  "message": "Hello, Ahmad! MCP is working.",
  "timestamp": "2026-01-16T19:00:00Z"
}
```

---

### Latihan 2: Query Database via MCP (Intermediate)

**Objective**: Buat MCP endpoint yang bisa query MySQL menggunakan natural language.

**Steps**:
1. Webhook menerima natural language query
2. AI (via HTTP Request ke OpenAI/Gemini) convert ke SQL
3. Execute SQL ke MySQL
4. Return formatted results

**Input**:
```json
{
  "tool": "query_db",
  "query": "Tampilkan 5 customer dengan pembelian terbanyak"
}
```

**Flow**:
```
Natural Language → AI → SQL → MySQL → Formatted JSON
```

---

### Latihan 3: Custom MCP untuk Sistem Internal (Advanced)

**Objective**: Bungkus sistem absensi internal (misal via API atau database) menjadi MCP Server.

**Tools yang harus didefinisikan**:
- `get_attendance`: Ambil data kehadiran karyawan
- `get_leave_balance`: Cek sisa cuti
- `request_leave`: Ajukan cuti (dengan approval workflow)

**Challenges**:
- Handle authentication
- Data masking (sembunyikan salary info)
- Audit logging

---

## ❓ Troubleshooting & FAQ

### Q1: Webhook tidak menerima request

**Kemungkinan penyebab**:
- Workflow belum di-activate
- URL salah (cek production vs test URL)
- Firewall memblokir incoming connection

**Solusi**:
```
1. Pastikan workflow Active (toggle hijau)
2. Gunakan URL yang benar:
   - Test: https://n8n.yourserver.com/webhook-test/mcp-query
   - Production: https://n8n.yourserver.com/webhook/mcp-query
3. Cek firewall rules
```

### Q2: Error "Cannot read property 'body' of undefined"

**Penyebab**: Request tidak punya body atau format salah

**Solusi**:
```javascript
// Tambahkan defensive check
const body = $input.first().json.body || $input.first().json;

if (!body || Object.keys(body).length === 0) {
  throw new Error('Request body is empty');
}
```

### Q3: AI tidak menggunakan tool yang disediakan

**Penyebab**: Prompt tidak cukup jelas tentang kapan pakai tool

**Solusi**: Gunakan Tool-Aware Prompting yang explicit:
```markdown
WAJIB gunakan tool `query_database` untuk pertanyaan tentang:
- Data penjualan
- Customer information
- Inventory status

JANGAN menjawab dari pengetahuan umum untuk pertanyaan data.
```

### Q4: Response time terlalu lambat

**Kemungkinan penyebab**:
- Query database tidak optimal
- Terlalu banyak data diproses
- Network latency

**Solusi**:
```
1. Tambahkan LIMIT di query SQL
2. Gunakan caching (Redis) untuk data yang sering diakses
3. Implementasi pagination untuk data besar
4. Monitor dengan timing logs
```

---

## 🎓 Checklist Penguasaan Phase 1

Sebelum lanjut ke Phase 2, pastikan Anda sudah:

### Konsep (Harus bisa jelaskan)
- [ ] Apa itu MCP dan mengapa penting
- [ ] Perbedaan MCP Client vs Server
- [ ] Transport layer: stdio vs HTTP/SSE
- [ ] Tool-Aware Prompting vs prompt biasa

### Praktis (Harus bisa buat)
- [ ] n8n workflow sebagai MCP endpoint
- [ ] Validasi dan security layer
- [ ] Integrasi ke minimal 1 database (MySQL/PostgreSQL/BigQuery)
- [ ] Custom MCP untuk 1 sistem internal

### Bonus (Nice to have)
- [ ] Rate limiting implementation
- [ ] Audit logging
- [ ] Error handling yang comprehensive
- [ ] Monitoring/alerting setup

---

## 🔗 Resources

### Dokumentasi
- [MCP Specification](https://modelcontextprotocol.io)
- [n8n Documentation](https://docs.n8n.io)
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)

### Video
- [Programmer Zaman Now - MCP](https://youtu.be/xNFeC4bJ4FY)

### Templates
- [Phase1-MCP-BigQuery-Connector.json](./Phase1-MCP-BigQuery-Connector.json)

---

*Phase 1 Deep Dive | Version 1.0 | 2026-01-16*
