# 🧾 SkokPOS

**Multi-Purpose Point of Sales System** — A full-featured, offline-first POS application with delivery live tracking, thermal printing, and sales best practices.

![Status](https://img.shields.io/badge/Status-In%20Development-yellow)
![Platform](https://img.shields.io/badge/Platform-PWA-blue)
![Framework](https://img.shields.io/badge/Framework-Next.js%2015-black)
![Database](https://img.shields.io/badge/Database-Firebase-orange)

---

## ✨ Features

### 🛒 Core POS
- Fast checkout (3-tap to complete sale)
- Product catalog with categories, variants & modifiers
- Barcode scanner support
- Multi-payment: Cash, Card, E-Wallet, Split Payment
- Discounts (percentage & fixed amount)
- Tax calculation (PPN 12%, configurable)
- Hold & recall orders
- Sequential order numbering

### 🖨️ Thermal Printing
- ESC/POS protocol via WebUSB & Web Bluetooth
- Customizable receipt template (Bahasa Indonesia)
- Kitchen Order Tickets (KOT)
- Print preview before printing
- Auto-print on order completion

### 🚚 Delivery & Live Tracking
- Delivery management Kanban board
- Real-time GPS driver tracking on map (Leaflet + OpenStreetMap)
- Customer-facing tracking page (no login required)
- Driver mobile interface with one-tap status updates
- ETA calculation
- Demo simulation mode for testing

### 📦 Inventory Management
- Stock level tracking with low-stock alerts
- Stock adjustment with audit trail
- Multi-store stock with inter-store transfer
- CSV bulk import/export

### 📊 Reports & Analytics
- Revenue charts, category sales, payment distribution
- Hourly sales heatmap
- End-of-day reports with cash reconciliation
- Export to CSV/PDF
- Multi-store aggregate reporting

### 👥 Staff Management
- Role-based access control (Admin, Kasir, Dapur, Driver)
- PIN-based quick login for shift changes
- Per-staff sales performance tracking
- Clock in/out with work hours log

### 💎 Customer & Loyalty
- Customer database with purchase history
- Loyalty points system
- Customer tiers (Regular, Silver, Gold, Platinum)

### 🍳 Kitchen Display System (KDS)
- Real-time order queue
- Color-coded priority (wait time)
- Audio alerts for new orders
- One-tap item completion

---

## 🏗️ Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 15 (App Router) + React 19 |
| Styling | Vanilla CSS (Design System with Custom Properties) |
| State | Zustand + TanStack React Query |
| Database | Firebase Firestore (offline persistence) |
| Realtime | Firebase Realtime Database (GPS tracking) |
| Auth | Firebase Authentication |
| Maps | Leaflet + OpenStreetMap |
| Charts | Recharts |
| Printing | Custom ESC/POS engine (WebUSB / Web Bluetooth) |
| Icons | Lucide React |

---

## 🌐 Key Architecture

- **Offline-First**: Full functionality without internet, auto-sync on reconnect
- **PWA**: Installable on Android/iOS/Desktop, works like a native app
- **Multi-Store**: Independent catalogs, inventory, and reporting per outlet
- **Responsive**: Optimized for tablet (POS), phone (driver), desktop (admin)
- **i18n**: Bahasa Indonesia (default) with English option

---

## 📁 Project Structure

```
skokpos/
├── docs/                    # Project documentation
│   ├── implementation_plan.md  # Technical implementation plan
│   └── SkokPOS_SOW_v1.0.md    # Statement of Work
├── public/                  # PWA assets (manifest, icons, sounds)
├── src/
│   ├── app/                 # Next.js App Router pages
│   │   ├── (pos)/           # POS screens (checkout, delivery, kitchen)
│   │   ├── (admin)/         # Admin screens (inventory, reports, staff)
│   │   ├── (driver)/        # Driver mobile view
│   │   └── track/           # Public delivery tracking
│   ├── components/          # Reusable UI components
│   ├── lib/                 # Utilities (Firebase, printer, tracking)
│   ├── stores/              # Zustand state stores
│   └── styles/              # CSS design system
└── package.json
```

---

## 📄 Documentation

- [Implementation Plan](docs/implementation_plan.md) — Technical blueprint & architecture
- [Statement of Work](docs/SkokPOS_SOW_v1.0.md) — Project scope, deliverables & timeline

---

## 📝 License

This project is proprietary. All rights reserved.

---

*Built with ❤️ for Indonesian businesses*
