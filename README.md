# Rapport d'Expertise : Élévation de Privilèges et Audit d'Intégrité (Lab 2)
 
**Analyste :** Mohamed AIT OURAJLI  
**Date :** 26/04/2026  
**Établissement :** ENSA Marrakech  
**Environnement :** Android Studio (AVD) - API 33  
 
---
 
## 1. Objectifs et Périmètre de l'Audit
 
Cette mission d'audit vise à évaluer la robustesse des mécanismes de protection d'Android face à un utilisateur disposant des privilèges superutilisateur (**root**). L'objectif est de comprendre comment la rupture de la chaîne de confiance affecte l'isolation des données (*sandboxing*).
 
### Fiche de Périmètre
 
| Champ | Détail |
|-------|--------|
| **Cible** | MyTestApp v1.0 (APK de test) |
| **Environnement** | Instance AVD isolée (aucun compte Google lié) |
| **Vecteurs** | ADB (Android Debug Bridge), Shell Root |
| **Éthique** | Données 100% fictives, réinitialisation complète après session |
 
---
 
## 2. Accès Root et Altération du Système
 
### Obtention des privilèges
 
L'élévation de privilèges sur l'émulateur permet de contourner les restrictions de sécurité du noyau Linux. La procédure suivante a été appliquée pour monter la partition système en mode écriture :
 
```bash
# Lancement de l'AVD avec système modifiable
emulator -avd Pixel_Lab_API_33 -writable-system
 
# Basculement du démon ADB en mode root
adb root 
 
# Remontage des partitions pour modification
adb remount
```
 
### Indicateurs de succès
 
Après exécution, les tests de contrôle suivants valident l'état de l'environnement :
 
- **Identité** : `adb shell id` renvoie `uid=0(root)`.
- **Intégrité** : `verifiedbootstate` passe à l'état orange, indiquant que la signature du bootloader n'est plus verrouillée.
---
 
## 3. Analyse de la Chaîne de Confiance (Verified Boot)
 
La sécurité d'Android repose sur la validation successive de chaque composant au démarrage.
 
- **Verified Boot (AVB)** : Ce mécanisme assure que le code exécuté provient du constructeur.
- **Rupture de confiance** : En utilisant `adb disable-verity`, nous avons neutralisé les contrôles d'intégrité des partitions. Cela permet d'injecter des binaires (comme `su`) mais expose l'appareil à des modifications persistantes indétectables par le système d'exploitation.
---
 
## 4. Scénarios Fonctionnels de Référence
 
Avant l'audit intrusif, trois scénarios "métier" ont été documentés pour servir de base de test :
 
| ID | Scénario | Action | Résultat Attendu |
|----|---------|---------|----|
| S1 | Inventaire | Lancement de l'app | Liste de 5 produits affichée |
| S2 | Recherche | Saisie de "Smart" | Filtrage sur l'item "Smartphone" |
| S3 | Consultation | Clic sur "Smartphone" | Fiche produit avec prix (599,00 €) |
 
---
 
## 5. Audit de Sécurité (Standards OWASP)
 
L'accès root permet d'appliquer les méthodologies du MASTG (Mobile Application Security Testing Guide).
 
### Stockage Local (MASVS STORAGE-1)
 
Grâce aux droits root, nous avons pu inspecter le répertoire privé de l'application :
 
```bash
adb shell ls -la /data/data/ma.ensa.mytestapp/
```
 
L'audit vérifie si des données sensibles sont présentes en clair dans les fichiers XML de `shared_prefs`.
 
### Fuite d'informations (Logcat)
 
Extraction des journaux système pour identifier des fuites de données sensibles pendant l'exécution :
 
```bash
adb logcat -d | grep -iE "token|auth|password" > report_leaks.txt
```
 
---
 
## 6. Matrice des Risques et Remédiation
 
| Risque Identifié | Impact | Mesure Corrective Appliquée |
|------------------|--------|---------------------------|
| Rupture du Sandbox | Accès aux données privées | Chiffrement via Android Keystore |
| Persistance de données | Fuite d'informations post-audit | Procédure de "Wipe Data" systématique |
| Instabilité Système | Résultats non reproductibles | Utilisation de Snapshots AVD |
| Accès réseau ouvert | Communication non maîtrisée | Isolation réseau (Host-only) |
 
---
 
## 7. Clôture et Restauration
 
Conformément aux bonnes pratiques de cybersécurité, l'environnement a été remis à zéro à l'issue des tests :
 
- **Wipe Data** : Réinitialisation complète via le Device Manager d'Android Studio.
- **Validation** : Vérification visuelle du retour à l'écran de configuration d'usine.
- **Traçabilité** : Archivage du journal de bord et des captures d'écran.
Ce rapport a été rédigé dans un cadre pédagogique strict. Aucune manipulation n'a été effectuée sur un terminal de production.
