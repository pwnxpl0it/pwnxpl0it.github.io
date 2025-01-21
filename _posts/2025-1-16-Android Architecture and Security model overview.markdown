---
layout: post
title: "Android Architecture and Security Model Overview"
date: 2025-01-16
categories: [Android, Security, Mobile application penetration testing]
tags: [Android, Linux, Security, Architecture, SELinux]
description: "An overview of Android's architecture, including its layers, security model, and virtual machines."
---

# **Android Architecture**

Android OS is built on the Linux kernel and organized into multiple layers, each serving a specific purpose. These layers are:

- **Applications**
- **Application Framework**
- **Libraries and Android Runtime**
- **Hardware Abstraction Layer (HAL)**
- **Linux Kernel**

Each layer depends on the security of the layers beneath it. The **Linux kernel** is the foundation, providing core system functionalities and security features.

![Android Architecture](https://techvidvan.com/tutorials/wp-content/uploads/sites/2/2021/06/Android-Architecture-detail.jpg)

---

## **Android Architecture Layers**

### **1. Applications Layer**
This layer contains all the user-installed applications, as well as system applications, that provide end-user functionality.

---

### **2. Application Framework**
The framework exposes APIs to developers, enabling applications to interact with system components and device hardware. It simplifies:
- Rendering UI components.
- Managing device resources.
- Interfacing with hardware components like sensors and cameras.

---

### **3. Libraries and Android Runtime**

#### **Libraries**
Written in C/C++, these libraries handle essential low-level services like:
- Graphics rendering.
- Multimedia playback.
- Networking.

#### **Android Runtime**
This layer is critical for running applications and is responsible for executing their code.
The Android Runtime includes:
1. **Dalvik Virtual Machine (DVM)**: Legacy VM used in older Android versions.
2. **Android Runtime (ART)**: Replaced Dalvik starting with Android 5.0 (Lollipop).

#### Key Library Types:
- **Dalvik VM Libraries**: Enable interaction with the Dalvik VM.
- **Java Interoperability Libraries**: Adapted core Java libraries for Dalvik/ART.
- **Android Libraries**: Provide core functionality used in application development.

---

### **4. Hardware Abstraction Layer (HAL)**
The HAL acts as an interface between the hardware and higher layers. It standardizes hardware interactions, making it easier for Android to support various device configurations.

---

### **5. Linux Kernel**
Android is based on the Linux kernel version 2.6 (and later versions).
The kernel provides:
- Process and memory management.
- Security features like SELinux.
- Hardware drivers for communication with peripherals.

---

## **Android Virtual Machines**

Android apps are written in Java and compiled into platform-independent Dalvik Executable (DEX) files, which the Android Virtual Machines (VMs) execute.

### **Why Virtual Machines?**
- Abstraction: VMs abstract hardware differences, ensuring apps work across devices and OS versions.
- Portability: Developers focus on app logic without worrying about device-specific optimizations.

### **Dalvik vs ART**
- **Dalvik VM**:
  - Used in Android versions before KitKat (4.4).
  - Executes DEX files at runtime using a Just-In-Time (JIT) compilation approach.

- **Android Runtime (ART)**:
  - Introduced in KitKat (4.4) and fully adopted in Lollipop (5.0).
  - Uses Ahead-of-Time (AOT) compilation, converting DEX files to native machine code during installation for better performance.

**Key Differences**:

| Feature            | Dalvik VM           | Android Runtime (ART)      |
|---------------------|---------------------|-----------------------------|
| Compilation         | Just-In-Time (JIT) | Ahead-of-Time (AOT)         |
| Performance         | Moderate           | Faster app execution        |
| Resource Efficiency | Higher runtime overhead | Lower runtime overhead |

---

## **DEX, ODEX, and OAT Files**

- **DEX**:
  Standard file format used for Android apps downloaded from the Play Store.

- **ODEX** (Optimized DEX):
  Optimized version of DEX, precompiled for faster boot-time performance. Typically used by OEMs.

- **OAT**:
  Optimized bytecode format used by ART, offering significant performance improvements over ODEX.

---

## **Java vs Native Code**

- Most Android apps are written in Java, executed by the VM as DEX files.
- Native code (C/C++) is used for performance-critical applications like games.
- Using native code can bypass VM abstraction layers but introduces risks, such as memory corruption vulnerabilities.

---

## **Android Security Model**

The Android security model consists of two main layers:

1. **Operating System Security**:
   Ensures applications are isolated from one another using techniques like UID separation and sandboxing.

2. **Application-Level Security**:
   Developers can:
   - Expose specific functionality to other apps through permissions.
   - Limit app capabilities based on their risk tolerance.

### **Application Isolation**
Android isolates application data and execution, allowing apps with different trust levels to coexist securely on the same device.

---

### **UID Separation**
- Each application is assigned a unique **User ID (UID)** upon installation.
- The UID determines which files and processes the app can access.
- Apps cannot access files owned by other UIDs unless explicitly shared.

**Example**:
When listing application directories:

```bash
ls -la
```

Each directory is owned by a unique alphanumeric username tied to a numeric UID (use `-n` flag to view uid and guid).

> To view the UID  and GUID of an app, use the `ls` command:
> ```bash
> ls -n /data/data/com.example.app
> ```
> The UID and GUID should be the same for the app's directory.
{:.prompt-tip}

### **Sandboxing**
- Each app runs as a separate process under its own UID, ensuring strict isolation.
- Android implements sandboxing at the OS level using Linux permissions.

---

### **Enhancements with SELinux**
- Before Android 4.3:
  UID separation was the primary isolation mechanism. Root access compromises could lead to system-wide attacks.

- Starting with Android 4.3:
  SELinux (Security-Enhanced Linux) was introduced to enforce stricter security policies.
  - SELinux enforces "deny by default," allowing only explicitly permitted interactions.

