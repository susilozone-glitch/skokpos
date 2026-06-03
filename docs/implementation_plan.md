# SkokPOS — Multi-Purpose POS System

A full-featured, offline-first Point of Sales **PWA + Native App** with delivery live tracking, thermal printing, and sales best practices.

> **Platform**: PWA (browser) + Android (Play Store) + iOS (App Store) — from one codebase via Ionic + Capacitor.

## Architecture Overview

```mermaid
graph TB
    subgraph "Client (Ionic + Capacitor)"
        UI["Ionic App<br/>React Components"]
        CAP["Capacitor<br/>Native Runtime"]
        SW["Service Worker<br/>Offline Cache"]
        IDB["IndexedDB / SQLite<br/>Local Data Store"]
    end

    subgraph "Native Plugins (Capacitor)"
        GPS["@capacitor/geolocation<br/>Background GPS"]
        CAM["@capacitor/camera<br/>Photo Capture"]
        BLE["Capacitor BLE<br/>Thermal Printer"]
        BAR["Barcode Scanner<br/>Product Lookup"]
    end

    subgraph "Firebase Backend"
        AUTH["Firebase Auth<br/>Staff Login + Roles"]
        FS["Cloud Firestore<br/>Primary Database"]
        RT["Realtime Database<br/>Driver GPS Tracking"]
        FCM["Cloud Messaging<br/>Order Notifications"]
        CF["Cloud Functions<br/>Business Logic"]
        STORE["Cloud Storage<br/>Product Images + POD"]
    end

    subgraph "External"
        OSM["OpenStreetMap<br/>Leaflet Maps"]
    end

    UI --> CAP
    UI --> SW
    UI --> IDB
    CAP --> GPS
    CAP --> CAM
    CAP --> BLE
    CAP --> BAR
    SW --> FS
    IDB <-->|"Sync"| FS
    GPS -->|"Live Location"| RT
    UI --> AUTH
    UI --> OSM
    CF --> FCM
```

## Tech Stack

| Layer | Technology | Rationale |
|---|---|---|
| **Framework** | **Ionic 8 + React** | Mobile-first, 100+ native-like components, PWA built-in |
| **Native Runtime** | **Capacitor 6** | Native access: camera, GPS (background!), barcode, BLE |
| **UI** | React 19 + **Tailwind CSS v4** | Utility-first, custom layouts beyond Ionic components |
| **Components** | **Ionic UI Components** | IonModal, IonActionSheet, IonToast, IonTabs, IonList, etc. |
| **State** | Zustand + React Query | Lightweight, offline-friendly |
| **Database** | Firebase Firestore | Real-time sync, offline persistence |
| **GPS Tracking** | Firebase Realtime DB + **@capacitor/geolocation** | Background GPS for driver tracking |
| **Auth** | Firebase Auth | Role-based access (Super Admin/Admin/Cashier/Kitchen/Driver) |
| **Maps** | Leaflet + OpenStreetMap | Free, no API key needed |
| **Charts** | Recharts | Flexible chart components |
| **Printing** | ESC/POS via **Capacitor BLE / WebUSB** | Thermal printer: Bluetooth + USB |
| **Camera** | **@capacitor/camera** | Native camera for Proof of Delivery photos |
| **Barcode** | **@capacitor-community/barcode-scanner** | Native barcode scanning |
| **Offline** | Service Worker + Capacitor Preferences | Full offline-first capability |
| **Currency** | IDR (Indonesian Rupiah) | Default with formatting (Rp 50.000) |
| **Icons** | **Ionicons** + Lucide React | Native-style icons + modern set |
| **Build Targets** | **PWA + Android APK + iOS IPA** | One codebase → 3 platforms |

---

## Decisions (Confirmed)

- ✅ **App Name**: SkokPOS
- ✅ **Firebase Project**: Create new project
- ✅ **Tax Rate**: 12% PPN (configurable)
- ✅ **Receipt Language**: Bahasa Indonesia (adjustable/switchable)
- ✅ **Multi-outlet**: Yes, multi-store support included
- ✅ **Payment Gateway**: Mock for now (Midtrans/Xendit integration later)
- ✅ **Store Mode**: Dynamic — Retail (default) & Restaurant, selectable per outlet

> [!NOTE]
> **Thermal Printer**: Capacitor BLE plugin for Bluetooth printers. WebUSB fallback for USB printers. Both require HTTPS or localhost for PWA mode.

> [!NOTE]
> **GPS Tracking**: Capacitor Geolocation plugin supports **background GPS tracking** on Android & iOS native builds. PWA mode is foreground-only.

---

## Dynamic Store Mode System

SkokPOS adapts its features based on the **store category** selected during setup. Each outlet can have its own mode.

### Store Categories

| Mode | Target Business | Examples |
|---|---|---|
| **🛒 Retail** (Default) | Warung, minimarket, toko, retail | Warung Madura, Indomaret-style, toko kelontong |
| **🍽️ Restoran** | Restaurant, café, food court, bakery | Warteg, café, rumah makan, bakery |

### Feature Toggle Matrix

| Feature | 🛒 Retail | 🍽️ Restoran |
|---|:---:|:---:|
| Barcode Scanner | ✅ Primary | ✅ Optional |
| Kategori Produk | ✅ | ✅ |
| Varian Produk (Ukuran/Warna) | ✅ | ✅ |
| **Modifier / Add-on** (Topping, Level Pedas) | ❌ Hidden | ✅ Active |
| **Tipe Order** (Dine-in / Takeaway) | ❌ Hidden | ✅ Active |
| **Nomor Meja** | ❌ Hidden | ✅ Active |
| **Kitchen Display System (KDS)** | ❌ Hidden | ✅ Active |
| **Tiket Dapur (KOT)** | ❌ Hidden | ✅ Active |
| **Peran Dapur (Kitchen Role)** | ❌ Hidden | ✅ Active |
| Delivery & Tracking | ✅ | ✅ |
| Stok / Inventaris | ✅ Unit-based | ✅ + Ingredient-level |
| Pelanggan & Loyalty | ✅ | ✅ |
| Laporan & Analitik | ✅ | ✅ |
| Struk Thermal | ✅ Simple | ✅ + Meja + Tipe Order |
| Sidebar Menu | Standard | + Dapur, + Meja |

### Setup Wizard Flow

```mermaid
flowchart TD
    A["🚀 Buka SkokPOS"] --> B{"Sudah setup?"}
    B -->|Belum| C["📋 Setup Wizard"]
    B -->|Sudah| H["→ Checkout"]

    C --> D["1️⃣ Pilih Kategori Toko\n🛒 Retail / 🍽️ Restoran"]
    D --> E["2️⃣ Info Bisnis\nNama, Alamat, Telepon, Logo"]
    E --> F["3️⃣ Pengaturan Awal\nPajak, Printer, Bahasa"]
    F --> G["4️⃣ Buat Outlet Pertama\nNama outlet, Alamat"]
    G --> H
```

### Architecture: How Mode Switching Works

```javascript
// src/stores/settingsStore.js
const STORE_MODES = {
  RETAIL: 'retail',      // Warung, minimarket, toko
  RESTAURANT: 'restaurant' // Restoran, café, rumah makan
};

// Feature flags derived from store mode
const getModeFeatures = (mode) => ({
  hasModifiers: mode === 'restaurant',
  hasTableNumber: mode === 'restaurant',
  hasOrderType: mode === 'restaurant',   // Dine-in / Takeaway
  hasKitchenDisplay: mode === 'restaurant',
  hasKitchenTicket: mode === 'restaurant',
  hasKitchenRole: mode === 'restaurant',
  hasIngredientStock: mode === 'restaurant',
  hasBarcodeScanner: true,  // Both modes
  hasDelivery: true,         // Both modes
  hasLoyalty: true,          // Both modes
});
```

## Proposed Changes

### Phase 1: Project Foundation, Design System & Setup Wizard

#### [NEW] Project Setup
- Initialize Next.js 15 project with App Router
- Configure PWA with `next-pwa` (Service Worker, manifest.json)
- **Install & configure Tailwind CSS v4**
- **Initialize Shadcn/ui** (`npx shadcn@latest init`) with New York style
- Set up Firebase SDK (Auth, Firestore, Realtime DB, **Cloud Storage for logo uploads**)
- Configure offline persistence
- **Set up i18n (multi-language) system**

#### [NEW] `src/lib/i18n/` — Internationalization System
Lightweight JSON-based translation system (no heavy library needed):

- `config.js` — Language config, default locale, supported locales
- `id.json` — 🇮🇩 Bahasa Indonesia translations (default)
- `en.json` — 🇬🇧 English translations
- `useTranslation.js` — React hook: `const { t } = useTranslation()`
- `LanguageContext.jsx` — React Context provider for active language
- `formatters.js` — Locale-aware number, currency, and date formatting

**Supported Languages:**

| Code | Language | Status |
|---|---|---|
| `id` | 🇮🇩 Bahasa Indonesia | Default |
| `en` | 🇬🇧 English | Supported |

**What Gets Translated:**

| Surface | Examples |
|---|---|
| 📱 UI Labels | Menu, buttons, headings, placeholders, tooltips |
| 🧱 Receipts | Header, footer, item labels, payment labels |
| 📊 Reports | Chart titles, export headers, date formats |
| 🔔 Notifications | Low stock alerts, order updates, errors |
| ⚠️ Error Messages | Validation errors, connection issues |
| 📅 Date/Time | "3 Juni 2026" vs "June 3, 2026" |
| 💰 Currency | "Rp 50.000" (both locales, IDR stays as IDR) |
| 📦 PO & Inventory | Status labels, vendor forms, stock opname |

**Translation Sample:**

```json
// src/lib/i18n/id.json (Bahasa Indonesia)
{
  "common": {
    "save": "Simpan",
    "cancel": "Batal",
    "delete": "Hapus",
    "edit": "Ubah",
    "search": "Cari...",
    "back": "Kembali",
    "confirm": "Konfirmasi",
    "loading": "Memuat..."
  },
  "nav": {
    "checkout": "Kasir",
    "delivery": "Pengiriman",
    "kitchen": "Dapur",
    "inventory": "Inventaris",
    "vendors": "Vendor",
    "purchaseOrders": "Pesanan Pembelian",
    "stockOpname": "Stok Opname",
    "reports": "Laporan",
    "staff": "Karyawan",
    "customers": "Pelanggan",
    "settings": "Pengaturan"
  },
  "checkout": {
    "cart": "Keranjang",
    "subtotal": "Subtotal",
    "discount": "Diskon",
    "tax": "PPN",
    "total": "Total",
    "pay": "Bayar",
    "cash": "Tunai",
    "card": "Kartu",
    "ewallet": "E-Wallet",
    "change": "Kembalian",
    "holdOrder": "Tahan Pesanan",
    "recallOrder": "Ambil Pesanan",
    "dineIn": "Makan di Tempat",
    "takeaway": "Bawa Pulang",
    "tableNumber": "Nomor Meja"
  },
  "inventory": {
    "stock": "Stok",
    "lowStock": "Stok Rendah",
    "outOfStock": "Habis",
    "minStock": "Stok Minimum",
    "adjustment": "Penyesuaian",
    "reorder": "Perlu Restock"
  },
  "po": {
    "draft": "Draft",
    "sent": "Dikirim",
    "received": "Diterima",
    "partial": "Sebagian",
    "cancelled": "Dibatalkan",
    "createPO": "Buat Pesanan Pembelian",
    "sendViaWA": "Kirim via WhatsApp",
    "receiveGoods": "Terima Barang"
  },
  "receipt": {
    "thankYou": "Terima Kasih!",
    "noReturn": "Barang yang sudah dibeli tidak dapat dikembalikan",
    "cashier": "Kasir",
    "outlet": "Outlet",
    "orderType": "Tipe",
    "table": "Meja"
  }
}
```

```json
// src/lib/i18n/en.json (English)
{
  "common": {
    "save": "Save",
    "cancel": "Cancel",
    "delete": "Delete",
    "edit": "Edit",
    "search": "Search...",
    "back": "Back",
    "confirm": "Confirm",
    "loading": "Loading..."
  },
  "nav": {
    "checkout": "Checkout",
    "delivery": "Delivery",
    "kitchen": "Kitchen",
    "inventory": "Inventory",
    "vendors": "Vendors",
    "purchaseOrders": "Purchase Orders",
    "stockOpname": "Stock Count",
    "reports": "Reports",
    "staff": "Staff",
    "customers": "Customers",
    "settings": "Settings"
  },
  "checkout": {
    "cart": "Cart",
    "subtotal": "Subtotal",
    "discount": "Discount",
    "tax": "Tax",
    "total": "Total",
    "pay": "Pay",
    "cash": "Cash",
    "card": "Card",
    "ewallet": "E-Wallet",
    "change": "Change",
    "holdOrder": "Hold Order",
    "recallOrder": "Recall Order",
    "dineIn": "Dine In",
    "takeaway": "Takeaway",
    "tableNumber": "Table Number"
  }
}
```

**Usage in Components:**
```jsx
function CartPanel() {
  const { t } = useTranslation();
  return (
    <div>
      <h2>{t('checkout.cart')}</h2>        {/* "Keranjang" or "Cart" */}
      <span>{t('checkout.subtotal')}</span> {/* "Subtotal" */}
      <button>{t('checkout.pay')}</button>  {/* "Bayar" or "Pay" */}
    </div>
  );
}
```

**Language Switching:**
- Selected during **Setup Wizard** (Step 3: Pengaturan Awal)
- Changeable anytime in **Settings** (all roles can change)
- Instant switch — no page reload needed
- Saved per user preference in local storage

#### [NEW] `src/theme/variables.css` — Design System & Color Theme

**🎨 Color Theme: Indigo Blue**

Ionic CSS variables + Tailwind custom theme:

**Primary Palette (Indigo):**

| Shade | Hex | Usage |
|---|---|---|
| 50 | `#EEF2FF` | Backgrounds, hover states |
| 100 | `#E0E7FF` | Selected item background |
| 200 | `#C7D2FE` | Borders, dividers |
| 300 | `#A5B4FC` | Disabled state |
| 400 | `#818CF8` | Icons, links |
| 500 | `#6366F1` | Secondary buttons |
| **600** | **`#4F46E5`** | **← PRIMARY** |
| 700 | `#4338CA` | Hover on primary |
| 800 | `#3730A3` | Active/pressed |
| 900 | `#312E81` | Text on light bg |
| 950 | `#1E1B4B` | Sidebar background |

**Light & Dark Theme:**

| Element | Light Mode | Dark Mode |
|---|---|---|
| Background | `#FFFFFF` | `#0F172A` (Slate 900) |
| Surface/Card | `#F8FAFC` (Slate 50) | `#1E293B` (Slate 800) |
| Sidebar | `#1E1B4B` (Indigo 950) | `#0F0D2E` |
| Text Primary | `#1E293B` (Slate 800) | `#F1F5F9` (Slate 100) |
| Text Secondary | `#64748B` (Slate 500) | `#94A3B8` (Slate 400) |
| Border | `#E2E8F0` (Slate 200) | `#334155` (Slate 700) |
| Primary Button | `#4F46E5` text white | `#6366F1` text white |

**Semantic Colors:**

| Purpose | Color | Hex | Usage |
|---|---|---|---|
| ✅ Success | Emerald | `#059669` | Bayar, Lunas, Stock OK |
| ⚠️ Warning | Amber | `#D97706` | Stok rendah, Hold order |
| 🔴 Danger | Red | `#DC2626` | Batal, Void, Error |
| ℹ️ Info | Blue | `#2563EB` | Notifikasi, Info |

**POS-Specific Badge Colors:**

| Badge | Color | Hex |
|---|---|---|
| 🟢 Bayar / Lunas | Green | `#059669` |
| 🔴 Batal / Void | Red | `#DC2626` |
| 🟡 Hold / Pending | Amber | `#D97706` |
| 🔵 QRIS | Primary | `#4F46E5` |
| 🟠 COD | Orange | `#EA580C` |
| 🟣 DP (Uang Muka) | Purple | `#7C3AED` |
| 🩵 Transfer Bank | Teal | `#0D9488` |

**Ionic CSS Variables:**
```css
:root {
  --ion-color-primary: #4F46E5;
  --ion-color-primary-shade: #4338CA;
  --ion-color-primary-tint: #6366F1;
  --ion-color-success: #059669;
  --ion-color-warning: #D97706;
  --ion-color-danger: #DC2626;
  --ion-background-color: #FFFFFF;
  --ion-text-color: #1E293B;
}

.dark {
  --ion-background-color: #0F172A;
  --ion-text-color: #F1F5F9;
  --ion-color-primary: #6366F1;
}
```

**Typography**: Inter font from Google Fonts
**Dark Mode**: `class` strategy — toggle via Ionic `ion-palette-dark`
**Animations**: Custom keyframes for skeleton loading, slide-in, fade, pulse
**Responsive**: Mobile-first with Tailwind breakpoints (`sm:`, `md:`, `lg:`, `xl:`)

#### [NEW] Ionic UI Components (Built-in)
No separate installation needed — all included in `@ionic/react`:

| Component | POS Usage |
|---|---|
| `IonButton` | All action buttons, payment, confirm, cancel |
| `IonModal` | Payment modal, discount modal, confirmations |
| `IonActionSheet` | Quick actions, payment method selection |
| `IonSheet / IonDrawer` | Held orders drawer, mobile cart panel |
| `IonList + IonItem` | Inventory, vendor list, staff, PO items |
| `IonSegment` | Product categories, PO status tabs, report periods |
| `IonSelect` | Vendor picker, driver assignment, outlet selector |
| `IonBadge` | Order status, stock alerts, role badges |
| `IonToast` | Success/error notifications, low stock alerts |
| `IonSearchbar` | Product search, customer search |
| `IonDatetime` | Report date range, PO dates, expiry dates |
| `IonCard` | Product cards, dashboard stat cards, order cards |
| `IonInput` | Search, barcode input, forms |
| `IonLabel` | Form labels |
| `IonPopover` | Action menus, tooltips |
| `IonAvatar` | Staff avatar in header |
| `IonRefresher` | Pull-to-refresh on lists |
| `IonItemSliding` | Swipe actions on list items |
| `IonFab` | Floating action button for quick add |
| `IonLoading` | Loading states |
| `IonToggle` | Settings toggles (dark mode, tax inclusive) |
| `IonChip` | Filter tags, category pills |
| `IonInfiniteScroll` | Lazy-load long product lists |

#### [NEW] `src/app/setup/page.jsx` — Setup Wizard (First-Time Only)
Multi-step onboarding wizard shown on first launch:
- **Step 1 — Kategori Toko**: Choose 🛒 Retail or 🍽️ Restoran (visual cards with icons & descriptions)
- **Step 2 — Info Bisnis**: Store name, address, phone number, **logo upload**
  - Drag-and-drop or click-to-browse image upload
  - Image preview with crop & resize (max 512x512px)
  - Supports PNG, JPG, SVG formats
  - Stored in Firebase Cloud Storage (`stores/{storeId}/logo`)
  - Optional — uses default SkokPOS icon if skipped
- **Step 3 — Pengaturan Awal**: Tax rate (12% default), currency, language, printer setup
- **Step 4 — Outlet Pertama**: Create first outlet with name, address, and **optional outlet-specific logo**
- Saves config to Firestore + local storage, then redirects to `/checkout`
- Includes sample product data seeding based on chosen mode:
  - **Retail**: Indomie, Aqua, Beras, Minyak Goreng, Sabun, etc.
  - **Restoran**: Nasi Goreng, Mie Ayam, Es Teh, Kopi Susu, etc.

**🖼️ Where the Logo Appears:**

| Location | Description |
|---|---|
| 🧱 Struk / Receipt | Printed at top of thermal receipt (converted to ESC/POS bitmap) |
| 📋 Sidebar | Displayed above store name in the navigation sidebar |
| 🔐 Login / PIN Screen | Centered logo on staff login and PIN entry screen |
| 🚚 Customer Tracking | Shown on the public delivery tracking page |
| 📊 Reports Header | Displayed on exported PDF reports |

#### [NEW] `src/components/ui/LogoUpload.jsx` — Reusable Logo Upload Component
- Drag-and-drop zone with click fallback
- Real-time image preview (circular crop)
- Client-side resize to 512x512px before upload (saves bandwidth)
- Upload progress indicator
- Remove/replace logo button
- Used in: Setup Wizard (Step 2), Settings (Business Info), Outlet Management

#### [NEW] `src/components/layout/` — App Shell
- `Sidebar.tsx` — Collapsible navigation with **mode-aware menu items** (Kitchen & Table links hidden in Retail mode)
- `Header.tsx` — Search, notifications bell, user avatar, dark mode toggle, **active outlet selector**
- `TabBar.tsx` — `IonTabBar` bottom navigation for phone/tablet screens
- `AppShell.tsx` — Responsive layout: `IonSplitPane` (sidebar on desktop/tablet, tabs on mobile)

#### [NEW] `src/lib/storeMode.js` — Store Mode Engine
- `STORE_MODES` enum (retail, restaurant)
- `getModeFeatures(mode)` — Returns feature flags for the active mode
- `useModeFeatures()` — React hook to access feature flags in components
- Conditionally renders/hides UI elements based on mode

---

### Phase 2: Product Catalog & Checkout (Core POS)

#### [NEW] `src/stores/` — State Management (Zustand)
- `cartStore.js` — Cart state (items, quantities, discounts, tax, total)
- `productStore.js` — Product catalog with search/filter
- `authStore.js` — User session and role
- `settingsStore.js` — App settings (tax rate, currency, printer config, **storeMode**, active outlet)
- `shiftStore.js` — **🆕** Shift management state (open/close, starting cash, transactions)

#### [NEW] `src/app/(pos)/checkout/page.jsx` — Main POS Screen
The heart of the app — split-screen layout:
- **Left panel (70%)**: Product grid with categories, search bar, and barcode scanner input
- **Right panel (30%)**: Cart with running total, discount controls, and payment buttons
- Product cards with image, name, price, and quick-add
- Category tabs with horizontal scrolling
- Real-time search with debounce
- Quantity adjustment (+/−) in cart
- Hold order / Recall order functionality
- **🆕 Open Price / Custom Item**: Add item not in catalog with manual name & price
- **🆕 Weight-based input**: Products sold per-kg show weight input (e.g., 2.5 kg)
- **🆕 Multi-Price**: Auto-apply wholesale price when qty meets threshold
- **🆕 Bon/Hutang**: "Bayar Nanti" button for credit sales to known customers
- **🆕 Rounding**: Auto-round total to nearest Rp 100/500/1.000 (configurable)
- **🆕 Service Charge**: 🍽️ Restaurant: auto-add 5-10% service charge (separate from tax)
- **🍽️ Restaurant only**: Order type toggle (Dine-in / Takeaway / Delivery), table number input, modifier selection on product add

#### Cart Summary Display:
```
  Subtotal:           Rp  86.000
  Diskon (10%):      -Rp   8.600
  Service Charge (5%): Rp  3.870  ← 🍽️ Restoran only
  Ongkir:             Rp 10.000  ← Jika delivery
  PPN (12%):          Rp  9.752
  Pembulatan:        -Rp     22  ← Auto-round
  ══════════════════════════════
  TOTAL:              Rp 101.000
```

#### [NEW] `src/app/(pos)/checkout/components/`
- `ProductGrid.jsx` — Responsive product grid with category filtering
- `ProductCard.jsx` — Individual product card with variant/modifier support
- `CartPanel.jsx` — Shopping cart with line items + rounding + service charge
- `CartItem.jsx` — Single cart item with qty controls
- `PaymentModal.jsx` — Payment method selection (Cash, QRIS, Card, E-Wallet, Transfer, COD, Voucher)
- `DiscountModal.jsx` — Apply percentage or fixed discount
- `HeldOrdersDrawer.jsx` — View and recall held orders
- `BarcodeInput.jsx` — Invisible input field for barcode scanner
- `OpenPriceModal.jsx` — **🆕** Manual item entry (name + price) for uncatalogued items
- `WeightInput.jsx` — **🆕** Numeric weight input with kg unit for weight-based products
- `CreditSaleModal.jsx` — **🆕** Select customer → confirm bon/hutang sale
- `QRISDisplay.jsx` — **🆕** Display QRIS QR code for customer to scan
- `QuickCashButtons.jsx` — **🆕** Preset cash amounts: Uang Pas, 50K, 100K, 200K, 500K
- `ChangeDenomination.jsx` — **🆕** Breakdown kembalian per pecahan uang
- `VoucherInput.jsx` — **🆕** Input voucher code or redeem store credit/loyalty points
- `TipSelector.jsx` — **🆕** 🍽️ Optional tip: 5%, 10%, 15%, or custom amount
- `DPModal.jsx` — **🆕** Down payment partial payment with remaining balance tracking
- `PaymentRefInput.jsx` — **🆕** Reference number input for non-cash payments
- `BankTransferInfo.jsx` — **🆕** Show bank account details for transfer payment
- `OrderTypeSelector.jsx` — Dine-in / Takeaway / **🆕 Delivery** toggle
- `TableSelector.jsx` — **🍽️ Restaurant only**: Table number picker
- `ModifierPicker.jsx` — **🍽️ Restaurant only**: Add-on/topping selection modal

#### [NEW] Payment System Architecture

**Payment Methods:**

| Method | Icon | Tersedia | Ref # | Laci Kas |
|---|---|---|---|---|
| 💵 Tunai | Cash | Selalu | ❌ | ✅ Auto-open |
| 📱 QRIS | QR | Selalu | ✅ Auto | ❌ |
| 💳 Kartu Debit/Kredit | Card | Selalu | ✅ Required | ❌ |
| 📲 E-Wallet (GoPay/OVO/DANA) | Wallet | Selalu | ✅ Optional | ❌ |
| 🏦 Transfer Bank | Bank | Selalu | ✅ Required | ❌ |
| 📒 COD | Delivery | Hanya Delivery | ❌ | ❌ |
| 🎁 Voucher | Coupon | Jika ada voucher | ✅ Code | ❌ |
| 💎 Store Credit | Credit | Jika saldo > 0 | ❌ | ❌ |
| ⭐ Loyalty Points | Points | Jika poin > 0 | ❌ | ❌ |
| 💰 Uang Muka (DP) | DP | Selalu | ❌ | Depends |

**QRIS System:**
- Static QRIS: QR tetap yang bisa dicetak & tempel di kasir
- Dynamic QRIS: QR per transaksi dengan nominal tertanam
- Display QR di layar POS untuk customer scan
- Logo toko di tengah QR code
- Kasir konfirmasi "Sudah dibayar" (mock — real gateway di fase berikutnya)

**Quick Cash Flow:**
```
  Total: Rp 33.000
  ┌────────────┬────────────┐
  │ ✅ Uang Pas │ Rp 50.000  │
  ├────────────┼────────────┤
  │ Rp 100.000 │ Rp 200.000 │
  ├────────────┴────────────┤
  │   Manual: [________]    │
  ├─────────────────────────┤
  │  Kembalian: Rp 17.000   │
  │  = 1×10.000 + 1×5.000   │
  │    + 1×2.000             │
  └─────────────────────────┘
```

**Rounding Configuration:**

| Setting | Contoh (Total: Rp 33.040) | Result |
|---|---|---|
| Off | Rp 33.040 | No change |
| Ke Rp 100 (bawah) | Rp 33.000 | -Rp 40 |
| Ke Rp 100 (terdekat) | Rp 33.000 | -Rp 40 |
| Ke Rp 500 (bawah) | Rp 33.000 | -Rp 40 |
| Ke Rp 1.000 (bawah) | Rp 33.000 | -Rp 40 |

**Down Payment (DP) Flow:**
1. Checkout → pilih "Bayar DP" → set persentase (50%) atau nominal
2. Bayar DP dengan metode apapun (tunai/QRIS/kartu)
3. Order status: "DP Dibayar" → pesanan disiapkan
4. Pelunasan: customer datang / delivery → bayar sisa
5. Struk DP + Struk Pelunasan dicetak terpisah

**Cash Drawer:**
- Auto-open via ESC/POS command saat pembayaran tunai selesai
- Manual open: tombol "Buka Laci" (PIN admin required)
- Log setiap pembukaan laci di activity log

#### [NEW] `src/app/(pos)/shift/` — Shift Management (Buka/Tutup Kasir)
- **Buka Shift**:
  - Kasir input modal awal (starting cash)
  - Record start time, kasir name
  - Must open shift before first transaction
- **Selama Shift**:
  - Track all transactions (sales, refunds, voids)
  - Running cash balance = starting cash + cash sales − cash refunds
- **Tutup Shift**:
  - Input actual cash count (per denomination: Rp 100.000, 50.000, 20.000, etc.)
  - System vs actual comparison with variance
  - Add notes for discrepancies
  - Print shift summary on thermal printer
- **Serah Terima**: Transfer shift to another kasir without closing store
- **Shift History**: View past shifts with all details

#### [NEW] `src/app/(pos)/returns/page.jsx` — Retur, Refund & Void
- **Void Transaksi**: Cancel a recently completed transaction (requires Super Admin/Admin PIN approval)
- **Retur Barang**:
  - Search order by number or scan receipt barcode
  - Select items to return (partial or full)
  - Choose refund method: cash back, store credit, or exchange
  - Auto-update inventory (returned stock added back)
- **Retur Sebagian**: Return 1 of 5 items from an order
- **Log Retur**: Complete history of all returns/voids with reason and approver
- **Void time limit**: Configurable (e.g., void only within 15 minutes of sale)

#### [NEW] `src/lib/firebase/` — Firebase Configuration
- `config.js` — Firebase app initialization
- `firestore.js` — Firestore helpers with offline persistence
- `auth.js` — Authentication helpers
- `realtime.js` — Realtime Database for GPS tracking

#### [NEW] `src/lib/models/` — Data Models
```
Store:         { id, name, address, phone, logo, storeMode, taxRate, taxInclusive, language, createdAt }
Outlet:        { id, storeId, name, address, storeMode, modules, isActive }
Product:       { id, name, sku, barcode, categoryId, price, cost, wholesalePrice, wholesaleMinQty, unit, soldByWeight, expiryDate, image, variants[], modifiers[], isActive, stock, minStock, outletId, vendorId }
Order:         { id, orderNumber, items[], subtotal, discount, serviceCharge, tax, deliveryFee, rounding, tip, total, payments[], status, cashierId, customerId, outletId, orderType, tableNumber, shiftId, isCredit, isCOD, dpAmount, dpPaid, deliveryAddressId, deliveryNotes, scheduledTime, createdAt }
Payment:       { id, orderId, method, amount, refNumber, bankName, voucherCode, pointsUsed, status, createdAt }
Category:      { id, name, icon, color, sortOrder }
Modifier:      { id, name, price, group, isRequired }  // 🍽️ Restaurant only
Vendor:        { id, name, contactPerson, phone, email, address, notes, products[], isActive, outletId, createdAt }
PurchaseOrder: { id, poNumber, vendorId, outletId, items[], status, subtotal, tax, total, notes, createdBy, createdAt, sentAt, receivedAt }
POItem:        { productId, productName, qty, qtyReceived, unitPrice, subtotal }
StockOpname:   { id, outletId, items[], status, countedBy, approvedBy, createdAt, completedAt }
Shift:         { id, outletId, cashierId, startingCash, actualCash, expectedCash, variance, status, startedAt, closedAt, notes }
Return:        { id, orderId, items[], refundAmount, refundMethod, reason, approvedBy, createdBy, createdAt }
Credit:        { id, customerId, orderId, amount, paidAmount, remainingAmount, status, dueDate, outletId, createdAt }
ActivityLog:   { id, userId, action, target, details, outletId, timestamp }
Voucher:       { id, code, type, value, minOrder, maxUses, usedCount, validUntil, outletId, isActive }
StoreCredit:   { id, customerId, balance, transactions[], outletId, createdAt }
DeliveryAddress: { id, customerId, label, address, lat, lng, notes }
Delivery:      { id, orderId, driverId, status, attempts, codAmount, codCollected, codSettled, proofPhotoUrl, proofNotes, failReason, zoneId, fee, assignedAt, pickedUpAt, deliveredAt, failedAt }
DeliveryZone:  { id, outletId, name, type, coordinates, fee, isActive }
```

---

### Phase 3: Thermal Printing

#### [NEW] `src/lib/printer/`
- `escpos.js` — ESC/POS command builder (text formatting, alignment, barcode, QR code, cut paper)
- `usbPrinter.js` — WebUSB connection manager (discover, connect, print)
- `bluetoothPrinter.js` — Web Bluetooth fallback for wireless printers
- `receiptTemplate.js` — Receipt layout builder (mode-aware):

  **🛒 Retail Receipt:**
  ```
  ================================
          SKOKPOS
       Jl. Contoh No. 123
        Tel: 021-1234567
  ================================
  Kasir: Ahmad    03/06/2026 14:30
  No: INV-20260603-0001
  Outlet: Cabang Utama
  --------------------------------
  Indomie Goreng   x5  Rp  17.500
  Aqua 600ml       x3  Rp  12.000
  --------------------------------
  Subtotal:           Rp  29.500
  PPN (12%):          Rp   3.540
  ================================
  TOTAL:              Rp  33.040
  ================================
  Bayar (Tunai):      Rp  50.000
  Kembali:            Rp  16.960
  --------------------------------
       Terima Kasih!
  ================================
  ```

  **🍽️ Restaurant Receipt:**
  ```
  ================================
          SKOKPOS
       Jl. Contoh No. 123
        Tel: 021-1234567
  ================================
  Kasir: Ahmad    03/06/2026 14:30
  No: INV-20260603-0001
  Outlet: Cabang Utama
  Tipe: Dine-in | Meja: 5
  --------------------------------
  Nasi Goreng Spesial x2 Rp 50.000
    + Telur Ceplok     x2 Rp  6.000
    + Level Pedas 3
  Es Teh Manis        x3 Rp 30.000
  --------------------------------
  Subtotal:           Rp  86.000
  Diskon (10%):      -Rp   8.600
  PPN (12%):          Rp   9.288
  ================================
  TOTAL:              Rp  86.688
  ================================
  Bayar (Tunai):      Rp 100.000
  Kembali:            Rp  13.312
  --------------------------------
       Terima Kasih!
   Barang yang sudah dibeli
    tidak dapat dikembalikan
  ================================
  ```
- `kitchenTicket.js` — **🍽️ Restaurant only**: KOT/BOT template (order items with modifiers, large font, table number)

#### [NEW] `src/components/printer/`
- `PrinterSetup.jsx` — Printer discovery and pairing UI
- `PrintPreview.jsx` — On-screen receipt preview before printing
- `PrinterStatus.jsx` — Connection indicator in header

#### [NEW] Struk Digital via WhatsApp
- After payment, option: "Kirim struk ke WhatsApp?"
- Opens WhatsApp with pre-formatted receipt text (wa.me API)
- Includes: order number, items, total, payment method, store info
- Alternative: generate receipt as image and share

#### [NEW] Cetak Label Barcode
- Generate barcode (Code128/EAN13) for products without barcode
- Print barcode sticker labels on thermal printer
- Label includes: product name, price, barcode
- Batch print: select multiple products → print all labels at once
- Custom label size: 40×30mm, 50×25mm, 60×40mm

---

### Phase 4: Delivery Management & Live Tracking

#### [NEW] `src/app/(pos)/delivery/page.jsx` — Delivery Dashboard
- Kanban board layout: New → Preparing → Picked Up → En Route → Delivered
- Drag-and-drop order cards between columns
- Assign/reassign drivers to orders
- Real-time map showing all active drivers
- ETA calculations
- **🆕 COD badge** on order cards that are Cash on Delivery
- **🆕 Driver availability counter** in header ("3 driver aktif")
- **🆕 Scheduled orders** section showing upcoming deliveries

#### [NEW] `src/app/(pos)/delivery/components/`
- `DeliveryBoard.jsx` — Kanban columns with order cards
- `OrderCard.jsx` — Order summary card with status badge + COD tag
- `DriverAssignment.jsx` — Driver selection dropdown (shows only online drivers)
- `LiveMap.jsx` — Leaflet map with driver markers, delivery routes, and **zone overlay**
- `DeliveryTimeline.jsx` — Status history timeline
- `DeliveryFeeCalculator.jsx` — **🆕** Auto-calculate ongkir based on distance/zone
- `AddressPicker.jsx` — **🆕** Map pin + address autocomplete + saved addresses
- `BatchAssignment.jsx` — **🆕** Assign multiple orders to one driver trip
- `ScheduleSlotPicker.jsx` — **🆕** Time slot selector for scheduled deliveries
- `FailedDeliveryModal.jsx` — **🆕** Failed delivery reason + retry/return options
- `ProofOfDelivery.jsx` — **🆕** Photo capture + notes for delivery confirmation
- `CODReconciliation.jsx` — **🆕** Driver cash collection tracking & settlement

#### [NEW] `src/app/track/[orderId]/page.jsx` — Customer Tracking Page
- Public page (no auth required)
- Live map with animated driver marker
- Order details and status
- ETA with progress bar
- Delivery person name and contact
- Auto-refresh via Realtime Database listener
- **🆕 Multi-stop indicator** if driver has multiple deliveries

#### [NEW] `src/app/(driver)/driver/page.jsx` — Driver Mobile View
- Optimized for phone screens
- **🆕 Online/Offline toggle** at top of screen
- Current delivery with navigation
- **🆕 Multi-stop trip view** with all delivery points
- Order list queue
- Status update buttons (one-tap: "Picked Up", "En Route", "Delivered")
- **🆕 "Gagal Kirim" button** with reason selection
- **🆕 Photo capture** for proof of delivery
- **🆕 COD input** — input cash received from customer
- GPS broadcasting (sends location every 5 seconds to Realtime DB)

#### [NEW] `src/lib/tracking/`
- `gpsTracker.js` — Geolocation API wrapper with battery-efficient polling
- `locationSync.js` — Push GPS coordinates to Firebase Realtime DB
- `etaCalculator.js` — Simple distance-based ETA estimation
- `deliveryFee.js` — **🆕** Fee calculation engine (flat/per-km/zone-based)
- `zoneManager.js` — **🆕** Delivery zone polygon manager + radius check
- `autoAssign.js` — **🆕** Auto-assign nearest available online driver

#### [NEW] Ongkos Kirim (Delivery Fee) System
3 models, configurable per outlet:

| Model | Contoh | Cara Kerja |
|---|---|---|
| **Gratis** | Rp 0 | Semua pengiriman gratis |
| **Flat Rate** | Rp 10.000 | Ongkir tetap untuk semua jarak |
| **Per Kilometer** | Rp 3.000/km (min Rp 5.000) | Hitung jarak → kalikan tarif |
| **Zona** | Zona 1: Rp 5.000, Zona 2: Rp 10.000, Zona 3: Rp 20.000 | Berdasarkan zona pengiriman |

- **Free delivery threshold**: Gratis ongkir jika order ≥ Rp X (konfigurabel)
- Ongkir tampil sebagai line item di struk
- Ongkir masuk ke laporan keuangan

#### [NEW] Customer Address & Map Pin
- Multiple alamat tersimpan per pelanggan (Rumah, Kantor, Lainnya)
- Pin lokasi di peta (Leaflet map tap)
- Catatan alamat: "Lantai 3, pintu kiri", "Depan minimarket"
- Autocomplete pencarian alamat
- Last-used address auto-selected

#### [NEW] Cash on Delivery (COD) Tracking
- Order ditandai sebagai COD di delivery board
- Driver input jumlah uang diterima dari pelanggan
- Admin lihat total uang COD yang harus disetor per driver
- Setor kas: driver setor → admin konfirmasi → status lunas
- Laporan saldo COD outstanding per driver
- Alert: driver dengan saldo COD tinggi yang belum setor

#### [NEW] WhatsApp Notifications to Customer
Auto-send via wa.me API at each status change:

| Status | Template Pesan |
|---|---|
| ✅ Diterima | "Pesanan #INV-xxx sudah diterima, sedang diproses 👨‍🍳" |
| 🚚 Ditugaskan | "Driver [nama] sedang menuju toko untuk mengambil pesanan Anda" |
| 📦 Diantar | "Pesanan sedang dalam perjalanan! Lacak di: [link tracking]" |
| ✅ Terkirim | "Pesanan sudah sampai! Terima kasih 🙏" |
| ❌ Gagal | "Maaf, pengiriman gagal. Tim kami akan menghubungi Anda" |

#### [NEW] Proof of Delivery (POD)
- Driver capture photo saat serah terima barang
- Upload ke Firebase Cloud Storage, linked ke order
- Admin bisa lihat foto di detail order
- Catatan driver: "Dititipkan ke satpam", "Diterima langsung"
- Mencegah dispute "belum terima"

#### [NEW] Driver Online/Offline Status
- Toggle online/offline di driver mobile view
- Dashboard admin menampilkan driver available
- Auto-offline jika tidak ada aktivitas 30 menit
- Count "X driver aktif" di delivery board header
- Only online drivers shown in assignment dropdown

#### [NEW] Multi-Order per Trip (Batch Delivery)
- Assign 2-5 order ke 1 driver dalam 1 trip
- Urutkan stop berdasarkan jarak (nearest first algorithm)
- Peta multi-stop menampilkan semua titik delivery
- Status independen per order (satu delivered, lainnya masih en route)
- Driver view: numbered stop list with navigation

#### [NEW] Failed Delivery Handling
- Alasan gagal: pelanggan tidak ada, alamat salah, menolak, dll.
- Opsi setelah gagal:
  - **Retry**: Jadwalkan ulang pengiriman
  - **Return**: Kembalikan ke toko (stok kembali jika COD)
  - **Refund**: Proses refund jika sudah bayar
- Log semua percobaan pengiriman
- Counter: attempt 1/3

#### [NEW] Scheduled Delivery
- Saat checkout, pilih: "Kirim Sekarang" atau slot waktu (10:00-12:00, 12:00-14:00, dll.)
- Scheduled orders muncul di board pada waktu yang ditentukan
- Reminder notifikasi driver 15 menit sebelum jadwal
- Calendar view untuk scheduled deliveries

#### [NEW] Auto-Assign Driver
- Auto-assign ke driver online terdekat (berdasarkan GPS)
- Jika jarak sama, assign ke driver dengan order paling sedikit (least-busy)
- Manual override oleh admin kapan saja
- Notifikasi push ke driver saat order baru ditugaskan
- Timeout: jika driver tidak respond dalam 2 menit, auto-assign ke driver berikutnya

#### [NEW] Delivery Zone & Radius
- Set radius maksimum pengiriman per outlet (contoh: 15 km)
- Gambar zona pengiriman di peta (polygon / circle)
- Penolakan otomatis: "Maaf, alamat di luar area pengiriman kami"
- Zona terkait ongkir: Zona 1 = Rp X, Zona 2 = Rp Y
- Visual overlay di admin delivery map

---

### Phase 5: Inventory, Reports, Staff & Customers

#### [NEW] `src/app/(admin)/inventory/page.jsx` — Inventory Management
- Stock levels table with search and filters
- Low stock alerts (visual badges + notification)
- Stock adjustment (in/out with reason)
- Stock history log (full audit trail)
- Bulk import/export (CSV)
- **Min stock threshold** per product (triggers smart reorder)
- **🆕 Tanggal Kadaluarsa (Expiry Date)**:
  - Input expiry date per product batch
  - Alert: produk mendekati kadaluarsa (7 hari, 30 hari sebelum)
  - Color-coded: 🟢 OK | 🟡 Mendekati kadaluarsa | 🔴 Sudah kadaluarsa
  - FIFO enforcement: produk kadaluarsa terdekat dijual duluan
  - Report: daftar produk kadaluarsa untuk write-off
- **🆕 Produk Timbangan (Weight-based)**:
  - Per product: toggle "Jual per satuan" vs "Jual per kg"
  - Input berat di checkout: 2.5 kg × Rp 15.000 = Rp 37.500
  - Support unit: kg, gram, liter, meter
- **🆕 Harga Grosir / Multi-Price**:
  - Per product: set harga satuan + harga grosir + minimum qty grosir
  - Example: Indomie Rp 3.500/pcs, Rp 3.000/pcs jika beli ≥ 40 pcs
  - Auto-switch price at checkout when qty threshold met
  - Optional: harga per tier pelanggan (Regular/Silver/Gold/Platinum)

#### [NEW] `src/app/(admin)/vendors/page.jsx` — Vendor / Supplier Management
- Vendor list with search, filter by status (active/inactive)
- Add/edit vendor: name, contact person, phone, email, address, notes
- **Link products to vendor** — which vendor supplies which products
- Vendor performance: total purchases, last order date, average delivery time
- Purchase history per vendor
- Quick action: "Buat PO" (create Purchase Order) from vendor page

#### [NEW] `src/app/(admin)/purchase-orders/page.jsx` — Purchase Orders (PO)
- PO list with status tabs: Semua | Draft | Dikirim | Diterima | Dibatalkan
- **Create PO**:
  - Select vendor → auto-populate vendor’s products
  - Add items with qty and unit price
  - Auto-calculate subtotal & total
  - Add notes/catatan
  - PO number format: `PO-YYYYMMDD-NNNN`
- **PO Status Flow**:
  ```
  Draft → Dikirim → Diterima (Sebagian/Penuh) → Selesai
                → Dibatalkan
  ```
- **Send PO to vendor**:
  - Share via WhatsApp (formatted text message)
  - Export as PDF (with store logo)
  - Copy link
- **Receive goods** — opens Goods Receiving flow

#### [NEW] `src/app/(admin)/purchase-orders/receive/[poId]/page.jsx` — Goods Receiving
- View PO items with expected qty
- Input actual received qty per item
- Mark items: Diterima Penuh | Diterima Sebagian | Tidak Diterima
- **Auto-update inventory** on confirm
- Option to create new PO for remaining items (partial delivery)
- Print receiving slip on thermal printer

#### [NEW] `src/app/(admin)/stock-opname/page.jsx` — Stock Opname (Physical Count)
- Start new stock opname session
- Scan barcode or search product → input physical count
- **System vs Physical comparison** with variance column
- Color-coded: 🟢 Match | 🟡 Minor difference | 🔴 Major difference
- Approve adjustments (Super Admin / Admin only)
- Auto-generate adjustment log with reason "Stock Opname"
- History of past opname sessions

#### [NEW] Smart Reorder System
- Dashboard widget: "Produk Perlu Restock" (Products Need Restock)
- When `stock <= minStock` → product appears in alert list
- One-click: "Buat PO Otomatis" → generates draft PO grouped by vendor
- Suggested qty = `(minStock * 2) - currentStock` (configurable multiplier)
- Notification push for low stock alerts

#### [NEW] `src/app/(admin)/reports/page.jsx` — Reports & Analytics
Comprehensive reporting suite with date range picker and export options:

**📊 Dashboard / Ringkasan Harian:**
- Revenue today vs yesterday (% change)
- Order count, average order value
- Top 5 products, low stock count, pending POs

**💰 Laporan Penjualan (Sales):**
- Penjualan harian / mingguan / bulanan (line chart)
- Penjualan per produk (best sellers table)
- Penjualan per kategori (donut chart)
- Penjualan per outlet (comparison)
- Penjualan per kasir (staff performance)
- Penjualan per jam (hourly heatmap — identify peak hours)
- Penjualan per metode bayar (cash vs card vs e-wallet)
- Penjualan per tipe order (🍽️ dine-in vs takeaway vs delivery)
- Trend penjualan (weekly/monthly trend)

**📦 Laporan Inventaris (Inventory):**
- Stok saat ini (all products with current level)
- Stok rendah & habis (below threshold)
- Pergerakan stok (in/out history per product)
- Nilai inventaris (total value = stock × cost price)
- Stok opname history (past sessions & variances)

**🏦 Laporan Pembelian (Purchase):**
- Pembelian per vendor (spending breakdown)
- Riwayat PO (all purchase orders with status)
- Harga beli trend (cost price changes over time)
- Vendor performance (delivery time, fulfillment rate)

**👥 Laporan Pelanggan (Customer):**
- Pelanggan terbanyak (top spenders)
- Pelanggan baru per periode
- Loyalty points summary (earned/redeemed)
- Frekuensi kunjungan

**👨‍💼 Laporan Karyawan (Staff):**
- Penjualan per kasir
- Jumlah transaksi per kasir
- Driver performance (🚚 deliveries count, avg time)

**🚚 Laporan Pengiriman (Delivery):**
- Delivery per hari (count & revenue)
- Delivery per driver
- Rata-rata waktu kirim

**💸 Laporan Keuangan Sederhana:**
- Profit & Loss sederhana (revenue − cost = gross profit)
- Pajak terkumpul (PPN collected per period)
- Diskon diberikan (total discounts)
- Ringkasan kas (cash in register, open/close shift)

**📤 Export Formats:**
- PDF (with store logo)
- CSV / Excel
- Share via WhatsApp (daily summary)
- Thermal print (end-of-day summary on receipt printer)

**🆕 Laporan Shift (Shift Reports):**
- Ringkasan per shift: penjualan, refund, void, kas masuk/keluar
- Selisih kas (expected vs actual)
- Riwayat shift per kasir

**🆕 Laporan Retur & Void:**
- Daftar semua retur dan void dengan alasan
- Total nilai refund per periode
- Produk paling sering di-retur

**🆕 Laporan Hutang (Credit/Bon):**
- Total piutang outstanding
- Hutang per pelanggan
- Hutang jatuh tempo
- Pembayaran hutang per periode

**🆕 Laporan Kadaluarsa:**
- Produk mendekati kadaluarsa (7/30 hari)
- Produk sudah kadaluarsa (perlu write-off)
- Nilai kerugian produk kadaluarsa

#### [NEW] Laporan Otomatis / Scheduled Reports
- **Laporan Harian Otomatis**: Auto-kirim ringkasan ke WhatsApp owner jam 22:00
- **Laporan Mingguan**: Ringkasan mingguan tiap Senin pagi
- **Konfigurasi**: Super Admin pilih laporan mana yang dikirim otomatis
- **Format**: Teks ringkas via WhatsApp API (wa.me)

#### [NEW] Dashboard KPI Real-time
Real-time KPI cards yang auto-update setiap transaksi:

| KPI Card | Value | Visual |
|---|---|---|
| Revenue Hari Ini | Rp 5.234.000 | Counter animasi, vs kemarin (+12%) |
| Jumlah Transaksi | 47 orders | vs kemarin |
| Rata-rata Order | Rp 111.362 | AOV (Average Order Value) |
| Gross Profit | Rp 1.892.000 | Revenue − COGS |
| Target vs Aktual | 75% | Progress bar animasi |
| YoY Growth | +23% | vs bulan yang sama tahun lalu |
| MoM Growth | +8% | vs bulan lalu |

**Target System:**
- Super Admin set target harian/bulanan per outlet
- Progress bar di dashboard: hijau (≥100%), kuning (50-99%), merah (<50%)
- Notifikasi saat target tercapai 🎉

#### [NEW] Laporan Perbandingan (Comparison Reports)
| Comparison | Description |
|---|---|
| Periode vs Periode | Minggu ini vs minggu lalu, bulan ini vs bulan lalu |
| Outlet vs Outlet | Side-by-side perbandingan kinerja antar outlet |
| Produk Growth | Produk mana yang naik / turun (bar chart +/−) |
| Year over Year | Penjualan tahun ini vs tahun lalu (line chart overlay) |
| Kasir vs Kasir | Perbandingan kinerja antar kasir |

Visual: dual-axis line chart, stacked bar chart, delta indicators (▲ +12%, ▼ −5%)

#### [NEW] Laporan Pembayaran (Payment Reports)
| Report | Visual |
|---|---|
| Distribusi Metode Bayar | Pie/donut chart: % tunai vs QRIS vs kartu vs e-wallet |
| Trend Cashless | Line chart: cashless ratio per minggu (makin naik?) |
| Rekonsiliasi COD | Tabel: driver → uang COD terkumpul → sudah setor → belum setor |
| Laporan DP (Uang Muka) | Daftar DP yang belum dilunasi, aging |
| Laporan Tips (🍽️) | Total tips per kasir/hari, distribusi tips |
| Voucher Usage | Voucher terpakai vs sisa, revenue impact, top vouchers |
| Rounding Impact | Total pembulatan per periode (berapa uang yang "hilang" karena bulatkan ke bawah) |

#### [NEW] Laporan Kerugian / Shrinkage Report
```
┌─────────────────────────────────────────────┐
│  📉 LAPORAN KERUGIAN — Juni 2026            │
│                                              │
│  Selisih Stok Opname:     Rp    450.000     │
│  Void & Retur:            Rp    280.000     │
│  Produk Kadaluarsa:       Rp    320.000     │
│  Total Diskon Diberikan:  Rp  1.200.000     │
│  Pembulatan:              Rp     15.600     │
│  ═══════════════════════════════════════     │
│  TOTAL SHRINKAGE:         Rp  2.265.600     │
│  (2.3% dari revenue)                        │
│                                              │
│  Benchmark: < 3% = BAIK ✅                  │
│             3-5% = PERLU PERHATIAN ⚠️        │
│             > 5% = KRITIS 🔴                │
└─────────────────────────────────────────────┘
```

#### [NEW] Smart Alerts & Notifications
Proactive alerts — owner tidak perlu cek manual:

| Alert | Trigger | Action |
|---|---|---|
| 🎉 Target Tercapai | Revenue melewati target harian | Push notification + WA |
| ⚠️ Revenue Drop | Revenue hari ini 30% lebih rendah dari rata-rata | Push notification |
| 🔴 Best Seller Habis | Produk terlaris stok = 0 | Push + badge di sidebar |
| 💰 Margin Alert | Produk dijual di bawah cost price | Badge di produk |
| 🏦 Cash Variance | Selisih kas > Rp 50.000 saat tutup shift | Notifikasi ke admin |
| ⏰ Expiry Alert | Produk kadaluarsa dalam 7 hari | Daily morning alert |
| 📒 Hutang Jatuh Tempo | Hutang pelanggan sudah lewat due date | Alert + list |

**Notification channels**: In-app toast, push notification (Capacitor), WhatsApp

#### [NEW] ABC Analysis (Inventory Classification)
Klasifikasi otomatis berdasarkan kontribusi revenue:

| Class | Criteria | Action |
|---|---|---|
| **A** ⭐ | 20% produk = 80% revenue | Selalu ready stock, reorder tinggi |
| **B** | 30% produk = 15% revenue | Stock level moderate |
| **C** | 50% produk = 5% revenue | Evaluasi: pertahankan atau hapus? |

- Auto-calculated dari data penjualan 30 hari terakhir
- Visual: bar chart sorted by revenue contribution
- Recommendation: "Produk X bisa dihapus, hanya 0.1% revenue"
- Update otomatis setiap minggu

#### [NEW] Laporan Pajak (Tax Report)
Untuk keperluan pelaporan SPT:

| Report | Description |
|---|---|
| PPN Terkumpul | Total PPN per bulan (Rp X dari Y transaksi) |
| Service Charge | Total service charge terkumpul (restoran) |
| Revenue Kena Pajak | Netto setelah diskon, sebelum pajak |
| Rekap Pajak Bulanan | Tabel: bulan → revenue → PPN → service charge |
| Export Pajak | CSV format siap import ke e-Faktur / e-SPT |

#### [NEW] Owner Mobile Dashboard
Optimized untuk pemilik usaha yang cek dari HP:

| Feature | Description |
|---|---|
| Quick Stats Card | Revenue, orders, profit — swipe refresh |
| Push Notification | "Revenue hari ini Rp 5.2 juta 🎉" jam 22:00 (Capacitor Push) |
| Chart Mini | Sparkline chart 7 hari terakhir |
| Compare Yesterday | Badge: ▲ +12% vs kemarin |
| Multi-outlet Switcher | Quick switch antar outlet di HP |

#### [NEW] Forecasting (Prediksi)
Simple forecasting berdasarkan data historis:

| Forecast | Method | Output |
|---|---|---|
| Prediksi Revenue | Moving average 30 hari | "Estimasi bulan depan: Rp 85 juta" |
| Prediksi Stok Habis | Daily sales rate × current stock | "Indomie habis dalam 5 hari" |
| Hari Terlaris | Historical day-of-week analysis | "Sabtu rata-rata 2× Senin" |
| Jam Terlaris | Hourly heatmap analysis | "Peak: 11:30-13:00 dan 17:00-19:00" |
| Seasonal Trend | Month-over-month pattern | "Desember biasanya +40% (Natal)" |

#### [NEW] Export & Sharing Enhanced
| Feature | Description |
|---|---|
| PDF Branded | Laporan PDF dengan logo toko, warna brand, header/footer |
| CSV / Excel | Semua laporan bisa export ke CSV |
| Google Sheets | Auto-push data ke Google Sheets (opsional, via Cloud Functions) |
| Email Report | Kirim laporan via email selain WhatsApp |
| Print A4 | Cetak laporan A4 dari browser (CSS @media print) |
| Share Image | Screenshot chart → share ke WA/social media |
| Scheduled Export | Auto-export laporan bulanan ke format tertentu |

#### [NEW] `src/app/(admin)/activity-log/page.jsx` — Activity Log / Audit Trail
- Log semua aktivitas: edit produk, hapus pesanan, ubah harga, void, retur, login/logout, perubahan settings
- Filter: per user, per tipe aksi, per tanggal
- **Immutable**: Log tidak bisa dihapus atau diedit oleh siapapun
- Detail: siapa, apa, kapan, dari nilai apa ke nilai apa
- Export ke CSV untuk audit
- Hanya bisa dilihat oleh Super Admin & Admin

#### [NEW] `src/app/(pos)/credit/page.jsx` — Bon / Hutang (Credit Sales)
- **Daftar Piutang**: Semua hutang pelanggan yang belum lunas
- **Buat Hutang**: Saat checkout, pilih "Bayar Nanti" → linked ke pelanggan
- **Bayar Hutang**: Pelanggan bayar sebagian atau lunas
- **Batas Kredit**: Set limit hutang per pelanggan (configurable by Super Admin)
- **Jatuh Tempo**: Set tanggal jatuh tempo per transaksi hutang
- **Reminder**: Notifikasi hutang mendekati/melewati jatuh tempo
- **Status**: Belum Bayar → Bayar Sebagian → Lunas
- **Riwayat**: Riwayat pembayaran hutang per pelanggan

#### [NEW] `src/app/(admin)/staff/page.jsx` — Staff Management
- Staff list with roles and status
- Add/edit staff with role assignment
- Role-based access control (**6 roles**):
  - **🔑 Super Admin**: Full access — store mode, outlet management, tax settings, staff roles, data reset. Created automatically during Setup Wizard.
  - **👔 Admin**: Reports, inventory, customers, staff list (cannot change store mode, outlets, or tax)
  - **💳 Kasir (Cashier)**: POS checkout, order management, customer lookup. **Sub-level: Senior / Junior**
  - **📦 Gudang (Warehouse)**: Terima barang, stock opname, kelola inventaris. No checkout, no reports, no settings
  - **🍳 Dapur (Kitchen)**: Kitchen display, order status updates (🍽️ Restaurant only)
  - **🚚 Driver**: Delivery view, GPS tracking
- PIN-based quick login (for shift changes at POS terminal)
- Only Super Admin can assign/change roles

**Role Permission Matrix**:

| Feature | 🔑 Super Admin | 👔 Admin | 💳 Kasir | 📦 Gudang | 🍳 Dapur | 🚚 Driver |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| Ganti kategori toko | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Kelola outlet | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Ubah tarif pajak | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Kelola staff & roles | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Hapus data / reset | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Backup / restore | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Buat role custom | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Lihat laporan | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Kelola inventaris | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| **Kelola vendor** | ✅ | ✅ | ❌ | 👁️ Read | ❌ | ❌ |
| **Buat & kelola PO** | ✅ | ✅ | ❌ | 👁️ Read | ❌ | ❌ |
| **Terima barang (receiving)** | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| **Stock opname** | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| Kelola pelanggan | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| Checkout / POS | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| Kitchen Display | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |
| Delivery Board | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| Driver View | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Printer & receipt settings | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Theme & language | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

#### [NEW] Approval Chain (Rantai Persetujuan)
PIN verification untuk aksi sensitif:

| Action | Kasir Langsung? | Perlu Approve | Approver |
|---|---|---|---|
| Diskon ≤ 10% | ✅ (Senior only) | ❌ | — |
| Diskon > 10% | ❌ | ✅ PIN | Admin / Super Admin |
| Diskon > 25% | ❌ | ✅ PIN | Super Admin only |
| Void transaksi | ❌ | ✅ PIN | Admin / Super Admin |
| Retur / refund | ❌ | ✅ PIN | Admin / Super Admin |
| Buka laci kas manual | ❌ | ✅ PIN | Admin / Super Admin |
| Ubah harga produk | ❌ | ✅ PIN | Admin / Super Admin |
| Hapus produk | ❌ | ✅ PIN | Super Admin only |
| Cancel order setelah print | ❌ | ✅ PIN | Admin / Super Admin |

```
Flow: Kasir tap "Void" → IonModal "Masukkan PIN Admin"
      → Admin input PIN → Approved → Void tercatat di activity log
      → Log: "Void oleh Kasir Ani, disetujui oleh Admin Budi"
```

#### [NEW] Session & Security
| Feature | Description |
|---|---|
| Auto-logout | Logout otomatis setelah X menit tidak aktif (konfigurabel: 15/30/60 min) |
| Concurrent Session Limit | 1 akun hanya bisa login di 1 device (cegah sharing PIN) |
| Login History | Log: siapa login, kapan, dari device apa |
| Failed Login Lock | 5× salah PIN → akun terkunci 15 menit |
| Force Logout | Super Admin bisa paksa logout staff dari remote |
| Device Binding | Opsional: kasir hanya bisa login dari device tertentu |
| Password Policy | PIN minimal 4 digit, Super Admin minimal 6 digit |

#### [NEW] Kasir Sub-Permissions (Senior / Junior)
Tidak semua kasir sama — configurable per staff:

| Permission | 💳 Kasir Senior | 💳 Kasir Junior |
|---|:---:|:---:|
| Checkout | ✅ | ✅ |
| Berikan diskon ≤ 10% | ✅ | ❌ (perlu approve) |
| Hold order | ✅ | ✅ |
| Retur (dengan approve) | ✅ | ❌ |
| Lihat pelanggan | ✅ | ✅ |
| Buka laci kas manual | ✅ (PIN sendiri) | ❌ |
| Lihat laporan shift sendiri | ✅ | ❌ |
| Ubah jumlah item setelah print | ✅ (PIN) | ❌ |

Implementasi: Tambahkan field `permissionLevel: 'senior' | 'junior'` di staff model.

#### [NEW] Staff Profile & Schedule
| Feature | Description |
|---|---|
| Foto Profil | Upload foto staff (tampil di header & activity log) |
| Jadwal Kerja | Tentukan shift per staff (pagi 06-14 / siang 14-22 / malam 22-06) |
| Hari Libur | Tandai hari libur per staff |
| Status Aktif/Non-aktif | Non-aktifkan akun tanpa hapus (cuti, resign) |
| Riwayat Kinerja | Ringkasan: total transaksi, revenue dihasilkan, rata-rata per transaksi |

#### [NEW] Serah Terima Shift (Shift Handover)
| Feature | Description |
|---|---|
| Transfer Shift | Kasir A serah terima ke Kasir B tanpa tutup toko |
| Hitung Kas Bersama | Kas dihitung oleh Kasir outgoing & incoming, keduanya confirm |
| Catatan Serah Terima | "Ada 2 retur pending, 1 COD belum setor" |
| Log Handover | Tercatat di activity log: siapa → siapa, waktu, selisih kas |

#### [NEW] Staff Performance Dashboard
| Metric | Visual |
|---|---|
| Revenue per Kasir | Bar chart ranking |
| Transaksi per Shift | Counter per shift |
| Average Order Value | AOV per kasir |
| Void/Retur Rate | % per kasir (🔴 jika > 5%) |
| Kecepatan Checkout | Rata-rata waktu scan → bayar |
| Delivery per Driver | Jumlah & waktu rata-rata |
| Rating Driver | Sukses/Gagal delivery ratio |

#### [NEW] Custom Role Builder (🔑 Super Admin Only)
| Feature | Description |
|---|---|
| Buat Role Baru | Nama role custom (misal: "Supervisor", "Pramuniaga") |
| Pilih Permissions | Checklist granular permissions |
| Clone Role | Duplikat role existing → modifikasi |
| Per-outlet Role | Role berbeda per outlet |
| Max 10 custom roles | Batas agar tidak terlalu kompleks |

#### [NEW] Time-Based Access
| Feature | Description |
|---|---|
| Jam Kerja | Kasir A hanya bisa login 08:00-16:00 |
| Akses Weekend | Staff tertentu hanya boleh akses di hari kerja |
| Holiday Block | Block login pada tanggal tertentu |
| Overtime Alert | Notifikasi jika staff login di luar jadwal |

#### [NEW] Role-Based Notifications
| Role | Notifikasi Yang Diterima |
|---|---|
| 🔑 Super Admin | Semua alerts + revenue reports + security alerts |
| 👔 Admin | Stok rendah, PO, shift variance, expiry, void/retur alerts |
| 💳 Kasir | Order delivery baru, shift reminder, target reminder |
| 📦 Gudang | Barang datang, PO approved, stock opname reminder |
| 🍳 Dapur | Order baru, modifier note, rush order |
| 🚚 Driver | Delivery assignment, route update, COD reminder |

#### [NEW] `src/app/(admin)/customers/page.jsx` — Customer Database
- Customer list with search
- Purchase history per customer
- Loyalty points system (earn points per purchase, redeem for discounts)
- Customer groups/tiers (Regular, Silver, Gold, Platinum)

#### [NEW] `src/app/(pos)/kitchen/page.jsx` — Kitchen Display System (KDS)
- Full-screen order queue
- Color-coded by priority/wait time (green → yellow → red)
- One-tap "Done" to mark items as prepared
- Audio notification for new orders
- Auto-dismiss completed orders after 30 seconds

---

### Phase 6: Settings & Configuration

#### [NEW] `src/app/(admin)/settings/page.jsx` — Settings Page
- **🔒 Super Admin Only**:
  - Kategori Toko: Switch between 🛒 Retail and 🍽️ Restoran mode (per outlet)
  - Outlet Management: Add/edit/switch outlets, set mode per outlet
  - Tax Settings: Rate, inclusive/exclusive toggle
  - Backup/Restore: Export/import data
  - **🔧 Kelola Modul (Module Visibility)**: Show/hide modules per outlet
- **👔 Admin + Super Admin**:
  - Business Info: Store name, address, phone, logo
  - Receipt Settings: Header, footer, custom text
  - Printer Settings: Connection type, test print
  - Notification Settings: Sound, desktop notifications
- **🔓 All Roles**:
  - Theme: Light/Dark mode, accent color
  - Language: Bahasa Indonesia / English switch

#### [NEW] Module Visibility Management (🔑 Super Admin Only)
Super Admin can show/hide entire modules to simplify the UI for their business needs.
Config is saved **per outlet** in Firestore.

**Toggleable Modules:**

| Module | Default (🛒 Retail) | Default (🍽️ Restoran) | What Happens When Hidden |
|---|:---:|:---:|---|
| 🚚 **Pengiriman (Delivery)** | ✅ On | ✅ On | Delivery menu, driver view, tracking removed |
| 🏦 **Vendor / Supplier** | ✅ On | ✅ On | Vendor page hidden from sidebar |
| 📝 **Pesanan Pembelian (PO)** | ✅ On | ✅ On | PO & receiving pages hidden |
| 📋 **Stok Opname** | ✅ On | ✅ On | Stock count page hidden |
| 👥 **Pelanggan & Loyalty** | ✅ On | ✅ On | Customer page hidden, loyalty disabled |
| 🍳 **Dapur / KDS** | ❌ Off | ✅ On | Kitchen display & KOT printing hidden |
| 🍽️ **Meja / Table** | ❌ Off | ✅ On | Table number input hidden from checkout |
| 🌟 **Modifier / Add-on** | ❌ Off | ✅ On | Modifier picker hidden from checkout |
| 📊 **Laporan Lanjutan** | ✅ On | ✅ On | Advanced reports hidden (keeps dashboard) |
| 🔔 **Smart Reorder** | ✅ On | ✅ On | Auto-reorder suggestions disabled |

**How it works:**
```javascript
// src/stores/settingsStore.js
const moduleVisibility = {
  delivery: true,
  vendors: true,
  purchaseOrders: true,
  stockOpname: true,
  customers: true,
  kitchen: false,      // Auto-set by store mode
  tables: false,       // Auto-set by store mode
  modifiers: false,    // Auto-set by store mode
  advancedReports: true,
  smartReorder: true,
};
```

**UI in Settings:**
```
┌─────────────────────────────────────────┐
│  🔧 Kelola Modul                        │
│                                         │
│  Pilih modul yang aktif untuk outlet ini │
│                                         │
│  🚚 Pengiriman          [█████] ON     │
│  🏦 Vendor / Supplier   [█████] ON     │
│  📝 Pesanan Pembelian   [█████] ON     │
│  📋 Stok Opname         [█████] ON     │
│  👥 Pelanggan & Loyalty [█████] ON     │
│  🍳 Dapur / KDS         [─────] OFF    │
│  🍽️ Meja / Table        [─────] OFF    │
│  🌟 Modifier / Add-on   [─────] OFF    │
│  📊 Laporan Lanjutan    [█████] ON     │
│  🔔 Smart Reorder       [█████] ON     │
│                                         │
│  ⚠️ Beberapa modul otomatis aktif        │
│  berdasarkan kategori toko               │
│                                         │
│         [ Simpan Pengaturan ]            │
└─────────────────────────────────────────┘
```

**Behavior:**
- Hidden modules are **removed from sidebar** navigation
- Hidden module pages return **redirect to /checkout** if accessed directly
- When store mode changes (Retail → Restoran), restaurant modules auto-enable but Super Admin can still turn them off
- **Checkout** and **Settings** cannot be hidden (always required)
- Module visibility is stored in Firestore under `outlets/{outletId}/modules`

---

## Project File Structure

```
skokpos/
├── android/                   # 🤖 Capacitor Android project (auto-generated)
├── ios/                       # 🍎 Capacitor iOS project (auto-generated)
├── public/
│   ├── manifest.json          # PWA manifest
│   ├── icons/                 # App icons (192px, 512px, adaptive)
│   └── sounds/                # Notification sounds
├── src/
│   ├── App.tsx                # Root app with IonApp + IonReactRouter
│   ├── routes.tsx             # All route definitions (React Router)
│   ├── theme/
│   │   ├── variables.css      # Ionic CSS variables (colors, fonts)
│   │   └── tailwind.css       # Tailwind base + custom styles
│   ├── pages/                 # All page components
│   │   ├── Setup.tsx          # First-time setup wizard
│   │   ├── pos/
│   │   │   ├── Checkout.tsx   # Main POS checkout (mode-aware)
│   │   │   ├── Shift.tsx      # Shift management (open/close kasir)
│   │   │   ├── Returns.tsx    # Return, refund & void
│   │   │   ├── Credit.tsx     # Bon/hutang management
│   │   │   ├── Delivery.tsx   # Delivery management
│   │   │   └── Kitchen.tsx    # 🍽️ Kitchen Display System
│   │   ├── admin/
│   │   │   ├── Inventory.tsx  # Inventory + expiry tracking
│   │   │   ├── Vendors.tsx    # Vendor/supplier management
│   │   │   ├── PurchaseOrders.tsx # PO + goods receiving
│   │   │   ├── StockOpname.tsx # Physical stock count
│   │   │   ├── Reports.tsx    # Analytics + scheduled reports
│   │   │   ├── ActivityLog.tsx # Audit trail
│   │   │   ├── Staff.tsx      # Staff management
│   │   │   ├── Customers.tsx  # Customer database
│   │   │   └── Settings.tsx   # App settings + module visibility
│   │   ├── driver/
│   │   │   └── DriverView.tsx # Driver mobile view
│   │   └── track/
│   │       └── TrackOrder.tsx # Public tracking page
│   ├── components/
│   │   ├── layout/            # AppShell, Sidebar, Header, TabBar
│   │   ├── pos/               # POS-specific components (cart, payment, QRIS, etc.)
│   │   ├── delivery/          # Delivery & tracking components
│   │   ├── printer/           # Thermal print + barcode label components
│   │   ├── setup/             # Setup wizard step components
│   │   └── charts/            # Chart components (Recharts)
│   ├── lib/
│   │   ├── firebase/          # Firebase config & helpers
│   │   ├── i18n/              # Internationalization
│   │   │   ├── config.ts      #    Language config
│   │   │   ├── id.json        #    🇮🇩 Bahasa Indonesia
│   │   │   ├── en.json        #    🇬🇧 English
│   │   │   ├── useTranslation.ts #  React hook
│   │   │   └── formatters.ts  #    Number/date/currency formatters
│   │   ├── capacitor/         # Capacitor plugin wrappers
│   │   │   ├── camera.ts      #    @capacitor/camera wrapper
│   │   │   ├── geolocation.ts #    @capacitor/geolocation wrapper
│   │   │   └── barcodeScanner.ts # @capacitor-community/barcode-scanner
│   │   ├── printer/           # ESC/POS printing + barcode label engine
│   │   ├── tracking/          # GPS & delivery tracking
│   │   ├── models/            # Data models & validation
│   │   ├── storeMode.ts       # Store mode engine & feature flags
│   │   ├── utils.ts           # cn() helper for Tailwind class merging
│   │   └── hooks/             # Custom React hooks
│   ├── stores/                # Zustand state stores (incl. shiftStore)
│   └── styles/                # Additional custom styles
├── capacitor.config.ts        # Capacitor configuration
├── ionic.config.json          # Ionic CLI configuration
├── tailwind.config.ts         # Tailwind CSS configuration
├── vite.config.ts             # Vite build configuration (Ionic uses Vite)
├── .env.local                 # Firebase config (gitignored)
└── package.json
```

---

## Implementation Phases & Timeline

| Phase | Scope | Est. Effort |
|---|---|---|
| **Phase 1** | Project setup, design system, app shell, **setup wizard**, i18n | Foundation |
| **Phase 2** | Product catalog, checkout, **QRIS, quick cash, rounding, service charge, voucher, DP, tips, bank transfer, cash drawer**, shift, retur/void, bon/hutang, multi-price, open price, weight-based | Core POS |
| **Phase 3** | Thermal printing, receipts, **WA digital receipt, barcode label printing**, KOT | Printing |
| **Phase 4** | Delivery board, live tracking, driver app, **ongkir, COD, WA notif, POD, zones, batch, auto-assign** | Delivery |
| **Phase 5** | Inventory, **expiry tracking**, vendors, PO, stock opname, **activity log, scheduled reports**, staff, customers, KDS | Management |
| **Phase 6** | Settings, **module visibility**, PWA optimization, final polish | Polish |

> [!TIP]
> I recommend building in this order so you can test the core POS flow (checkout → print receipt) as early as Phase 3, and add complexity progressively.

---

## Verification Plan

### Automated Tests
- Run `npm run build` to verify no build errors
- Run `npm run lint` for code quality
- Lighthouse PWA audit (installability, offline capability, performance)

### Manual Verification
- **Setup Wizard**: Fresh load → complete setup as Retail → verify correct features visible
- **Mode Switch**: Settings → change to Restaurant → verify modifiers, KDS, table number appear
- **Checkout Flow**: Add products → apply discount → process payment → print receipt
- **Shift Management**: Open shift → process sales → close shift → verify cash reconciliation
- **Retur/Void**: Complete sale → void within time limit → verify stock returns and refund processed
- **Bon/Hutang**: Checkout with "Bayar Nanti" → verify credit recorded → pay partial debt → verify balance update
- **Multi-Price**: Add 40+ items → verify wholesale price auto-applied
- **Open Price**: Add custom item → verify it appears in cart and receipt
- **Weight Product**: Add product per kg → input 2.5 kg → verify total calculation
- **QRIS Payment**: Display QR → scan → confirm → verify order completes
- **Quick Cash**: Tap Rp 100.000 button → verify change calculated correctly
- **Rounding**: Set rounding to Rp 100 → process sale → verify rounded total on receipt
- **Service Charge**: Switch to Restaurant mode → verify 5% charge added and separate from tax
- **Voucher**: Create voucher → apply at checkout → verify discount applied
- **Down Payment**: Pay 50% DP → verify partial payment recorded → pay remaining
- **Cash Drawer**: Process cash sale → verify drawer open command sent
- **Split Payment**: Pay with QRIS + Cash → verify both recorded with ref numbers
- **Thermal Print**: Connect USB/Bluetooth printer → test print receipt (both modes)
- **WA Receipt**: Complete sale → send receipt via WhatsApp → verify formatted text
- **Barcode Label**: Generate barcode → print label → scan label → verify product found
- **Expiry Tracking**: Add product with expiry → verify alerts at 30/7 days before
- **Activity Log**: Perform various actions → verify all logged with correct details
- **Scheduled Reports**: Configure daily report → verify WhatsApp delivery at set time
- **Delivery Tracking**: Create delivery order → open driver view → start tracking → verify live map updates
- **Ongkos Kirim**: Set flat rate → create delivery order → verify fee in cart and receipt
- **COD Tracking**: Create COD order → driver marks delivered + inputs cash → admin reconciles
- **WA Notification**: Create delivery → verify WhatsApp messages sent at each status change
- **Proof of Delivery**: Driver delivers → takes photo → verify photo visible in admin
- **Driver Status**: Driver toggles online → appears in assignment list → toggles offline → disappears
- **Multi-Order Trip**: Assign 3 orders to 1 driver → verify all stops on map → deliver one by one
- **Failed Delivery**: Mark delivery as failed → retry → verify attempt counter
- **Delivery Zone**: Draw zone on map → set fee → order outside zone → verify rejection
- **Offline Mode**: Disconnect WiFi → process sale → reconnect → verify data syncs to Firestore
- **Responsive Design**: Test on tablet (POS), phone (driver), desktop (admin)
- **PWA Install**: Install on Android via Chrome → verify works offline
- **Theme**: Toggle light/dark mode → verify all pages render correctly

---

## POS Best Practices Included

1. ✅ **Offline-First**: Never lose a sale due to internet issues
2. ✅ **Fast Checkout**: Minimal taps to complete a sale (3-tap checkout)
3. ✅ **Barcode Support**: Instant product lookup via scanner
4. ✅ **Hold & Recall**: Park orders and come back to them
5. ✅ **Split Payment**: Combine cash + card/e-wallet
6. ✅ **Auto Tax Calculation**: Configurable inclusive/exclusive tax
7. ✅ **Sequential Order Numbers**: INV-YYYYMMDD-NNNN format
8. ✅ **Shift Management**: Open/close shift with cash reconciliation
9. ✅ **Return & Refund**: Full and partial returns with inventory auto-update
10. ✅ **Void Transaction**: Cancel sales with admin approval and time limit
11. ✅ **Bon/Hutang (Credit Sales)**: Buy now pay later for trusted customers
12. ✅ **Multi-Price / Wholesale**: Auto wholesale pricing at quantity thresholds
13. ✅ **Open Price / Custom Item**: Sell uncatalogued items with manual entry
14. ✅ **Weight-based Products**: Sell per kg/gram/liter with decimal quantities
15. ✅ **WhatsApp Receipt**: Send digital receipt via WhatsApp
16. ✅ **Barcode Label Printing**: Generate and print barcode stickers
17. ✅ **Expiry Date Tracking**: FIFO enforcement with expiry alerts
18. ✅ **Activity Log / Audit Trail**: Immutable log of all changes
19. ✅ **Scheduled Reports**: Auto-send daily/weekly reports via WhatsApp
20. ✅ **End-of-Day Reports**: Automatic daily summary with cash reconciliation
21. ✅ **PIN Quick Login**: Fast staff switching at the terminal
22. ✅ **Customer Loyalty**: Points system with tier-based benefits
23. ✅ **Kitchen Display**: Real-time order queue for kitchen staff
24. ✅ **Multi-Role Access (5 roles)**: Super Admin → Admin → Kasir → Dapur → Driver
25. ✅ **Module Visibility**: Super Admin can show/hide modules per outlet
26. ✅ **Real-time Sync**: Changes sync across all connected devices instantly
27. ✅ **Multi-Language**: Bahasa Indonesia (default) + English
28. ✅ **Vendor & Purchase Orders**: Full procurement cycle
29. ✅ **Smart Reorder**: Auto-suggest PO when stock is low
30. ✅ **Stock Opname**: Physical count with system reconciliation
31. ✅ **Delivery Fee / Ongkir**: 3 models (gratis/flat/per-km/zona)
32. ✅ **Customer Address & Map Pin**: Saved addresses with GPS coordinates
33. ✅ **COD Tracking**: Cash collection & driver reconciliation
34. ✅ **WhatsApp Delivery Notifications**: Auto-notify customer at each status
35. ✅ **Proof of Delivery**: Photo confirmation on delivery
36. ✅ **Driver Online/Offline**: Availability status toggle
37. ✅ **Multi-Order per Trip**: Batch delivery with route optimization
38. ✅ **Failed Delivery Handling**: Retry, return, refund workflow
39. ✅ **Scheduled Delivery**: Time slot selection for future delivery
40. ✅ **Auto-Assign Driver**: Nearest online driver auto-assignment
41. ✅ **Delivery Zones**: Configurable radius & polygon zones
42. ✅ **QRIS Payment**: QR Indonesia Standard (static & dynamic)
43. ✅ **Quick Cash Buttons**: Preset denominations for fast cash input
44. ✅ **Rounding / Pembulatan**: Auto-round totals to nearest Rp 100/500/1.000
45. ✅ **Payment Reference Numbers**: Audit trail for non-cash payments
46. ✅ **Service Charge**: Restaurant 5-10% (separate from tax)
47. ✅ **Voucher & Store Credit**: Coupon codes + store credit from returns
48. ✅ **Down Payment / DP**: Partial payment with remaining balance
49. ✅ **Cash Drawer Integration**: Auto-open on cash payment
50. ✅ **Tips / Gratuity**: Optional tip for restaurant orders
51. ✅ **Bank Transfer**: Manual confirmation with ref number
52. ✅ **Change Denomination**: Breakdown kembalian per pecahan uang
53. ✅ **10+ Payment Methods**: Cash, QRIS, Card, E-Wallet, Transfer, COD, Voucher, Store Credit, Points, DP
54. ✅ **Dashboard KPI Real-time**: Revenue, orders, profit — auto-update every transaction
55. ✅ **Comparison Reports**: Period vs period, outlet vs outlet, YoY growth
56. ✅ **Payment Reports**: Method distribution, COD reconciliation, DP aging, tips, vouchers
57. ✅ **Shrinkage / Loss Reports**: Total loss = stock variance + void + expired + discounts
58. ✅ **Smart Alerts**: 7 proactive alerts (target reached, revenue drop, stock out, etc.)
59. ✅ **ABC Analysis**: Inventory classified A/B/C by revenue contribution
60. ✅ **Tax Reports**: PPN collected, service charge, revenue kena pajak — SPT-ready
61. ✅ **Owner Mobile Dashboard**: Quick stats, push notifications, sparkline charts
62. ✅ **Forecasting**: Predict revenue, stock depletion, peak hours/days
63. ✅ **Export Enhanced**: PDF branded, Google Sheets, email, A4 print, share image
64. ✅ **Gudang / Warehouse Role**: Dedicated warehouse staff with inventory-only access
65. ✅ **Approval Chain**: PIN verification for void, discount >10%, refund, manual cash drawer
66. ✅ **Session Security**: Auto-logout, concurrent limit, login lock after 5 failed attempts
67. ✅ **Kasir Sub-Permissions**: Senior vs Junior cashier permission levels
68. ✅ **Staff Profile & Schedule**: Photo, work schedule, active/inactive status
69. ✅ **Shift Handover**: Transfer shift between cashiers with joint cash count
70. ✅ **Staff Performance**: Revenue ranking, void rate, checkout speed per cashier
71. ✅ **Custom Role Builder**: Create custom roles with granular permissions
72. ✅ **Time-Based Access**: Restrict login to work hours, weekend, holidays
73. ✅ **Role-Based Notifications**: Each role gets relevant alerts only
74. ✅ **Indigo Blue Theme**: Professional color system with full light/dark mode
