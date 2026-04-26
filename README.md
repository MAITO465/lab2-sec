# lab2-sec
# 🔐 Rapport de TP — Rooting Android, Verified Boot & Sécurité Mobile
 
> **Cours :** Sécurité des applications mobiles
> **Auteur :** *AIT OURAJLI Mohamed*
> **Date :** *26/04/2026*
> **Établissement :** ENSA de Marrakech
> **Environnement :** Android Studio — AVD (émulateur)
 
---
 
## 📑 Table des matières
 
1. [Étape 1 — Obtenir les droits root sur l'AVD](#étape-1--obtenir-les-droits-root-sur-lavd)
2. [Étape 2 — Fiche de périmètre](#étape-2--fiche-de-périmètre)
3. [Étape 3 — Initialisation d'un AVD vierge](#étape-3--initialisation-dun-avd-vierge)
4. [Étape 4 — Déploiement de l'application de test](#étape-4--déploiement-de-lapplication-de-test)
5. [Étape 5 — Définition de 3 scénarios fonctionnels](#étape-5--définition-de-3-scénarios-fonctionnels)
6. [Étape 6 — Vue d'ensemble de la sécurité Android](#étape-6--vue-densemble-de-la-sécurité-android)
7. [Étape 7 — Verified Boot](#étape-7--verified-boot)
8. [Étape 8 — AVB (Android Verified Boot)](#étape-8--avb-android-verified-boot)
9. [Étape 9 — Qu'est-ce que le rooting ?](#étape-9--quest-ce-que-le-rooting-)
10. [Étape 10 — Utilité en contexte de laboratoire](#étape-10--utilité-en-contexte-de-laboratoire)
11. [Étape 11 — Matrice des risques](#étape-11--matrice-des-risques)
12. [Étape 12 — Contre-mesures](#étape-12--contre-mesures)
13. [Étape 13 — OWASP MASVS](#étape-13--owasp-masvs)
14. [Étape 14 — OWASP MASTG](#étape-14--owasp-mastg)
15. [Étape 15 — Récapitulatif des commandes](#étape-15--récapitulatif-des-commandes)
16. [Étape 16 — Fiche de traçabilité environnement](#étape-16--fiche-de-traçabilité-environnement)
17. [Étape 17 — Réinitialisation de l'AVD](#étape-17--réinitialisation-de-lavd)
18. [Étape 18 — Réinitialisation du device de labo](#étape-18--réinitialisation-du-device-de-labo)
19. [Étape 19 — Checklist de clôture](#étape-19--checklist-de-clôture)
---
 
## Étape 1 — Obtenir les droits root sur l'AVD
 
### 🎯 Motivation
Acquérir des **privilèges élevés sur un environnement éphémère** (AVD) permet d'étudier les effets du rooting sur les mécanismes de contrôle d'intégrité, sans exposer un appareil réel. Un système rooté ouvre l'accès à des zones habituellement restreintes — comme disposer d'un trousseau donnant accès à toutes les zones d'un bâtiment, y compris les zones sécurisées.
 
### 📋 Prérequis
- AVD lancé depuis Android Studio
- L'émulateur est bien détecté via `adb devices`
 
### 🛠️ Commandes (émulateur uniquement)
 
```bash
emulator -avd emulator-5554 -writable-system
adb root        # Lance adbd avec les droits administrateur si l'image le supporte
adb remount     # Monte /system en mode lecture/écriture
```
 
 
> **Explication :** `adb root` relance le serveur ADB en mode administrateur, tandis que `adb remount` rend la partition système modifiable — une opération normalement bloquée pour des raisons de sécurité.
 
### 🔍 Vérifications post-root
 
```bash
adb shell id                                   # Chercher uid=0(root)
adb shell getprop ro.boot.verifiedbootstate    # "orange"/"yellow" si verity désactivé
adb shell getprop ro.boot.veritymode           # État de dm-verity
adb shell getprop ro.boot.vbmeta.device_state  # État du vbmeta
adb shell "su -c id"                           # Vérifie la disponibilité de su
```
 
### 📊 Interprétation
 
| Résultat | Signification |
|----------|--------------|
| `uid=0(root)` | Droits root confirmés |
| `verifiedbootstate = orange/yellow` | Intégrité système non assurée |
| `su` répond | Accès superutilisateur actif dans le shell |
 
### 🔓 Désactivation de dm-verity
 
```bash
adb disable-verity   # Désactive la vérification d'intégrité dm-verity
adb reboot           # Redémarre l'émulateur
adb remount          # Monte de nouveau après redémarrage
```
 
> **Note de sécurité :** désactiver dm-verity revient à retirer le scellé d'un équipement : les modifications deviennent possibles, mais toute garantie d'authenticité disparaît.
 
### 📝 Capture des journaux système
 
```bash
adb logcat -d | tail -n 200 > logcat_root_check.txt
```
 
### ⚠️ Avertissement
**Ne jamais effectuer ces manipulations sur un appareil personnel.** Intervenir sur le bootloader ou flasher des partitions peut rendre l'appareil définitivement inutilisable et annule la garantie constructeur.
 
Pour un device de laboratoire en fastboot déverrouillé uniquement :
 
```bash
fastboot oem device-info
fastboot getvar avb_boot_state
fastboot boot magisk_patched.img   # Démarrage temporaire sans flashage
```
 
---
 
## Étape 2 — Fiche de périmètre
 
| Champ | Valeur |
|-------|--------|
| **Application** | MyTestApp v1.0 |
| **Support** | AVD (émulateur Android Studio) |
| **Objectif** | Analyser le rooting et ses conséquences sur la sécurité système |
| **Données** | Exclusivement fictives |
| **Réseau** | Environnement de test isolé |
 
> **Pourquoi c'est indispensable :** la fiche périmètre joue le rôle d'un cahier des charges : elle délimite précisément le champ d'intervention. Sans périmètre établi, aucun audit ne peut être considéré comme fiable.
 
---
 
## Étape 3 — Initialisation d'un AVD vierge
 
### 🚀 Procédure
1. Ouvrir **Android Studio → Device Manager → Start** (ou créer un AVD avec API 29 minimum)
2. Vérifier : écran d'accueil Android affiché, **aucun compte connecté**
3. Contrôle ADB :
   ```bash
   adb devices
   ```
 
### 💡 Recommandations
- Préférer les **versions récentes d'Android (API 29+)** pour disposer des protections modernes.
- **Ne jamais réutiliser un AVD précédemment utilisé** avec des applications résiduelles, au risque de contaminer les résultats.
---
 
## Étape 4 — Déploiement de l'application de test
 
```bash
adb install app-debug.apk
```
 
*Alternative :* bouton **Run ▶️** dans Android Studio.
 
### ✅ Points de contrôle
- L'application démarre sans erreur
- Les scénarios de base sont exécutables
- **Version consignée** : *MyTestApp v1.0* — les comportements de sécurité peuvent différer d'une version à l'autre.
---
 
## Étape 5 — Définition de 3 scénarios fonctionnels
 
Les scénarios représentent les **flux de référence** de l'application : simples, reproductibles, avec des entrées fixes.
 
### 📋 Tableau des scénarios
 
| # | Scénario | Entrée | Résultat attendu |
|---|----------|--------|------------------|
| 1 | **Affichage de l'écran principal** | Lancement de l'app | Liste de 5 produits dans l'ordre défini |
| 2 | **Filtrage par mot-clé** | Saisie de `Smart` | 1 résultat : "Smartphone" |
| 3 | **Consultation d'un article** | Tap sur "Smartphone" | Fiche avec description + prix `599,00 €` |
 
### 🧪 Automatisation via ADB
 
```bash
# Scénario 1
adb shell am start -n ma.ensa.mytestapp/.MainActivity
adb shell screencap -p /sdcard/s1.png && adb pull /sdcard/s1.png scenario1_accueil.png
 
# Scénario 2
adb shell input tap 540 280
adb shell input text "Smart"
adb shell screencap -p /sdcard/s2.png && adb pull /sdcard/s2.png scenario2_recherche.png
 
# Scénario 3
adb shell input tap 540 674
adb shell screencap -p /sdcard/s3.png && adb pull /sdcard/s3.png scenario3_detail.png
```
 
## Étape 6 — Vue d'ensemble de la sécurité Android
 
La sécurité d'Android repose sur plusieurs niveaux complémentaires. Le **sandboxing applicatif** confine chaque application dans un espace d'exécution isolé : les données d'une application ne sont pas accessibles par une autre. Le **modèle de permissions** soumet l'accès aux ressources sensibles (caméra, contacts, stockage) à une autorisation explicite de l'utilisateur. Enfin, l'**intégrité du système** est préservée par Verified Boot, SELinux et le verrouillage du bootloader, qui empêchent les altérations non autorisées. Ces mécanismes sont interdépendants, et **le rooting fragilise cet équilibre** en court-circuitant les permissions et en affaiblissant l'isolation.
 
🔗 Source : https://source.android.com/docs/security
 
---
 
## Étape 7 — Verified Boot
 
### 🎯 Objectif
**S'assurer que le système qui s'exécute correspond bien à celui produit par le fabricant**, sans altération. Chaque composant est validé par une signature cryptographique avant d'être chargé.
 
### 🔗 Chaîne de confiance (Chain of Trust)
Mécanisme de **vérifications successives** où chaque composant authentifie le suivant avant de lui transmettre le contrôle — depuis la racine matérielle immuable jusqu'à la partition système.
 
### 🏰 Pourquoi le démarrage sécurisé est fondamental
Un démarrage compromis peut **neutraliser l'ensemble des couches de sécurité ultérieures** — sandboxing, permissions, chiffrement. C'est le maillon initial de la chaîne : si la base est compromise, tout l'édifice devient vulnérable.
 
### 🧪 Vérification
 
```bash
adb shell getprop ro.boot.verifiedbootstate    # Attendu "green" sur image officielle
```
 
### 🎨 Signification des états
 
| État | Signification | Comportement |
|------|--------------|--------------|
| 🟢 **Green** | Système authentifié et intact | Démarrage normal |
| 🟡 **Yellow / Orange** | Système modifié (clé personnalisée reconnue) | Avertissement affiché |
| 🔴 **Red** | Intégrité compromise | Démarrage bloqué ou alerte critique |
 
🔗 Source : https://source.android.com/docs/security/features/verifiedboot
 
---
 
## Étape 8 — AVB (Android Verified Boot)
 
**AVB** constitue la version 2.0 de Verified Boot. Il apporte une **vérification d'intégrité au niveau des partitions** via des métadonnées signées (`vbmeta`), ainsi qu'une **protection anti-rollback** qui empêche l'installation de versions obsolètes du système susceptibles de contenir des failles corrigées.
 
### 🧪 Vérification (fastboot — device de labo uniquement)
 
```bash
fastboot getvar avb_boot_state
```
 
🔗 Source : https://source.android.com/docs/security/features/verifiedboot/avb
 
---
 
## Étape 9 — Qu'est-ce que le rooting ?
 
Le **rooting** désigne l'obtention des **privilèges superutilisateur** sur un appareil Android, permettant un accès administrateur complet. Cette élévation de droits **neutralise les protections natives et rompt la chaîne de confiance** du système. En contexte de laboratoire, elle permet d'accéder à des niveaux d'observation plus profonds (forensique, analyse de code malveillant). En raison des risques associés, elle exige un cadre strict : **environnement isolé, traçabilité et remise à zéro obligatoire**.
 
### 📜 Origine du terme
Le mot **"root"** provient d'UNIX, où l'administrateur système — doté de droits illimités — est nommé d'après la racine (`/`) de l'arborescence de fichiers. Android étant fondé sur Linux, rooter un appareil revient à **endosser ce rôle d'administrateur omnipotent**.
 
---
 
## Étape 10 — Utilité en contexte de laboratoire
 
> **Dans un cadre strictement autorisé**, des droits élevés permettent de :
 
- ✅ **Consulter des artefacts système normalement inaccessibles** (ex. : `/data/data/`)
- ✅ **Analyser les comportements bas niveau d'une application** (hooks, mémoire, IPC)
- ✅ **Évaluer la robustesse du stockage face à un attaquant disposant de droits root**
### 💡 Exemple concret
Avec les droits root, il est possible de déterminer si une application :
- ❌ **Mauvaise pratique** : s'appuie uniquement sur le sandboxing Android pour protéger ses données
- ✅ **Bonne pratique** : chiffre elle-même ses informations sensibles
### ⚖️ Cadre légal
Dans certaines juridictions, le rooting peut **enfreindre les conditions d'utilisation du constructeur** ou des dispositions légales relatives aux protections techniques. **Une autorisation formelle est indispensable** avant toute intervention de ce type.
 
---
 
## Étape 11 — Matrice des risques
 
| # | Risque identifié | Conséquence possible |
|---|-----------------|----------------------|
| 1 | **Intégrité non garantie** | Résultats d'audit biaisés |
| 2 | **Surface d'attaque élargie** (hors labo) | Exposition à des menaces externes |
| 3 | **Données sensibles accessibles** | Risque de violation de la vie privée |
| 4 | **Instabilité système** | Résultats non reproductibles |
| 5 | **Confusion comptes personnels/test** | Fuite d'informations privées |
| 6 | **Nettoyage incomplet en fin de séance** | Persistance de données résiduelles |
| 7 | **Réseau non cloisonné** | Impact involontaire sur des systèmes tiers |
| 8 | **Traçabilité inexistante** | Impossibilité d'auditer ou reproduire les tests |
 
> **Principe fondamental :** chaque risque recensé doit être associé à une mesure corrective — c'est le socle de toute démarche de gestion des risques en cybersécurité.
 
---
 
## Étape 12 — Contre-mesures
 
| # | Mesure mise en place | Risques couverts |
|---|---------------------|------------------|
| 1 | **Réseau isolé** — aucune communication non maîtrisée | R2, R7 |
| 2 | **Données exclusivement fictives** — aucune donnée réelle | R3, R5 |
| 3 | **AVD/Device dédié** aux tests de sécurité uniquement | R2, R5 |
| 4 | **Snapshot / Wipe** systématique en fin de session | R6 |
| 5 | **Journal de configuration détaillé** pour reproductibilité | R4, R8 |
| 6 | **Aucun compte personnel** utilisé lors des tests | R3, R5 |
| 7 | **Contrôle strict des APK** installées | R2, R3 |
| 8 | **Horodatage et captures** de chaque étape clé | R8 |
 
---
 
## Étape 13 — OWASP MASVS
 
> **MASVS** (*Mobile Application Security Verification Standard*) = référentiel OWASP définissant **ce qui doit être vérifié** dans une application mobile.
 
### 🔐 STORAGE-1 — Stockage des données sensibles
Les **données critiques** (clés API, mots de passe, tokens) doivent être stockées via des **mécanismes de chiffrement adaptés** (Android Keystore, EncryptedSharedPreferences) — jamais en clair sur le disque.
 
### 🌐 NETWORK-1 — Sécurisation des communications
L'ensemble des échanges réseau doit transiter via **TLS correctement configuré**, avec une **validation rigoureuse des certificats** (interdiction de `TrustAllCerts`, certificate pinning recommandé).
 
### 🛠️ Application lors des tests avec droits root
- Inspection des fichiers dans `/data/data/[package]/`
- Interception du trafic réseau via un proxy HTTPS
🔗 Source : https://mas.owasp.org/MASVS/
 
---
 
## Étape 14 — OWASP MASTG
 
> **MASTG** (*Mobile Application Security Testing Guide*) = guide OWASP décrivant **comment mettre en œuvre** les vérifications MASVS.
 
### 🧪 Test 1 — Analyse des SharedPreferences
Rechercher des **données sensibles stockées en clair** dans les fichiers de préférences :
```bash
adb shell
run-as ma.ensa.mytestapp
cat /data/data/ma.ensa.mytestapp/shared_prefs/*.xml
```
 
### 🧪 Test 2 — Détection de fuites dans les logs
Identifier des **informations confidentielles exposées** dans `logcat` pendant l'exécution :
```bash
adb logcat -d | grep -iE "password|token|key|secret"
```
 
### 💡 Note technique
Avec les droits root, le répertoire **`/data/data/`** devient accessible — il contient les données privées de toutes les applications, normalement cloisonnées par le sandboxing.
 
🔗 Source : https://mas.owasp.org/MASTG/
 
---
 
## Étape 15 — Récapitulatif des commandes
 
### 🔧 Commandes principales
 
```bash
adb devices
adb root
adb remount
adb shell id
adb shell getprop ro.boot.veritymode
adb shell getprop ro.boot.vbmeta.device_state
adb shell "su -c id"
```
 
### 🆘 Erreur fréquente
Si `adb root` retourne :
> *"adbd cannot run as root in production builds"*
 
Ce comportement est **attendu sur les builds de production**. Il est nécessaire d'utiliser un **émulateur** ou un appareil de laboratoire configuré pour le root.
 
### 🔓 Désactivation de verity
 
```bash
adb disable-verity    # Désactiver dm-verity
adb reboot            # Redémarrer l'émulateur
adb remount           # Monter après redémarrage
adb logcat -d | tail -n 200 > logcat_root_check.txt
```
 
### 🔬 Fastboot (labo uniquement)
 
```bash
fastboot oem device-info
fastboot getvar avb_boot_state
fastboot boot magisk_patched.img   # Démarrage temporaire sans flashage
```
 
### 🪄 Magisk en bref
**Magisk** est un outil de rooting qui modifie l'image boot **sans toucher à la partition système** ("systemless root"). Cette technique permet dans certains cas de contourner les mécanismes de détection du root.
 
---
 
## Étape 16 — Fiche de traçabilité environnement
 
| Champ | Valeur |
|-------|--------|
| **Date / Auteur** | *[À compléter]* |
| **Support** | AVD Pixel 4 API 33 |
| **Version Android / API** | Android 13 (API 33) |
| **Application + version** | MyTestApp v1.0 |
| **Scénario 1** | Affichage écran principal → Liste des 5 produits |
| **Scénario 2** | Recherche "Smart" → 1 résultat (Smartphone) |
| **Scénario 3** | Tap sur Smartphone → Détail avec prix 599,00 € |
| **Observations** | `verifiedbootstate = orange` après disable-verity, `uid=0(root)` confirmé |
| **Limites** | L'AVD ne simule pas AVB matériel ; fastboot réel non disponible |
| **Reset effectué** | ✅ Oui — Wipe Data via Device Manager |
| **Preuve du reset** | Capture de l'assistant de configuration initial |
  
---
 
## Étape 17 — Réinitialisation de l'AVD
 
### 🖥️ Via l'interface graphique
**Android Studio → Device Manager → ⋮ → Wipe Data** (ou Delete + Recreate)
 
### 💻 Via la ligne de commande
 
```bash
adb emu avd stop
adb emu avd wipe-data
```
 
### ⚠️ Importance du reset
Ne pas remettre à zéro l'environnement après les tests expose la session suivante à des données résiduelles susceptibles de fausser les résultats, voire de compromettre la confidentialité.
 
 
---
 
## Étape 18 — Réinitialisation du device de labo (si utilisé)
 
### 🔄 Procédure
1. **Paramètres système → Réinitialisation usine → Redémarrer**
2. Contrôler l'**absence de comptes, profils ou certificats résiduels**
3. Preuve : assistant de configuration initial visible à l'écran
### 🔍 Vérification complémentaire
S'assurer de l'**absence de certificats racine personnalisés** dans les paramètres de sécurité — leur présence pourrait permettre l'interception de trafic HTTPS.
 
### 🔧 Option fastboot (labo uniquement)
 
```bash
fastboot erase userdata    # Puis redémarrer
```
 
> **Différence clé :** la réinitialisation via les paramètres peut laisser des traces dans certaines partitions ; `fastboot erase` réalise quant à lui un **effacement bas niveau plus exhaustif**.
 
---
 
### 🔗 Schéma — Chaîne de confiance
 
```
┌───────────────────────────┐
│  Racine de confiance      │  ← Clé immuable inscrite en matériel
│  (Root of Trust)          │
└────────────┬──────────────┘
             │ vérifie
             ▼
┌───────────────────────────┐
│      Bootloader           │
└────────────┬──────────────┘
             │ vérifie
             ▼
┌───────────────────────────┐
│        Noyau (Kernel)     │
└────────────┬──────────────┘
             │ vérifie
             ▼
┌───────────────────────────┐
│   Partition /system       │
└────────────┬──────────────┘
             │ vérifie
             ▼
┌───────────────────────────┐
│   Applications & Données  │
└───────────────────────────┘
```
 
---
 
## Étape 19 — Checklist de clôture
 
### ✅ Avant les tests (Planification)
- [ ] Périmètre rédigé et validé
- [ ] AVD vierge créé
- [ ] Application de test installée
- [ ] 3 scénarios documentés
- [ ] Versions Android et application consignées
### ✅ Après les tests (Vérification & Actions correctives)
- [ ] Données de test supprimées
- [ ] Reset effectué (wipe AVD ou réinitialisation device)
- [ ] Preuve de reset archivée
- [ ] Rapport et traçabilité déposés sur GitHub
- [ ] **Aucun compte personnel utilisé**
> **Démarche professionnelle :** cette checklist s'inscrit dans le cycle **PDCA** (*Plan – Do – Check – Act*) : planification, exécution, vérification, remédiation.
 
---
 
## 📎 Annexes
 
### 🔗 Références officielles
- [Android Security Documentation](https://source.android.com/docs/security)
- [Verified Boot](https://source.android.com/docs/security/features/verifiedboot)
- [Android Verified Boot (AVB)](https://source.android.com/docs/security/features/verifiedboot/avb)
- [OWASP MASVS](https://mas.owasp.org/MASVS/)
- [OWASP MASTG](https://mas.owasp.org/MASTG/)

 
---
 
> ⚠️ **Avertissement :** L'ensemble des manipulations décrites dans ce rapport a été réalisé **exclusivement dans un environnement de laboratoire autorisé** (émulateur Android Studio). Aucun appareil personnel n'a été modifié et aucune donnée réelle n'a été exposée. La reproduction de ces opérations sur un appareil en production enfreindrait les conditions d'utilisation du constructeur et exposerait l'appareil à des risques importants.
