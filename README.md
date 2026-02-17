# SmartCook

Aplikasi mobile berbasis Flutter dengan backend Node.js untuk menemukan dan mengelola resep masakan dengan fitur AI chatbot, manajemen kulkas, dan rekomendasi resep yang disesuaikan dengan preferensi pengguna.

## 📱 Tentang SmartCook

SmartCook adalah aplikasi resep masakan full-stack yang membantu pengguna menemukan resep berdasarkan:
- **Bahan yang tersedia di kulkas** - Rekomendasi resep berdasarkan bahan yang dimiliki
- **Preferensi kesehatan** - Filter resep berdasarkan alergi dan riwayat medis
- **Waktu makan** - Resep untuk breakfast, lunch, dan dinner
- **AI Chat Bot** - Interaksi dengan Google Gemini AI untuk rekomendasi resep yang lebih interaktif
- **Offline Support** - Aplikasi dapat digunakan offline dengan sinkronisasi otomatis saat online kembali

## 🏗 Arsitektur Sistem

```
┌─────────────────────────────────┐
│   Flutter App (Frontend)        │
│   - Mobile UI/UX                │
│   - Offline Caching             │
│   - State Management            │
└──────────────┬──────────────────┘
               │
               │ HTTP + API Key + JWT Token
               ↓
┌─────────────────────────────────┐
│   Express API (Backend)         │
│   - RESTful API                 │
│   - Authentication              │
│   - Business Logic              │
└──────────────┬──────────────────┘
               │
               ├─── MongoDB (Database)
               │    ├── Users
               │    ├── Recipes
               │    ├── FridgeItems
               │    ├── Favorites
               │    └── ChatHistory
               │
               ├─── Firebase Admin SDK (Google Auth)
               │
               └─── Google Gemini AI (Chat Bot)
```

## 🛠 Tech Stack

### Frontend
- **Flutter** - Mobile framework
- **Dart** - Programming language
- **Firebase Auth** - Authentication
- **Shared Preferences** - Local storage & caching

### Backend
- **Node.js** - Runtime environment
- **Express 5** - Web framework
- **MongoDB** - Database
- **Mongoose** - ODM
- **Google Gemini AI** - AI Chat Bot
- **Firebase Admin SDK** - Google Sign-In verification
- **JWT** - Authentication tokens

## 📁 Struktur Proyek

```
SmartCook/
├── smartcook-frontend/          # Flutter mobile application
│   ├── lib/                     # Source code
│   ├── pubspec.yaml             # Dependencies
│   ├── README.md                # Frontend documentation
│   └── LICENSE                  # Frontend license
│
├── smartcook-backend/           # Node.js REST API
│   ├── src/                     # Source code
│   ├── server.js               # Entry point
│   ├── package.json            # Dependencies
│   ├── README.md               # Backend documentation
│   └── LICENSE                 # Backend license
│
├── README.md                   # Dokumentasi utama (file ini)
├── .gitignore                  # Git ignore rules
└── LICENSE                     # Root license
```

## 📚 Dokumentasi

Untuk dokumentasi lengkap tentang masing-masing bagian proyek, silakan baca:

- **[📱 Frontend Documentation](smartcook-frontend/README.md)** - Dokumentasi lengkap tentang aplikasi Flutter, fitur, alur aplikasi, instalasi, dan setup
- **[⚙️ Backend Documentation](smartcook-backend/README.md)** - Dokumentasi lengkap tentang REST API, endpoints, database schema, security, dan deployment

## 🚀 Quick Start

### Prerequisites

**Untuk Frontend:**
- Flutter SDK (^3.5.0 atau lebih tinggi)
- Dart SDK
- Android Studio / VS Code dengan Flutter extension
- Firebase project setup

**Untuk Backend:**
- Node.js (v18 atau lebih tinggi)
- npm atau yarn
- MongoDB (Atlas atau lokal)
- Firebase project (untuk Google Sign-In)
- Google Gemini API key
- SMTP credentials (untuk email OTP)

### Instalasi Backend

1. Masuk ke folder backend:
   ```bash
   cd smartcook-backend
   ```

2. Install dependencies:
   ```bash
   npm install
   ```

3. Setup environment variables:
   ```bash
   cp .env.example .env
   # Edit .env dan isi semua value yang diperlukan
   ```

4. Jalankan server:
   ```bash
   # Development
   npm run dev
   
   # Production
   npm start
   ```

Lihat [Backend Documentation](smartcook-backend/README.md) untuk detail lengkap.

### Instalasi Frontend

1. Masuk ke folder frontend:
   ```bash
   cd smartcook-frontend
   ```

2. Install dependencies:
   ```bash
   flutter pub get
   ```

3. Setup Firebase:
   - Generate `firebase_options.dart` menggunakan FlutterFire CLI:
     ```bash
     flutterfire configure
     ```

4. Konfigurasi API:
   ```bash
   cp lib/config/api_config.example.dart lib/config/api_config.dart
   # Edit api_config.dart dan isi dengan URL backend dan API key
   ```

5. Jalankan aplikasi:
   ```bash
   flutter run
   ```

Lihat [Frontend Documentation](smartcook-frontend/README.md) untuk detail lengkap.

## ✨ Fitur Utama

### 🏠 Homepage
- Dashboard dengan rekomendasi resep berdasarkan waktu makan
- Kategori masakan (Sehat, Nutrisi Seimbang, Masakan Barat)
- Preview kulkas dengan bahan yang tersedia
- Resep populer dan rekomendasi personal

### 🔍 Search
- Pencarian resep dengan dukungan offline caching
- Riwayat pencarian terbaru

### 🤖 AI Chat Bot
- Interaksi dengan Google Gemini AI
- Streaming response untuk pengalaman real-time
- Recipe embeds langsung dalam chat
- Context-aware berdasarkan alergi dan bahan di kulkas

### 🧊 Kulkas Management
- Tambah/hapus bahan makanan
- Tracking tanggal kadaluarsa
- Rekomendasi resep berdasarkan bahan yang tersedia

### ⭐ Favorites
- Simpan resep favorit
- Sinkronisasi otomatis saat online kembali
- Dukungan offline

### 👤 Profile
- Manajemen profil pengguna
- Pengaturan alergi makanan
- Ubah email dan password

## 🔐 Authentication

Aplikasi mendukung dua metode autentikasi:
- **Email/Password** - Registrasi dan login tradisional
- **Google Sign-In** - Login menggunakan akun Google

## 🔄 Alur Aplikasi

### Frontend Flow
```
Splash Screen
    ↓
[Check Auth Status]
    ↓
    ├─→ Not Authenticated → Sign In / Sign Up
    │                           ↓
    │                    [Google Sign-In / Email-Password]
    │                           ↓
    │                    [Check Onboarding Status]
    │                           ↓
    │                    ├─→ Not Completed → Onboarding Flow
    │                    └─→ Completed → Homepage
    │
    └─→ Authenticated → [Check Onboarding]
                            ↓
                    ├─→ Not Completed → Onboarding Flow
                    └─→ Completed → Homepage
                                        ↓
                            [Bottom Navigation]
                                ├─→ Home
                                ├─→ Search
                                ├─→ Bot Chat
                                ├─→ Save/Favorites
                                └─→ Profile
```

### Backend Request Flow
```
Client Request (Flutter App)
    ↓
[API Key Validation]
    ↓
[Rate Limiting]
    ↓
[Route Handler]
    ↓
[Authentication Middleware] (jika diperlukan)
    ↓
[Controller]
    ↓
[Database Query / AI Processing]
    ↓
[Response]
```

## 📋 Requirements

### Frontend
- Flutter SDK ^3.5.0
- Dart SDK
- Android SDK (untuk Android development)
- Xcode (untuk iOS development, macOS only)
- Firebase project dengan Authentication enabled

### Backend
- Node.js v18 atau lebih tinggi
- MongoDB (Atlas atau lokal)
- Firebase project dengan Admin SDK
- Google Gemini API key
- SMTP credentials untuk email

## 🧪 Development

### Backend Development
```bash
cd smartcook-backend
npm run dev    # Auto-reload dengan --watch
```

### Frontend Development
```bash
cd smartcook-frontend
flutter run    # Run di device/emulator
```

## 🚢 Deployment

### Backend Deployment
- Setup environment variables di production server
- Gunakan MongoDB Atlas untuk database production
- Setup monitoring dan logging
- Gunakan HTTPS (SSL/TLS)
- Konfigurasi rate limiting sesuai traffic

Lihat [Backend Documentation](smartcook-backend/README.md) untuk detail deployment.

### Frontend Deployment
- Build APK untuk Android:
  ```bash
  flutter build apk --release
  ```
- Build App Bundle untuk Google Play:
  ```bash
  flutter build appbundle --release
  ```
- Build untuk iOS:
  ```bash
  flutter build ios --release
  ```

## 👥 Core Team

Tim **SmartCook** yang membangun proyek ini:

<table>
<tr>
<td align="center">
<img src="https://github.com/faturrahman82.png" width="100px" alt="Maul"/>
<br />
<strong>Maul</strong>
<br />
<sub>💻 <strong>Frontend Flutter Developer</strong></sub>
<br />
<sub>
📱 Flutter Implementation<br/>
🎯 Feature Development<br/>
🔧 Component Building<br/>
📊 State Management<br/>
🧪 Testing & Debugging<br/>
</sub>
<br />
<a href="https://github.com/faturrahman82">GitHub</a>
</td>
<td align="center">
<img src="https://github.com/geraldy-pf.png" width="100px" alt="Geraldy Putra Fazrian"/>
<br />
<strong>Geraldy Putra Fazrian</strong>
<br />
<sub>💻 <strong>Frontend Flutter Developer</strong></sub>
<br />
<sub>
📱 Flutter Implementation<br/>
🎯 Feature Development<br/>
🔧 Component Building<br/>
📊 State Management<br/>
🧪 Testing & Debugging<br/>
</sub>
<br />
<a href="https://github.com/geraldy-pf">GitHub</a>
</td>
<td align="center">
<img src="https://github.com/ChillGuyAdit.png" width="100px" alt="ChillGuyAdit"/>
<br />
<strong>ChillGuyAdit</strong>
<br />
<sub>🎨 <strong>UI/UX Designer</strong></sub>
<br />
<sub>
🎨 Visual Design<br/>
🖼️ Asset Creation<br/>
🎯 Design System<br/>
✨ User Experience<br/>
📐 Layout Design<br/>
</sub>
<br />
<a href="https://github.com/ChillGuyAdit">GitHub</a>
</td>
<td align="center">
<img src="https://github.com/Sadamdi.png" width="100px" alt="Sulthan Adam Rahmadi"/>
<br />
<strong>Sulthan Adam Rahmadi</strong>
<br />
<sub>🚀 <strong>Backend Developer</strong></sub>
<br />
<sub>
⚙️ Backend Server<br/>
🔧 Logic Implementation<br/>
🗄️ Database Design<br/>
🔐 API Development<br/>
🏗️ System Architecture<br/>
</sub>
<br />
<a href="https://github.com/Sadamdi">GitHub</a>
</td>
</tr>
</table>

## 📄 License

Proyek ini menggunakan **MIT License**.

Lihat file [LICENSE](LICENSE) untuk detail lengkap.

## 🙏 Acknowledgments

Terima kasih kepada semua kontributor yang telah membantu dalam pengembangan aplikasi SmartCook ini.

---

**SmartCook** - Temukan resep masakan terbaik untukmu! 🍳

Untuk dokumentasi lengkap, silakan baca:
- [📱 Frontend Documentation](smartcook-frontend/README.md)
- [⚙️ Backend Documentation](smartcook-backend/README.md)
