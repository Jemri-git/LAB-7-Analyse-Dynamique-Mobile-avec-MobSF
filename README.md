# LAB-7-Analyse-Dynamique-Mobile-avec-MobSF

**Environnement :** Windows 11 — AVD Android API 29/30 x86_64 — MobSF v3.1.7 Beta (Docker)  
**Cible :** DIVA (Damn Insecure and Vulnerable Android App)  
**Outils :** Android Studio — ADB — Docker — MobSF   

---


## 1. Vue d'ensemble

### Qu'est-ce que l'analyse dynamique ?

L'analyse dynamique consiste à observer le comportement d'une application **pendant qu'elle tourne** — contrairement à l'analyse statique qui lit le code sans l'exécuter. Elle permet de capturer ce qui échappe à la lecture du code : données écrites sur le disque, trafic réseau chiffré, comportements conditionnels au runtime.

**MobSF (Mobile Security Framework)** est une plateforme open source qui automatise les deux types d'analyse. Son module dynamique connecte un émulateur Android, injecte Frida, configure un proxy HTTPS et expose une interface web pour piloter l'analyse en temps réel.

**DIVA (Damn Insecure and Vulnerable Android App)** est une application volontairement vulnérable conçue pour apprendre à détecter les failles mobiles classiques — stockage insecure, credentials hardcodés, contrôle d'accès absent, validation d'entrée insuffisante.

### Objectifs du lab

- Configurer un émulateur Android propre sans Play Store
- Déployer MobSF via Docker
- Réaliser une analyse statique + dynamique de DIVA
- Détecter des vulnérabilités en temps réel : stockage insecure, intents non protégés, credentials hardcodés
- Utiliser Frida pour l'instrumentation dynamique depuis MobSF

### Vulnérabilités DIVA couvertes

| # | Challenge | Catégorie OWASP Mobile |
|---|---|---|
| 1 | Insecure Logging | M2 - Insecure Data Storage |
| 2 | Hardcoded Issues | M9 - Reverse Engineering |
| 3 | Insecure Data Storage (SharedPreferences) | M2 |
| 4 | Insecure Data Storage (SQLite) | M2 |
| 5 | Insecure Data Storage (Fichiers) | M2 |
| 6 | Insecure Data Storage (External Storage) | M2 |
| 7 | Input Validation Issues | M7 - Poor Code Quality |
| 8 | Access Control Issues | M6 - Insecure Authorization |

---

## 2. Architecture de la solution

```
┌─────────────────────────────────────────────────────────┐
│                    Machine hôte (Windows/Mac/Linux)      │
│                                                          │
│  ┌─────────────────────┐    ┌──────────────────────┐    │
│  │   Android Emulator  │    │   MobSF (Docker)     │    │
│  │   AVD API 29/30     │◄──►│   Port 8000          │    │
│  │   (ADB: 5554)       │    │   Frida Server       │    │
│  │   rooté             │    │   Proxy HTTPS        │    │
│  └─────────────────────┘    └──────────────────────┘    │
│            ▲                          ▲                  │
│            │ adb                      │ http://127.0.0.1:8000
│            └──────────────────────────┘                  │
│                                       │                  │
│                              ┌────────┴──────┐           │
│                              │   Navigateur  │           │
│                              │   Analyste    │           │
│                              └───────────────┘           │
└─────────────────────────────────────────────────────────┘
```

**Flux d'analyse :**
1. L'APK est uploadé dans MobSF → analyse statique automatique
2. MobSF installe l'APK sur l'émulateur via ADB
3. Frida Server est injecté dans l'émulateur (root obligatoire)
4. Tout le trafic réseau passe par le proxy MobSF (certificat CA installé)
5. L'analyste explore l'app → MobSF capture logs, trafic, fichiers, intents

---

## 3. Étape 1 — Création de l'AVD sans Play Store

### Pourquoi sans Play Store ?

| Type d'image | Play Store | Recommandé pour |
|---|---|---|
| Google Play | ✅ Inclus | Usage quotidien |
| Google APIs | ❌ Absent | Tests avec services Google |
| **Android pur** | **❌ Absent** | **✅ Analyse sécurité** |

Un émulateur sans Play Store garantit des logs propres, un trafic réseau sans bruit de fond (Firebase, Play Services) et une compatibilité totale avec le rooting nécessaire à Frida.

> ⚠️ MobSF supporte jusqu'à Android **API 30** maximum. À partir de l'API 31+, `/system` n'est plus accessible en écriture, ce qui bloque l'installation de Frida Server.

### Procédure

```
1. Android Studio → Tools → AVD Manager → Create Virtual Device
2. Modèle : Pixel 5 ou Pixel 6
3. System Image → onglet "x86 Images" → choisir API 29 ou 30
   → SANS le label "Google Play"
4. Nommer l'AVD : MobSF_DIVA_API_30
```

### Lancer l'émulateur via le script MobSF

```powershell
# Windows PowerShell
.\scripts\start_avd.ps1
# → choisir MobSF_DIVA_API_30

# Vérification
adb devices
# → emulator-5554   device
```
<img width="246" height="487" alt="2" src="https://github.com/user-attachments/assets/6d2d0395-a79c-4fe2-9238-fcfcc788fc98" />

> *Capture de l'émulateur AVD lancé sur Genymotion/AVD — écran d'accueil Android montrant les apps installées dont DIVA visible dans la liste*

---

## 4. Étape 2 — Cloner et lancer MobSF via Docker

### Cloner le dépôt

```bash
git clone https://github.com/MobSF/Mobile-Security-Framework-MobSF.git
cd Mobile-Security-Framework-MobSF
```

### Lancer MobSF

```bash
# Télécharger l'image Docker
docker pull opensecurity/mobile-security-framework-mobsf:latest

# Lancer MobSF connecté à l'émulateur
docker run -it --rm \
  -p 8000:8000 \
  -e MOBSF_ANALYZER_IDENTIFIER=emulator-5554 \
  opensecurity/mobile-security-framework-mobsf:latest
```

### Accès à l'interface

```
http://127.0.0.1:8000
Login : mobsf / mobsf
```

<img width="847" height="451" alt="1" src="https://github.com/user-attachments/assets/fc8713cb-e223-4aea-a6b2-67d3415bb0fe" />

> *Capture de l'interface MobSF v3.1.7 Beta accessible sur `127.0.0.1:8000` — page d'accueil avec le bouton "Upload & Analyze", la barre de progression "Analyzing.." et le champ "Search MD5" visible*

---

## 5. Étape 3 — Upload et analyse statique de DIVA

### Télécharger DIVA

```bash
# Option GitHub
git clone https://github.com/payatu/diva-android
# → APK dans le dossier du projet

# Option directe
# http://www.payatu.com/damn-insecure-and-vulnerable-app/
```

### Uploader dans MobSF

1. Cliquer sur **"Upload & Analyze"** dans MobSF
2. Glisser-déposer `diva-beta.apk`
3. Attendre la fin du scan statique (30–60 secondes)

MobSF analyse automatiquement :
- Permissions Android déclarées
- Activités, Services, Receivers exportés
- Appels réseau et stockage
- Certificat de signature
- Secrets hardcodés potentiels

<img width="830" height="73" alt="3" src="https://github.com/user-attachments/assets/f46b75ce-855f-4a09-9c8f-d3379bf11460" />


> *Capture de MobSF après l'analyse statique de DIVA — ligne de résultat encadrée en rouge montrant : icône Android, nom "Diva - 1.0", fichier "diva-beta.apk", package "jakhar.aseem.diva", boutons "Start Dynamic Analysis" et "View Report"*

---



## 6. Étape 4 — Démarrer l'analyse dynamique

### Lancer le Dynamic Analyzer

Dans MobSF, cliquer sur **"Start Dynamic Analysis"** depuis la page de résultats de DIVA.

MobSF effectue automatiquement :
- Installation de l'APK sur l'émulateur via ADB
- Injection de Frida Server
- Configuration du proxy HTTPS
- Démarrage du streaming d'écran

### Vérification dans les logs MobSF

```
Running HTTPS intercepting proxy
Invoking MobSF agents
Environment is ready for user assisted dynamic analysis.
Navigate through all the flows of the app manually.
Instrumenting app with frida
Successfully attached
Screen streaming started
```

<img width="825" height="456" alt="4" src="https://github.com/user-attachments/assets/0ec5f25b-a6d0-4dbb-84bb-c3f5e3fb5395" />

> *Capture de l'interface MobSF Dynamic Analyzer — à gauche l'émulateur affichant l'écran "Welcome to DIVA!" avec la liste des 13 challenges, à droite le panneau Dynamic Analyzer avec les logs de connexion et la section Frida Scripts montrant les options Default (API Monitoring, SSL Pinning Bypass, Root Detection Bypass, Debugger Check Bypass) et Auxiliary*

---

## 7. Étape 5 — Exploration des vulnérabilités DIVA

### Challenge 3 — Insecure Data Storage Part 1 (SharedPreferences)

**Objectif :** trouver où et comment les credentials sont stockés.

Sur l'émulateur, ouvrir le challenge **"3. Insecure Data Storage - Part 1"**, saisir un nom d'utilisateur et un mot de passe puis cliquer **SAVE**.

<img width="831" height="612" alt="5" src="https://github.com/user-attachments/assets/2e069c42-1c8b-4d22-a1a2-f3983f3395a7" />

> *Capture de MobSF Dynamic Analyzer — à gauche l'émulateur affichant le challenge "3. Insecure Data Storage - Part 1" avec les champs "Dila Dina" et le mot de passe saisi, le message "3rd party credentials saved successfully!" visible en bas. À droite le panneau Dynamic Analyzer avec les logs Frida actifs et les scripts Default cochés*

**Résultat — Fichier SharedPreferences exposé :**

Le File Monitor de MobSF révèle que les credentials ont été écrits en clair dans un fichier XML :

<img width="605" height="155" alt="6" src="https://github.com/user-attachments/assets/aede5610-a0e3-4c66-a682-42f9fa49fe03" />

> *Capture du contenu XML du fichier SharedPreferences intercepté par MobSF — structure XML montrant `<string name="user">Dila Dina</string>` et `<string name="password">password123</string>` stockés en clair*

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string name="user">Dila Dina</string>
    <string name="password">password123</string>
</map>
```

**Analyse de la vulnérabilité :**

| Élément | Valeur |
|---|---|
| Fichier | `/data/data/jakhar.aseem.diva/shared_prefs/*.xml` |
| Données exposées | Username + Password en clair |
| Risque OWASP | M2 — Insecure Data Storage |
| Impact | Extraction triviale par backup ADB ou accès root |
| Correction | `EncryptedSharedPreferences` (Jetpack Security) |

### Challenge 4 — Insecure Data Storage Part 2 (SQLite)

**Objectif :** trouver les credentials stockés dans une base SQLite.

Sur l'émulateur, ouvrir **"4. Insecure Data Storage - Part 2"**, saisir des credentials et cliquer SAVE.

<img width="832" height="608" alt="7" src="https://github.com/user-attachments/assets/a4c96637-a6b7-4410-9138-48e0a19b9d33" />

> *Capture de MobSF Dynamic Analyzer — à gauche l'émulateur affichant le challenge "4. Insecure Data Storage - Part 2" avec les champs "usertesting" et mot de passe saisis. À droite le panneau Dynamic Analyzer avec les logs montrant "Successfully attached", "Logcat Streaming started", "Screen streaming started" et la section Frida Scripts avec les options Default cochées*

**Résultat — Base SQLite non chiffrée :**

```
File Monitor →
/data/data/jakhar.aseem.diva/databases/divaDB

SELECT * FROM myuser;
→ id | username    | password
→ 1  | usertesting | [mot de passe en clair]
```

**Analyse de la vulnérabilité :**

| Élément | Valeur |
|---|---|
| Fichier | `/data/data/jakhar.aseem.diva/databases/divaDB` |
| Données exposées | Table `myuser` avec username + password en clair |
| Risque OWASP | M2 — Insecure Data Storage |
| Impact | Extraction par backup ADB, root, ou exploitation d'une autre vulnérabilité |
| Correction | SQLCipher ou Room avec chiffrement |

---
## 8. Étape 6 — Instrumentation Frida dans MobSF

### Scripts Frida disponibles par défaut

MobSF intègre des scripts Frida prêts à l'emploi dans l'interface Dynamic Analyzer :

**Default (activés automatiquement) :**

| Script | Rôle |
|---|---|
| API Monitoring | Capture tous les appels API sensibles en temps réel |
| SSL Pinning Bypass | Neutralise le certificate pinning (TrustManager, OkHttp) |
| Root Detection Bypass | Bypass `Build.TAGS`, `File.exists()` pour `/su` |
| Debugger Check Bypass | Neutralise `Debug.isDebuggerConnected()` |

**Auxiliary (activables à la demande) :**

| Script | Rôle |
|---|---|
| Enumerate Loaded Classes | Liste toutes les classes Java chargées |
| Capture Strings | Capture les chaînes de caractères manipulées |
| Capture String Comparisons | Capture les comparaisons de chaînes (utile pour les secrets) |
| Enumerate Class Methods | Liste les méthodes d'une classe spécifiée |
| Search Class Pattern | Cherche des classes par pattern (ex: `ssl\.Trust*`) |
| Trace Class Methods | Trace les appels de méthodes spécifiées |

### Lancer l'instrumentation

```
1. Cocher les scripts souhaités dans le panneau Frida Scripts
2. Cliquer "Start Instrumentation"
3. Interagir avec l'app sur l'émulateur
4. Cliquer "Frida Live Logs" pour voir les captures en temps réel
```
---

## 9. Concepts clés

### 9.1 Analyse statique vs dynamique

| Approche | Ce qu'elle voit | Ce qu'elle manque |
|---|---|---|
| Statique (JADX, MobSF scan) | Code, permissions, strings hardcodées | Comportement conditionnel, chiffrement runtime |
| Dynamique (MobSF, Frida) | Fichiers créés, trafic réel, valeurs en mémoire | Code mort non exécuté |
| **Combinée** | **Vue complète** | — |

### 9.2 SharedPreferences — Vulnérabilité classique

```java
// Code vulnérable
SharedPreferences prefs = getSharedPreferences("creds", MODE_PRIVATE);
prefs.edit()
    .putString("user", username)
    .putString("password", password)  // stocké en clair !
    .apply();

// Correction
EncryptedSharedPreferences.create(
    "creds",
    MasterKey.Builder(context).build(),
    context,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
);
```

###  9.3 Proxy HTTPS MobSF

MobSF installe son certificat CA directement dans le store système de l'émulateur (pas utilisateur) — ce qui lui permet d'intercepter le trafic HTTPS même des apps avec Network Security Config restrictive.

---

## 10. Récapitulatif des commandes

### Setup émulateur

```bash
# Créer l'AVD (Android Studio AVD Manager ou ligne de commande)
sdkmanager "system-images;android-29;default;x86_64"
avdmanager create avd -n MobSF_DIVA_API_29 \
  -k "system-images;android-29;default;x86_64" --device "pixel_5"

# Lancer via le script MobSF
./scripts/start_avd.sh MobSF_DIVA_API_29   # Linux/Mac
.\scripts\start_avd.ps1                     # Windows

# Vérifier
adb devices
```

### MobSF Docker

```bash
# Télécharger
docker pull opensecurity/mobile-security-framework-mobsf:latest

# Lancer (Windows/Mac)
docker run -it --rm \
  -p 8000:8000 \
  -e MOBSF_ANALYZER_IDENTIFIER=emulator-5554 \
  opensecurity/mobile-security-framework-mobsf:latest

# Lancer (Kali Linux)
docker run -it --rm \
  --net=host \
  -e MOBSF_ANALYZER_IDENTIFIER=emulator-5554 \
  opensecurity/mobile-security-framework-mobsf:latest
```

### Accès

```
Interface web : http://127.0.0.1:8000
Login         : mobsf / mobsf
```

### Exploration depuis ADB (complémentaire)

```bash
# Voir les SharedPreferences
adb shell run-as jakhar.aseem.diva \
  cat /data/data/jakhar.aseem.diva/shared_prefs/*.xml

# Voir la base SQLite
adb shell run-as jakhar.aseem.diva \
  sqlite3 /data/data/jakhar.aseem.diva/databases/divaDB \
  "SELECT * FROM myuser;"

# Lancer une activité exportée
adb shell am start -n jakhar.aseem.diva/.APIActivity
```
---
