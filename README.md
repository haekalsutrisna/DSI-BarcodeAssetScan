# 📱 Fixed Asset Barcode Scanner
## Dokumentasi Lengkap — Setup, Development & Deployment
**Power Apps + Power Automate + Dynamics 365 Finance & Operations**

---

> **Project:** BarcodeScanner Fixed Asset  
> **Company:** PT. Dinamika Solusi Indonesia  
> **Developer:** Muhammad Haekal Sutrisna  
> **Environment:** CNMF — Contoso Entertainment China  
> **F&O Instance:** `dev-hseeb5e1653ceb6510devaos.axcloud.dynamics.com`  
> **Tanggal:** 12 Mei 2026  

---

## 📋 Daftar Isi

1. [Overview Arsitektur](#1-overview-arsitektur)
2. [Prerequisites](#2-prerequisites)
3. [Setup Azure App Registration](#3-setup-azure-app-registration)
4. [Setup Connection di Power Automate](#4-setup-connection-di-power-automate)
5. [Setup Solution di Power Platform](#5-setup-solution-di-power-platform)
6. [dataAreaId — Cek Akses Legal Entity User](#6-dataareid--cek-akses-legal-entity-user)
7. [Flow 1 — Validate / Get Fixed Asset](#7-flow-1--validate--get-fixed-asset)
8. [Flow 2 — Create Fixed Asset](#8-flow-2--create-fixed-asset)
9. [Power Apps — Scanner WMS](#9-power-apps--scanner-wms)
10. [QR Code Format & Data Testing](#10-qr-code-format--data-testing)
11. [Data Fixed Asset CNMF](#11-data-fixed-asset-cnmf)
12. [Troubleshooting](#12-troubleshooting)
13. [Alur Logika Lengkap](#13-alur-logika-lengkap)

---

## 1. Overview Arsitektur

```
┌─────────────────────────────────────────────────────────┐
│                    QR Code                              │
│         Format: AssetId|AssetName|Location              │
│         Contoh: CN-C000001|笔记本|Ruang IT Lantai 2     │
└────────────────────────┬────────────────────────────────┘
                         ↓ Scan via BarcodeReader1
┌─────────────────────────────────────────────────────────┐
│              Power Apps — Scanner WMS                   │
│  Split("|") → AssetId | AssetName | Location            │
│                                                         │
│  [Validate] → Flow 1 → D365 F&O OData FixedAssetsV2    │
│                ├── Found    → Save DISABLED             │
│                └── NotFound → Save ENABLED              │
│                                                         │
│  [Save]    → Flow 2 → D365 F&O Create FixedAssetsV2    │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│           Dynamics 365 Finance & Operations             │
│           Entity: FixedAssetsV2                         │
│           Company: CNMF                                 │
└─────────────────────────────────────────────────────────┘
```

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Power Apps Canvas App |
| Middleware | Power Automate Cloud Flow |
| Backend | Dynamics 365 F&O (OData REST API) |
| Auth | Microsoft Entra ID (OAuth 2.0 — Authorization Code) |
| Integration | D365 F&O Connector (shared_dynamicsax) |

---

## 2. Prerequisites

### Akun & Akses
- ✅ Akun Dynamics 365 F&O aktif di Azure Cloud
- ✅ Akses Power Apps (make.powerapps.com)
- ✅ Akses Power Automate (make.powerautomate.com)
- ✅ Akses Azure Portal (portal.azure.com) sebagai Admin
- ✅ Role: Global Admin atau Power Platform Admin

### Informasi yang Diperlukan
- **F&O URL:** `https://dev-hseeb5e1653ceb6510devaos.axcloud.dynamics.com`
- **Legal Entity / Company:** `CNMF`
- **Fixed Asset Groups:** `BLDG`, `COMP`, `EQUIP`
- **Tenant ID:** dari Microsoft Entra ID

### Software & Tools
- Browser: Microsoft Edge / Chrome
- Power Apps Mobile (untuk testing barcode di HP)

---

## 3. Setup Azure App Registration

### 3.1 — Register App di Azure Portal

1. Buka **portal.azure.com** → login dengan akun admin
2. Search → **"App registrations"** → klik **New registration**
3. Isi:

| Field | Value |
|-------|-------|
| Name | `PowerApps-FO-Connector` |
| Account type | Single tenant |
| Redirect URI | Kosongkan |

4. Klik **Register**
5. **Catat nilai berikut:**

```
Application (Client) ID:  xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Directory (Tenant) ID:    xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

### 3.2 — Buat Client Secret

1. Di app registration → **Certificates & secrets**
2. Klik **New client secret**

| Field | Value |
|-------|-------|
| Description | `FO-PowerApps` |
| Expiry | 24 months |

3. Klik **Add**
4. ⚠️ **COPY VALUE SEKARANG** — tidak bisa dilihat lagi setelah pindah halaman

```
Client Secret Value: xxxxxxxxxxxxxxxxxxxxxxxx
```

### 3.3 — Grant API Permission ke D365 F&O

1. Di app registration → **API permissions**
2. Klik **Add a permission** → tab **"APIs my organization uses"**
3. Search dengan Application ID: `00000015-0000-0000-c000-000000000000`
4. Pilih **Dynamics ERP**
5. Pilih permission berikut:

| Permission | Type | Fungsi |
|-----------|------|--------|
| `AX.FullAccess` | Delegated | Autentikasi user ke F&O |
| `Odata.FullAccess` | Delegated | Read/Write data via OData |

6. Klik **Add permissions**
7. Klik **✅ Grant admin consent for [org]**
8. Pastikan status menjadi **hijau ✅**

### 3.4 — Register App di Dalam D365 F&O

1. Buka D365 F&O → navigasi ke:
```
System Administration → Setup → Microsoft Entra ID Applications
```
2. Klik **New**

| Field | Value |
|-------|-------|
| Client ID | Application Client ID dari Step 3.1 |
| Name | `PowerApps Connector` |
| User ID | Admin user atau service account |

3. Klik **Save** ✅

---

## 4. Setup Connection di Power Automate

> ⚠️ **PENTING:** Jangan gunakan Client ID + Secret untuk connection D365 F&O di Power Automate. Connector ini menggunakan `authorization_code` grant (user login), bukan `client_credentials`. Menggunakan client credentials akan menyebabkan error:
> ```
> token:grantType has value: client_credentials 
> But only accepts: code
> ```

### Cara Buat Connection yang Benar

1. Buka **make.powerautomate.com**
2. Sidebar kiri → **Connections**
3. Klik **+ New connection**
4. Search: **"Dynamics 365 Finance and Operations"**
5. Klik **Create**
6. Popup Microsoft Login muncul → **login dengan akun D365 kamu**
7. Klik **Accept/Allow**
8. Connection muncul dengan status ✅ hijau

> ✅ Tidak perlu Client ID, Client Secret, atau Tenant ID untuk cara ini.

---

## 5. Setup Solution di Power Platform

### Mengapa Pakai Solution?

| Tanpa Solution | Dengan Solution |
|---------------|----------------|
| Flow susah dipanggil dari Power Apps | Flow langsung terdeteksi |
| Tidak bisa deploy ke environment lain | Mudah export/import |
| Tidak ada version control | Ada versioning |

### Buat Solution Baru

1. Buka **make.powerapps.com**
2. Sidebar kiri → **Solutions**
3. Klik **+ New solution**

| Field | Value |
|-------|-------|
| Display name | `Barcode Asset Scan` |
| Name | `BarcodeAssetScan` |
| Publisher | Pilih atau buat publisher |
| Version | `1.0.0.0` |

4. Klik **Create**

### Tambahkan Flow & App ke Solution

1. Buka solution → klik **+ Add existing**
2. Tambahkan:
   - **Automation → Cloud flow** → pilih `Flow 1 Barcode Assets Scan`
   - **Automation → Cloud flow** → pilih `Flow 2 Barcode Assets Scan`
   - **App → Canvas app** → pilih `Scanner WMS`

### Struktur Solution Final

```
📦 Barcode Asset Scan (Solution)
├── 📱 Scanner WMS (Canvas App)
├── ⚡ Flow 1 Barcode Assets Scan (GET)
└── ⚡ Flow 2 Barcode Assets Scan (CREATE)
```

---

## 6. dataAreaId — Cek Akses Legal Entity User

> ⚠️ **PENTING:** `dataAreaId` yang bisa digunakan di OData filter **tergantung akses Legal Entity** yang dimiliki akun yang dipakai untuk koneksi Power Automate. Menggunakan `dataAreaId` yang tidak diizinkan akan menyebabkan data kosong atau error akses.

---

### Cara Cek Legal Entity yang Bisa Diakses

1. Buka **D365 F&O**
2. Navigasi ke:
```
System Administration → Users → Users
```
3. Cari dan klik user yang dipakai untuk koneksi Power Automate  
   (contoh: `haekal@dinamika-si.com`)
4. Scroll ke section **Legal entities**
5. Catat semua **company code** yang terdaftar

---

### Contoh Tampilan Legal Entities di User

| Company Code | Company Name | Akses |
|-------------|-------------|-------|
| `CNMF` | Contoso Entertainment China | ✅ |
| `USMF` | Contoso Entertainment US | ✅ |
| `DAT` | Default Data | ✅ |

> Hanya company code di tabel ini yang bisa digunakan sebagai `dataAreaId` di filter OData.

---

### Cara Cek via URL F&O

Bisa juga cek langsung via OData di browser:
```
https://dev-hseeb5e1653ceb6510devaos.axcloud.dynamics.com/data/LegalEntities
```
Ini akan menampilkan semua Legal Entity yang bisa diakses oleh akun yang sedang login.

---

### Penggunaan dataAreaId di Flow 1 & Flow 2

#### Flow 1 — Filter Expression:
```
concat('FixedAssetNumber eq ''', triggerBody()?['text'], ''' and dataAreaId eq ''cnmf''')
```

#### Flow 2 — Create record field Company:
```
cnmf
```

> ✅ Ganti `cnmf` dengan company code sesuai akses user kamu jika berbeda.

---

### Jika Multi Legal Entity

Jika app perlu support **beberapa company sekaligus**, tambahkan input `CompanyId` di trigger Power Apps:

**Tambah input di Flow 1 & Flow 2:**

| # | Name | Type |
|---|------|------|
| 1 | `text` | Text (AssetId) |
| 2 | `companyid` | Text (dataAreaId) |

**Update filter Flow 1:**
```
concat('FixedAssetNumber eq ''', triggerBody()?['text'], ''' and dataAreaId eq ''', triggerBody()?['companyid'], '''')
```

**Update Power Apps Button Validate — OnSelect:**
```powerfx
Set(
    varAsset,
    Flow1BarcodeAssetsScan.Run(locAssetId, "cnmf")
);
```

---

## 7. Flow 1 — Validate / Get Fixed Asset

**Tujuan:** Cek apakah Fixed Asset sudah ada di D365 F&O berdasarkan AssetId dari QR code.

### Struktur Flow

```
Trigger (PowerApps)
    ↓ input: text (AssetId)
Lists items present in table (FixedAssetsV2)
    ↓ filter: FixedAssetNumber + dataAreaId
Condition: length > 0 ?
    ├── TRUE  → Parse JSON → Respond (status: "Found")
    └── FALSE → Respond (status: "NotFound")
```

### Trigger — When Power Apps calls a flow (V2)

Tambah **1 input:**

| # | Name | Type |
|---|------|------|
| 1 | `text` | Text |

### Action 1 — Lists items present in table

| Property | Value |
|----------|-------|
| Instance | `dev-hseeb5e1653ceb6510devaos.axcloud.dynamics.com` |
| Entity name | `FixedAssetsV2` |
| $filter | Expression (lihat di bawah) |
| $top | `1` |

**$filter Expression** — masukkan via tab Expression:
```
concat('FixedAssetNumber eq ''', triggerBody()?['text'], ''' and dataAreaId eq ''cnmf''')
```

### Action 2 — Condition

Klik tab **Code view** → paste:

```json
{
  "type": "If",
  "expression": {
    "greater": [
      "@length(body('Lists_items_present_in_table')?['value'])",
      0
    ]
  }
}
```

### Branch TRUE — Parse JSON

**Content** (via Expression tab):
```
body('Lists_items_present_in_table')?['value']?[0]
```

**Schema:**
```json
{
    "type": "object",
    "properties": {
        "FixedAssetNumber": {
            "type": "string"
        },
        "Name": {
            "type": "string"
        },
        "FixedAssetGroupId": {
            "type": "string"
        },
        "AssetType": {
            "type": "string"
        },
        "Location": {
            "type": "string"
        },
        "dataAreaId": {
            "type": "string"
        }
    }
}
```

### Branch TRUE — Respond to a Power App or flow

Klik **Code view** → paste:

```json
{
  "type": "Response",
  "kind": "PowerApp",
  "inputs": {
    "statusCode": 200,
    "schema": {
      "type": "object",
      "properties": {
        "assetid": {
          "title": "AssetId",
          "x-ms-dynamically-added": true,
          "type": "string",
          "x-ms-content-hint": "TEXT"
        },
        "assetname": {
          "title": "AssetName",
          "x-ms-dynamically-added": true,
          "type": "string",
          "x-ms-content-hint": "TEXT"
        },
        "location": {
          "title": "Location",
          "x-ms-dynamically-added": true,
          "type": "string",
          "x-ms-content-hint": "TEXT"
        },
        "assetgroup": {
          "title": "AssetGroup",
          "x-ms-dynamically-added": true,
          "type": "string",
          "x-ms-content-hint": "TEXT"
        },
        "status": {
          "title": "Status",
          "x-ms-dynamically-added": true,
          "type": "string",
          "x-ms-content-hint": "TEXT"
        }
      },
      "additionalProperties": {}
    },
    "body": {
      "assetid": "@{body('Parse_JSON')?['FixedAssetNumber']}",
      "assetname": "@{body('Parse_JSON')?['Name']}",
      "location": "@{body('Parse_JSON')?['Location']}",
      "assetgroup": "@{body('Parse_JSON')?['FixedAssetGroupId']}",
      "status": "Found"
    }
  }
}
```

### Branch FALSE — Respond to a Power App or flow

Klik **Code view** → paste:

```json
{
  "type": "Response",
  "kind": "PowerApp",
  "inputs": {
    "statusCode": 200,
    "schema": {
      "type": "object",
      "properties": {
        "assetid": {
          "title": "AssetId",
          "x-ms-dynamically-added": true,
          "type": "string",
          "x-ms-content-hint": "TEXT"
        },
        "assetname": {
          "title": "AssetName",
          "x-ms-dynamically-added": true,
          "type": "string",
          "x-ms-content-hint": "TEXT"
        },
        "location": {
          "title": "Location",
          "x-ms-dynamically-added": true,
          "type": "string",
          "x-ms-content-hint": "TEXT"
        },
        "assetgroup": {
          "title": "AssetGroup",
          "x-ms-dynamically-added": true,
          "type": "string",
          "x-ms-content-hint": "TEXT"
        },
        "status": {
          "title": "Status",
          "x-ms-dynamically-added": true,
          "type": "string",
          "x-ms-content-hint": "TEXT"
        }
      },
      "additionalProperties": {}
    },
    "body": {
      "assetid": "NotFound",
      "assetname": "NotFound",
      "location": "NotFound",
      "assetgroup": "NotFound",
      "status": "NotFound"
    }
  }
}
```

> ⚠️ **PENTING:** Schema kedua Respond (TRUE & FALSE) **HARUS IDENTIK** — nama field, jumlah field, dan type harus sama persis. Jika berbeda akan muncul error:
> ```
> ActionSchemaInvalid: The schema definitions for actions 
> with same status code must match.
> ```

### Save Flow 1 ✅

---

## 8. Flow 2 — Create Fixed Asset

**Tujuan:** Buat record Fixed Asset baru di D365 F&O.

### Struktur Flow

```
Trigger (PowerApps)
    ↓ inputs: AssetId, NewLocation, AssetName
Create a record (FixedAssetsV2)
    ↓
Respond to PowerApp (savestatus: "Saved")
```

### Trigger — When Power Apps calls a flow (V2)

Tambah **3 inputs:**

| # | Name | Type | Dari Power Apps |
|---|------|------|----------------|
| 1 | `AssetId` | Text | `locAssetId` |
| 2 | `NewLocation` | Text | `locLocation` |
| 3 | `AssetName` | Text | `locDescription` |

> ⚠️ Urutan parameter harus sesuai dengan urutan `.Run()` di Power Apps.

### Action 1 — Create a record

| Field | Value |
|-------|-------|
| Instance | `dev-hseeb5e1653ceb6510devaos.axcloud.dynamics.com` |
| Entity name | `FixedAssetsV2` |
| Fixed asset number | **KOSONGKAN** — auto-generate via Number Sequence |
| Fixed asset group | `COMP` (hardcode sesuai grup default) |
| Company | `cnmf` |
| Name | `⚡ AssetName` dari trigger |
| Location | `⚡ NewLocation` dari trigger |

> ⚠️ **JANGAN isi Fixed asset number secara manual** — F&O generate otomatis via Number Sequence. Jika diisi, akan error:
> ```
> Field 'Fixed asset number' does not allow editing.
> ```

> ⚠️ **Location harus kode valid** dari tabel Fixed asset locations, bukan free text.

### Action 2 — Respond to a Power App or flow

Klik **Code view** → paste:

```json
{
  "type": "Response",
  "kind": "PowerApp",
  "inputs": {
    "statusCode": 200,
    "schema": {
      "type": "object",
      "properties": {
        "savestatus": {
          "title": "SaveStatus",
          "x-ms-dynamically-added": true,
          "type": "string",
          "x-ms-content-hint": "TEXT"
        }
      },
      "additionalProperties": {}
    },
    "body": {
      "savestatus": "Saved"
    }
  }
}
```

### Save Flow 2 ✅

---

## 9. Power Apps — Scanner WMS

### Komponen Tree View

| Control | Type | Fungsi |
|---------|------|--------|
| `BarcodeReader1` | Barcode Reader | Scan QR Code |
| `TextInput1` | Text Input | Tampil Asset ID |
| `TextInput1_1` | Text Input | Tampil Description / AssetName |
| `TextInput1_2` | Text Input | Tampil Location |
| `Label1_4` | Label | Tampil User Email |
| `Label1_6` | Label | Tampil waktu sekarang |
| `Button1` | Button | Validate — trigger Flow 1 |
| `Button1_1` | Button | Save — trigger Flow 2 |
| `Label1_8` | Label | Status hasil validasi/save |

### Variables yang Digunakan

| Variable | Type | Fungsi |
|----------|------|--------|
| `locAssetId` | Context | AssetId hasil split QR |
| `locDescription` | Context | AssetName hasil split QR |
| `locLocation` | Context | Location hasil split QR |
| `varAsset` | Global | Response dari Flow 1 |
| `varResult` | Global | Response dari Flow 2 |
| `varAssetFound` | Global | Boolean — asset sudah ada di F&O |
| `varValidated` | Global | Boolean — sudah klik Validate |
| `varStatus` | Global | Text status yang ditampilkan |
| `varLoading` | Global | Boolean — loading indicator |

---

### Menambahkan Flow ke Power Apps

1. Buka Power Apps Studio → **Scanner WMS**
2. Klik **"..."** di toolbar atas → **Power Automate**
3. Panel Power Automate muncul di kiri
4. Klik **+ Add flow**
5. Pilih **Flow 1 Barcode Assets Scan**
6. Pilih **Flow 2 Barcode Assets Scan**
7. Tekan **Ctrl+S**

> ⚠️ Jika flow diupdate, lakukan **Remove → Add flow** ulang agar Power Apps refresh koneksinya.

---

### Formula Setiap Komponen

---

#### Screen1 — OnVisible

```powerfx
UpdateContext({
    locAssetId: "",
    locDescription: "",
    locLocation: ""
});
Set(varStatus, "");
Set(varValidated, false);
Set(varAssetFound, false);
Set(varLoading, false);
Set(varAsset, Blank())
```

---

#### BarcodeReader1 — OnScan

```powerfx
UpdateContext({
    locAssetId:     Index(Split(Last(BarcodeReader1.Barcodes).Value, "|"), 1).Value,
    locDescription: Index(Split(Last(BarcodeReader1.Barcodes).Value, "|"), 2).Value,
    locLocation:    Index(Split(Last(BarcodeReader1.Barcodes).Value, "|"), 3).Value
});
Set(varStatus, "");
Set(varValidated, false);
Set(varAssetFound, false);
Set(varLoading, false)
```

> ⚠️ Gunakan `Last(BarcodeReader1.Barcodes).Value` karena `.Barcodes` adalah collection, bukan single value.  
> ⚠️ Property yang tersedia di BarcodeReader1: `Barcodes`, `ContentLanguage`, `DisplayMode`, `Height`, `Width`, `X` — tidak ada `.Value` langsung.

---

#### TextInput1 — Asset ID

| Property | Formula |
|----------|---------|
| **Default** | `locAssetId` |
| **DisplayMode** | `DisplayMode.View` |
| **HintText** | `"Scan or Enter Asset ID"` |

---

#### TextInput1_1 — Description

| Property | Formula |
|----------|---------|
| **Default** | `locDescription` |
| **DisplayMode** | `DisplayMode.View` |
| **HintText** | `"Description"` |

---

#### TextInput1_2 — Location

| Property | Formula |
|----------|---------|
| **Default** | `locLocation` |
| **DisplayMode** | `DisplayMode.View` |
| **HintText** | `"Location"` |

---

#### Label1_4 — User Email

| Property | Formula |
|----------|---------|
| **Text** | `User().Email` |

---

#### Label1_6 — Waktu

| Property | Formula |
|----------|---------|
| **Text** | `Text(Now(), "yyyy-mm-dd hh:mm")` |

---

#### Button1 — Validate — OnSelect

```powerfx
Set(varLoading, true);
Set(varStatus, "🔍 Validating...");

Set(
    varAsset,
    Flow1BarcodeAssetsScan.Run(locAssetId)
);

If(
    varAsset.status = "Found",
    Set(varAssetFound, true);
    Set(varValidated, true);
    Set(varStatus, "⚠️ Asset already exists in F&O!"),

    Set(varAssetFound, false);
    Set(varValidated, true);
    Set(varStatus, "✅ Asset Validated — Ready to Save")
);

Set(varLoading, false)
```

#### Button1 — Validate — DisplayMode

```powerfx
If(IsBlank(locAssetId), DisplayMode.Disabled, DisplayMode.Edit)
```

---

#### Button1_1 — Save — OnSelect

```powerfx
If(
    !varAssetFound && varValidated,

    Set(varLoading, true);
    Set(varStatus, "💾 Saving...");

    Set(
        varResult,
        Flow2BarcodeAssetsScan.Run(
            locAssetId,
            locLocation,
            locDescription
        )
    );

    If(
        varResult.savestatus = "Saved",
        Set(varStatus, "Asset Successfully Saved ✅"),
        Set(varStatus, "❌ Save Failed! Check Flow 2")
    );

    Set(varLoading, false),

    If(
        !varValidated,
        Set(varStatus, "⚠️ Please validate first!"),
        Set(varStatus, "⚠️ Asset already exists in F&O!")
    )
)
```

#### Button1_1 — Save — DisplayMode

```powerfx
If(
    !varAssetFound && varValidated,
    DisplayMode.Edit,
    DisplayMode.Disabled
)
```

---

#### Label1_8 — Status Message

| Property | Formula |
|----------|---------|
| **Text** | `varStatus` |
| **Visible** | `Not(IsBlank(varStatus))` |
| **Color** | `If(varAssetFound \|\| !varValidated, Color.Red, Color.Green)` |
| **FontWeight** | `FontWeight.Bold` |

> ✅ Label hanya muncul jika `varStatus` tidak kosong. Setelah scan baru, `varStatus` di-reset ke `""` sehingga label otomatis hilang.

---

## 10. QR Code Format & Data Testing

### Format QR Code

```
AssetId|AssetName|Location
```

| Index | Field | Cara Split |
|-------|-------|-----------|
| 1 | AssetId | `Index(Split(..., "|"), 1).Value` |
| 2 | AssetName | `Index(Split(..., "|"), 2).Value` |
| 3 | Location | `Index(Split(..., "|"), 3).Value` |

---

### 🔴 Skenario 1 — Asset TIDAK ADA → Save Aktif

> Scan → Validate → **"✅ Asset Validated — Ready to Save"** → Klik Save → Create di F&O

```
CN-C999001|Laptop Dell Latitude|Ruang HRD Lantai 2
CN-C999002|Printer HP LaserJet|Ruang Admin Lantai 1
CN-E999001|AC Daikin 2PK|Ruang Server Lantai 3
```

---

### 🟡 Skenario 2 — Asset ADA, Lokasi Berbeda → Save Disabled

> Scan → Validate → **"⚠️ Asset already exists in F&O!"** → Save DISABLED

```
CN-C000001|笔记本|Ruang Gudang Lantai 4
CN-C000003|投影仪电视机|Lobby Utama Gedung A
CN-E000000001|52寸X590高清电视机|Ruang Direksi Lantai 5
```

---

### 🟢 Skenario 3 — Asset ADA, Lokasi Sesuai → Save Disabled

> Scan → Validate → **"⚠️ Asset already exists in F&O!"** → Save DISABLED

```
CN-C000001|笔记本|Ruang IT Lantai 2
CN-C000002|USB转换器|Ruang IT Lantai 2
CN-B000001|普通建筑物|Lantai 1 Gedung Utama
```

---

## 11. Data Fixed Asset CNMF

### Data Existing di D365 F&O (Company: CNMF)

| Fixed Asset Number | Name | Fixed Asset Group | Type | Location |
|-------------------|------|------------------|------|----------|
| CN-B000001 | 普通建筑物 | BLDG | Tangible | *(kosong)* |
| CN-B000002 | 主要建筑物 | BLDG | Tangible | *(kosong)* |
| CN-C000001 | 笔记本 | COMP | Tangible | *(kosong)* |
| CN-C000002 | USB转换器 | COMP | Tangible | *(kosong)* |
| CN-C000003 | 投影仪电视机 | COMP | Tangible | *(kosong)* |
| CN-C000004 | 37寸M120电视机 | COMP | Tangible | *(kosong)* |
| CN-E000000001 | 52寸X590高清电视机 | EQUIP | Tangible | *(kosong)* |

### Fixed Asset Groups CNMF

| Kode | Name | Number Sequence |
|------|------|----------------|
| BLDG | 建筑物 | CNFA-BLDG |
| COMP | 计算机及外设 | Auto |
| EQUIP | *(Peralatan)* | Auto |

---

## 12. Troubleshooting

### ❌ Error: Data kosong meskipun asset ada di F&O

```
Lists items present in table mengembalikan array kosong
meskipun asset terlihat ada di F&O.
```

**Penyebab:** `dataAreaId` di filter tidak sesuai dengan akses Legal Entity akun yang dipakai koneksi.  
**Fix:**
1. Buka D365 F&O → `System Administration → Users → Users`
2. Klik user yang dipakai koneksi Power Automate
3. Cek section **Legal entities** — pastikan company code (`cnmf`, `usmf`, dll.) terdaftar di sana
4. Gunakan hanya company code yang ada di list tersebut di filter Flow 1

---

### ❌ Error: `client_credentials` Grant Type

```
token:grantType has value: client_credentials
But only accepts: code
```

**Penyebab:** Mengisi Client ID + Secret di form connection Power Automate.  
**Fix:** Buat connection via **Connections page** → login dengan akun Microsoft, bukan pakai Client ID/Secret.

---

### ❌ Error: Only 1 of 2 keys provided

```
Only 1 of 2 keys provided for lookup, 
provide keys for dataAreaId, FixedAssetNumber.
```

**Penyebab:** Menggunakan action **"Get a record"** yang memerlukan composite key.  
**Fix:** Ganti ke **"Lists items present in table"** dengan filter:
```
concat('FixedAssetNumber eq ''', triggerBody()?['text'], ''' and dataAreaId eq ''cnmf''')
```

---

### ❌ Error: ActionSchemaInvalid — Schema must match

```
The schema definitions for actions with same 
status code must match.
```

**Penyebab:** Respond to PowerApp di cabang TRUE dan FALSE memiliki schema yang berbeda.  
**Fix:** Pastikan kedua Respond memiliki field yang **100% identik** — nama, jumlah, dan type sama persis.

---

### ❌ Error: 502 BadGateway / NoResponse

```
The server did not receive a response from upstream server.
```

**Penyebab:** D365 F&O dev environment sedang **hibernate/sleep**.  
**Fix:**
1. Buka URL F&O di browser → login → tunggu fully loaded
2. Cek status di LCS (lcs.dynamics.com) → pastikan **Deployed/Running**
3. Atau start VM dari Azure Portal jika status Stopped

---

### ❌ Error: Field Fixed asset number does not allow editing

```
Field 'Fixed asset number' does not allow editing.
```

**Penyebab:** Mengisi field Fixed asset number di Create record.  
**Fix:** **Kosongkan** field tersebut — F&O auto-generate via Number Sequence.

---

### ❌ Error: Value in field Location not found

```
Value in field 'Location' is not found in related table.
```

**Penyebab:** Location menggunakan free text, bukan kode valid dari tabel Location.  
**Fix:** Gunakan kode Location yang valid dari F&O:
```
Fixed assets → Setup → Fixed asset locations
```

---

### ❌ Scan tidak mengisi TextInput

**Penyebab:** TextInput Default menggunakan `BarcodeReader1.Value` — property ini tidak ada.  
**Fix:** Gunakan `Last(BarcodeReader1.Barcodes).Value` dan simpan ke context variable di OnScan, lalu TextInput Default pakai variable tersebut:
```powerfx
// OnScan:
UpdateContext({locAssetId: Index(Split(Last(BarcodeReader1.Barcodes).Value, "|"), 1).Value})

// TextInput Default:
locAssetId
```

---

### ❌ Label Status tidak reset setelah scan baru

**Penyebab:** `varStatus` tidak di-reset di OnScan.  
**Fix:** Tambahkan di BarcodeReader1 OnScan:
```powerfx
Set(varStatus, "");
Set(varValidated, false);
Set(varAssetFound, false)
```
Dan pastikan Label1_8 Visible:
```powerfx
Not(IsBlank(varStatus))
```

---

### ❌ Validate selalu muncul "Asset Validated" meski QR salah

**Penyebab:** Flow 1 tidak ada Condition — selalu return Found meskipun asset tidak ada.  
**Fix:** Tambahkan **Condition** di Flow 1 berdasarkan `length(body(...)?['value']) > 0`. Cabang FALSE return `status: "NotFound"`.

---

### ❌ Flow tidak muncul di Power Apps

**Penyebab:** Flow belum di-add atau sudah diupdate tapi belum di-refresh.  
**Fix:** Power Automate panel → **Remove flow** → **Add flow ulang** → Ctrl+S.

---

### ⚠️ Debug Variables di Power Apps

Tambahkan label sementara untuk debug:

```powerfx
"AssetFound: " & Text(varAssetFound) & 
" | Validated: " & Text(varValidated) & 
" | Status: " & varAsset.status &
" | AssetId: " & locAssetId
```

---

## 13. Alur Logika Lengkap

```
┌─────────────────────────────────────────────────────────┐
│                   Screen1 OnVisible                     │
│   Reset: locAssetId="", locDescription="",              │
│          locLocation="", varStatus="",                  │
│          varValidated=false, varAssetFound=false         │
└──────────────────────────┬──────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│              BarcodeReader1 OnScan                      │
│   Split QR Code by "|":                                 │
│   locAssetId     = Index(..., 1).Value                  │
│   locDescription = Index(..., 2).Value                  │
│   locLocation    = Index(..., 3).Value                  │
│   Reset: varStatus="", varValidated=false               │
│          varAssetFound=false                            │
│   Label1_8 → HILANG (varStatus kosong)                  │
└──────────────────────────┬──────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│              Button1 VALIDATE OnSelect                  │
│   Flow1BarcodeAssetsScan.Run(locAssetId)                │
│                                                         │
│   ┌── varAsset.status = "Found" ──────────────────┐    │
│   │   varAssetFound = true                        │    │
│   │   varValidated = true                         │    │
│   │   varStatus = "⚠️ Asset already exists!"      │    │
│   │   Label1_8 = MERAH                            │    │
│   │   Button Save = DISABLED                      │    │
│   └───────────────────────────────────────────────┘    │
│                                                         │
│   ┌── varAsset.status = "NotFound" ───────────────┐    │
│   │   varAssetFound = false                       │    │
│   │   varValidated = true                         │    │
│   │   varStatus = "✅ Asset Validated"             │    │
│   │   Label1_8 = HIJAU                            │    │
│   │   Button Save = AKTIF                         │    │
│   └───────────────────────────────────────────────┘    │
└──────────────────────────┬──────────────────────────────┘
                           ↓
         varAssetFound=false && varValidated=true ?
                           ↓ YES
┌─────────────────────────────────────────────────────────┐
│              Button1_1 SAVE OnSelect                    │
│   Flow2BarcodeAssetsScan.Run(                           │
│       locAssetId,      → AssetId                        │
│       locLocation,     → NewLocation                    │
│       locDescription   → AssetName                      │
│   )                                                     │
│                                                         │
│   ┌── varResult.savestatus = "Saved" ─────────────┐    │
│   │   varStatus = "Asset Successfully Saved ✅"    │    │
│   │   Label1_8 = HIJAU                            │    │
│   └───────────────────────────────────────────────┘    │
│                                                         │
│   ┌── Gagal ───────────────────────────────────── ┐    │
│   │   varStatus = "❌ Save Failed! Check Flow 2"  │    │
│   │   Label1_8 = MERAH                            │    │
│   └───────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

---

## 📌 Quick Reference

### F&O OData Endpoints

| Aksi | Method | URL |
|------|--------|-----|
| List Fixed Assets | GET | `/data/FixedAssetsV2?$filter=FixedAssetNumber eq 'CN-C000001' and dataAreaId eq 'cnmf'&$top=1` |
| Create Fixed Asset | POST | `/data/FixedAssetsV2` |
| Update Fixed Asset | PATCH | `/data/FixedAssetsV2(dataAreaId='cnmf',FixedAssetNumber='CN-C000001')` |

### Power Apps Keyboard Shortcuts

| Shortcut | Fungsi |
|----------|--------|
| `F5` | Play / Preview App |
| `Ctrl+S` | Save App |
| `Ctrl+Z` | Undo |
| `Escape` | Stop Preview |

### Power Automate Test

1. Buka flow → klik **Test**
2. Pilih **Manually**
3. Klik **Test** → isi input → **Run flow**
4. Cek tiap action: ✅ hijau = sukses, ❌ merah = error

---

*Dokumentasi ini mencakup setup lengkap dari Azure App Registration hingga testing di Power Apps.*  
*Dibuat: 12 Mei 2026 | Version: 1.0.0*
