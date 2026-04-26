# LAB6_security
# Rapport d'Analyse Statique — app-debug-androidTest.apk

---

## ⚠️ Note préliminaire

En raison de contraintes matérielles (ressources système insuffisantes pour faire tourner une machine virtuelle), l'environnement **Mobexler** n'a pas pu être installé localement pour la réalisation de ce TP.

En remplacement, l'analyse statique de l'APK a été effectuée via la plateforme en ligne **[MobSF Live](https://mobsf.live)**, qui expose une instance publique de Mobile Security Framework (MobSF) sans nécessiter d'installation locale. Cette approche couvre l'intégralité des fonctionnalités requises pour la partie **analyse statique** du TP et produit des résultats identiques à ceux obtenus via une installation locale de MobSF.

---

## 1. Informations générales

| Champ | Valeur |
|---|---|
| **APK analysé** | `app-debug-androidTest.apk` |
| **Package** | `com.example.app.test` |
| **Date d'analyse** | 26 Avril 2026, 16h21 |
| **Analyste** | Iliass Chbani |
| **Taille** | 0.53 MB |
| **MD5** | `7391f44599620441c5a46a8074b94616` |
| **SHA-1** | `1e65571c3d4e3b7dfebc195170769c5385a19292` |
| **SHA-256** | `ddb5e280cd20efacb1f3ec8896b9faef7c365d4d4577dccf1e3e33555de11198` |
| **Target SDK** | 36 |
| **Min SDK** | 24 (Android 7.0) |
| **Outil utilisé** | MobSF v4.5.0 via mobsf.live (instance publique) |
| **Score de sécurité** | **41/100 — Risque MOYEN (Grade B)** |
| **Rapport brut MobSF** | [Rapport_MobSF.pdf](https://github.com/user-attachments/files/27102676/Rapport_MobSF.pdf) |

---

## 2. Résumé exécutif

L'analyse statique de l'application `app-debug-androidTest.apk` (package `com.example.app.test`) révèle un **niveau de risque moyen**, avec un score de sécurité de **41/100**. L'application présente **3 vulnérabilités de sévérité HIGH** et **5 de sévérité MEDIUM**. Les principales problématiques concernent l'utilisation d'un **certificat de signature de débogage** en lieu et place d'un certificat de production, le **mode débogage activé** dans le manifeste Android (`android:debuggable=true`), la **compatibilité avec des versions Android obsolètes et non patchées** (minSdk=24), ainsi que la présence d'**activités exportées sans protection**. Des pratiques de code risquées ont également été identifiées, notamment l'usage d'un générateur de nombres aléatoires insuffisamment sécurisé et la création de fichiers temporaires pouvant contenir des données sensibles.

---

## 3. Vulnérabilités critiques

---

### 3.1 Application signée avec un certificat de débogage

- **Sévérité :** 🔴 HIGH
- **Catégorie MASVS :** MSTG-CODE-1 (Signature de l'application)
- **Description :** L'APK est signé avec le certificat de débogage Android par défaut (`CN=Android Debug, O=Android, C=US`). Un certificat de débogage est public et partagé entre tous les environnements de développement Android. Il ne doit jamais être utilisé pour distribuer une application en production.
- **Preuve :** Section *Certificate Information* — `X.509 Subject: CN=Android Debug, O=Android, C=US` / `Issuer: CN=Android Debug, O=Android, C=US`
- **Impact :** N'importe quel attaquant peut recompiler et signer une version modifiée (malveillante) de l'application avec le même certificat de débogage, facilitant ainsi la distribution de versions trojanisées.
- **Remédiation :** Générer un keystore de production privé et signer l'APK avec ce certificat avant toute distribution. Ne jamais utiliser le certificat de débogage en dehors de l'environnement de développement.

---

### 3.2 Mode débogage activé en production (`android:debuggable=true`)

- **Sévérité :** 🔴 HIGH
- **Catégorie MASVS :** MSTG-RESILIENCE-2
- **Description :** L'attribut `android:debuggable="true"` est défini dans le fichier `AndroidManifest.xml`. Cela permet à tout utilisateur possédant un accès physique ou via ADB de connecter un débogueur à l'application en cours d'exécution.
- **Preuve :** Manifest Analysis — *"Debug Enabled For App [android:debuggable=true]"* (HIGH)
- **Impact :** Un attaquant peut attacher un débogueur (ex. : `jdb`, `Android Studio Debugger`) pour inspecter la mémoire en temps réel, dumper la stack trace, modifier le flux d'exécution, extraire des secrets ou contourner des vérifications de sécurité.
- **Remédiation :** Définir `android:debuggable="false"` dans le manifeste pour les builds de production. Utiliser les variantes de build Gradle (`release`) pour s'assurer que le débogage est automatiquement désactivé.

---

### 3.3 Compatibilité avec des versions Android vulnérables (minSdk=24)

- **Sévérité :** 🔴 HIGH
- **Catégorie MASVS :** MSTG-PLATFORM-1
- **Description :** La valeur `minSdk=24` (Android 7.0 — Nougat) permet l'installation de l'application sur des appareils qui ne reçoivent plus de mises à jour de sécurité de Google. Ces versions contiennent de nombreuses vulnérabilités connues et non corrigées.
- **Preuve :** Manifest Analysis — *"App can be installed on a vulnerable unpatched Android version 7.0 [minSdk=24]"* (HIGH)
- **Impact :** Les utilisateurs installant l'application sur des appareils sous Android 7.x ou 8.x sont exposés à des exploits système connus, pouvant compromettre l'ensemble de l'appareil au-delà de l'application elle-même.
- **Remédiation :** Relever le `minSdk` à **29 (Android 10)** minimum, conformément aux recommandations de Google pour bénéficier d'un support de sécurité raisonnable.

---

### 3.4 Activités exportées sans protection

- **Sévérité :** 🟠 WARNING
- **Catégorie MASVS :** MSTG-PLATFORM-3
- **Description :** Deux activités sont déclarées avec `android:exported=true` sans mécanisme de protection (pas de `permission` requise ni de `intent-filter` restrictif) :
  - `androidx.test.core.app.InstrumentationActivityInvoker$BootstrapActivity`
  - `androidx.test.core.app.InstrumentationActivityInvoker$EmptyActivity`
- **Preuve :** Manifest Analysis — entrées 4 et 5 (WARNING)
- **Impact :** Toute application tierce installée sur le même appareil peut invoquer ces activités directement, ce qui peut entraîner une élévation de privilèges, une confusion d'intent, ou des détournements de flux applicatif.
- **Remédiation :** Si ces activités ne sont pas destinées à être accessibles par d'autres applications, passer `android:exported="false"`. Si l'export est nécessaire, définir une `android:permission` personnalisée de niveau `signature` pour restreindre l'accès.

---

### 3.5 Utilisation d'un générateur de nombres aléatoires non sécurisé

- **Sévérité :** 🟠 WARNING
- **Catégorie MASVS :** MSTG-CRYPTO-6
- **Référence CWE :** CWE-330 — *Use of Insufficiently Random Values*
- **OWASP Top 10 Mobile :** M5 — Insufficient Cryptography
- **Description :** Le code source utilise une instance de `Random` (pseudo-aléatoire) là où `SecureRandom` devrait être employé pour toute opération liée à la sécurité.
- **Preuve :** Code Analysis — fichier `org/junit/runner/manipulation/Ordering.java` (WARNING)
- **Impact :** Les valeurs générées sont prévisibles et peuvent être reconstituées par un attaquant, compromettant ainsi toute opération cryptographique ou de génération de tokens qui en dépend.
- **Remédiation :** Remplacer toutes les occurrences de `java.util.Random` par `java.security.SecureRandom` pour les usages liés à la sécurité (tokens, salts, nonces, etc.).

---

## 4. Autres observations

- L'application journalise des informations via des appels à des méthodes de log (fichiers `BaseTestRunner.java`, `Version.java`, `TestRunner.java`). Des données sensibles ne doivent jamais transiter par les logs (MSTG-STORAGE-3 / CWE-532).
- Des fichiers temporaires sont créés par `org/junit/rules/TemporaryFolder.java` avec des permissions par défaut potentiellement permissives (MSTG-STORAGE-2 / CWE-276).
- Le flag `android:allowBackup` est absent du manifeste : par défaut, Android autorise la sauvegarde des données applicatives via ADB, ce qui peut exposer des données locales.
- Du code anti-détection de VM et anti-debug a été identifié dans `classes.dex` (`Build.FINGERPRINT`, `Build.MODEL`, `Build.HARDWARE`, `Debug.isDebuggerConnected()`). Bien que cela puisse être légitime dans un contexte de test, cela mérite attention dans un contexte de production.

---

## 5. Recommandations prioritaires

1. **[CRITIQUE]** Signer l'APK avec un certificat de production privé et supprimer tout usage du certificat de débogage pour les builds destinés à être distribués.
2. **[CRITIQUE]** Désactiver le mode débogage en production en forçant `android:debuggable="false"` dans le manifeste (ou via la configuration Gradle `release`).
3. **[ÉLEVÉE]** Relever le `minSdkVersion` à **29 (Android 10)** pour exclure les appareils ne bénéficiant plus de mises à jour de sécurité.
4. **[ÉLEVÉE]** Passer `android:exported="false"` sur les activités internes non destinées à être invoquées par des applications tierces, ou les protéger par une permission de niveau `signature`.
5. **[MOYENNE]** Remplacer `java.util.Random` par `java.security.SecureRandom` dans tous les contextes liés à la sécurité, et supprimer ou filtrer les appels de log en production.

---

## 6. Annexes techniques

### 6.1 Permissions déclarées

| Permission | Niveau | Description |
|---|---|---|
| `android.permission.REORDER_TASKS` | Normal | Permet de déplacer des tâches au premier plan. Peut être exploitée pour forcer une application malveillante au premier plan. |

Aucune permission dangereuse (`dangerous`) n'a été détectée. Aucun chevauchement avec les permissions abusées par des malwares connus (0/25 Malware Permissions, 0/44 Other Common Permissions).

---

### 6.2 Composants exportés identifiés

| Composant | Type | Exporté | Protégé |
|---|---|---|---|
| `InstrumentationActivityInvoker$BootstrapActivity` | Activity | ✅ Oui | ❌ Non |
| `InstrumentationActivityInvoker$EmptyActivity` | Activity | ✅ Oui | ❌ Non |
| `InstrumentationActivityInvoker$EmptyFloatingActivity` | Activity (Main) | — | — |

Aucun Service, Receiver ou Provider exporté détecté.

---

### 6.3 Endpoints et domaines détectés

Aucun endpoint réseau ni domaine externe n'a été détecté lors de l'analyse statique. Aucune configuration de sécurité réseau (Network Security Config) n'a été identifiée dans le manifeste.

---

### 6.4 Synthèse des findings MobSF

| Sévérité | Nombre |
|---|---|
| 🔴 HIGH | 3 |
| 🟠 WARNING / MEDIUM | 5 |
| ℹ️ INFO | 1 |
| ✅ SECURE | 1 |
| 🔥 HOTSPOT | 0 |

---

*Rapport généré à partir de l'analyse MobSF v4.5.0 — mobsf.live — 26 Avril 2026*
