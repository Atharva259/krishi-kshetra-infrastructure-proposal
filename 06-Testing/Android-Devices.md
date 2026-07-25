# 📱 LOW-END MOBILE DEVICE OPTIMIZATION & VERIFICATION SPECIFICATION

**Document ID**: `KK-QA-AND-2026-V1`  
**Classification**: PUBLIC / ENGINEERING REFERENCE  
**Target Platform**: Android (API Level 21+ / Android 5.0 Lollipop through Android Go Editions)  
**Status**: ACTIVE  

---

## ℹ️ 1. Context & Architectural Constraints
The primary users of the Krishi Kshetra mobile application are farmers, field agents, and warehouse operators in regions where mobile hardware is typified by entry-level specifications. To ensure high usability, accessibility, and reliability, the application must be designed from the ground up to operate within extreme hardware constraints.

### ⚠️ Core Hardware Constraints
* **Random Access Memory (RAM)**: Target devices possess **1.0 GB to 3.0 GB of total system RAM** (typically running Android Go Edition). The app must maintain a memory footprint of **under 150 MB** during active execution to prevent Out-Of-Memory (OOM) background termination.
* **Storage Performance**: Low-end devices utilize slow eMMC storage rather than modern UFS. High disk write/read operations will cause UI freezes (ANRs - Application Not Responding).
* **Processing Unit (CPU/GPU)**: Dual-core or quad-core MediaTek/Snapdragon entry-level SOCs with low single-core performance. CPU-intensive operations (image processing, heavy JSON serialization) must be offloaded to background threads.
* **Network Availability**: Intermittent, high-latency 2G/3G networks or shared local Wi-Fi. Offline-first functionality is mandatory.

---

## 🛠️ 2. Architectural Guidelines & Best Practices

To target these constraints effectively, development teams must strictly adhere to the following architectural design principles:

### 2.1 Offline-First Data Architecture
The application must remain fully functional without internet access. Data is cached locally and synchronized opportunistically.

```
[ Mobile UI ] <──> [ View Model ] <──> [ Repository Layer ]
                                               │
                       ┌───────────────────────┴───────────────────────┐
                       ▼ (Offline Cache)                               ▼ (Opportunistic Sync)
                [ SQLite / Room DB ]                           [ Remote REST API / gRPC ]
```

* **Local Database**: Use SQLite via Room DB. Do not use heavy ORMs that introduce run-time overhead.
* **Data Ingestion**: Network operations must update the local DB first; the UI must observe the local DB using streams (e.g., Kotlin Flow or LiveData). This guarantees immediate UI response.
* **Payload Serialization**: Keep payloads small. Use highly optimized JSON parsers (e.g., Moshi with code generation) or Protocol Buffers instead of reflection-based parsers (e.g., GSON).

### 2.2 Memory & Allocation Optimization
Garbage Collection (GC) pauses cause visible frame drops (jank) on weak CPUs.
* **Avoid Reflection**: Avoid runtime dependency injection frameworks that rely on reflection. Use compile-time dependency injection (e.g., Dagger/Hilt or Koin compile-time check).
* **Object Pooling**: Recycle memory-heavy objects such as bitmap decoders, date formatters, and network buffers.
* **Leak Prevention**: Ensure Lifecycle Owners are correctly detached. Nullify adapter references in Fragment `onDestroyView()`.

### 2.3 UI & Rendering Optimization
Maintain a target frame rate of **60 FPS** even on entry-level GPUs.
* **Flat Layout Hierarchies**: Avoid deep nested layouts. Use `ConstraintLayout` to keep layouts flat and reduce layout passes.
* **Overdraw Elimination**: Do not render overlapping background colors that are hidden from the user's view. Enable "Show GPU Overdraw" during debugging.
* **Asset Optimization**:
  * Use **Vector Drawables (SVG)** instead of rasterized PNGs to reduce APK size.
  * For rasterized images (e.g., crop diagnostic photos), enforce WebP format compression.
  * Maximum image upload dimensions must be capped at 1080p, resized on the edge before transmission.

### 2.4 Battery & Network Serialization
* **Job Scheduling**: Schedule intensive sync operations using Android `WorkManager`. Restrict execution to periods when the device is **charging** and connected to **Wi-Fi** (unless forced by user).
* **Batching Network Requests**: Group API updates into singular batched payloads instead of initiating multiple continuous HTTP handshakes.

---

## 📊 3. Target Device Profile Matrix
The QA team will perform manual and automated testing across the following target profiles:

| Profile Level | Target RAM / Storage | Representative Devices | OS Version Range | Target APK Size |
| :--- | :--- | :--- | :--- | :--- |
| **Tier 1 (Ultra Low-End)** | 1.0 GB / 8 GB or 16 GB | Samsung Galaxy A01 Core, Redmi Go, Tecno Pop | Android 8.1 - 10 (Go Edition) | `< 12 MB` |
| **Tier 2 (Low-End)** | 2.0 GB - 3.0 GB / 32 GB | Xiaomi Redmi 9A, Samsung A10s, Motorola E7 | Android 9.0 - 11 | `< 18 MB` |
| **Tier 3 (Medium-End)** | 4.0 GB+ / 64 GB+ | Samsung Galaxy A32, Realme C35, Oneplus Nord CE | Android 11+ | Unlimited |

---

## 🧪 4. Performance Testing & Emulation Guidelines

Developers must test their builds under simulated low-end environments:

### 4.1 System Emulation Configuration
When using the Android Studio Emulator, configure the AVD (Android Virtual Device) with the following hardware profiles:
* **RAM**: Set to `1024 MB`.
* **VM Heap**: Set to `128 MB`.
* **CPU Throttle**: Enable CPU throttling to simulate 25% of standard host CPU speed.
* **Network Emulator**: Set network speed to `Edge` or `Gprs` and latency to `400ms (Average 3G)`.

### 4.2 On-Device Debugging Settings
Verify rendering efficiency using on-device developer options:
1. **Profile HWUI Rendering**: Enable this to view visual frame rendering time bars on screen. Any bar exceeding the green line (16ms) indicates frame drops.
2. **Don't Keep Activities**: Enable this to simulate aggressive OS RAM reclamation and verify that the application correctly saves and restores state without crashing.
3. **Limit Background Processes**: Set limit to `No background processes` to test service restoration behavior.

---

## 📈 5. QA Metrics & Acceptance Criteria
For an application build to pass the low-end device QA check, it must satisfy these criteria:

* **Cold Startup Time**: Must not exceed **3.0 seconds** on a Tier 1 device.
* **Hot Startup Time**: Must not exceed **1.0 second**.
* **Memory Peak Consumption**: Must remain under **120 MB** during standard workflows.
* **Battery Consumption**: Must not consume more than **3% of battery capacity per hour** of active usage (GPS excluded).
* **Crash Rate**: Crash-free sessions must exceed **99.9%** across Tier 1 and Tier 2 devices.
* **ANR Rate**: ANR sessions must be **less than 0.1%**.
