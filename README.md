# LAB 16 — Analyse et Interception du Trafic HTTPS sur Android

## Contournement Dynamique du SSL Pinning à l'aide de Frida, Objection et Burp Suite

---

# Résumé

Les applications mobiles modernes utilisent fréquemment le mécanisme de SSL Pinning afin de renforcer la sécurité des communications réseau et d'empêcher les attaques de type Man-in-the-Middle (MITM). Bien que cette protection améliore la confidentialité des échanges, elle représente également un obstacle lors des audits de sécurité et des tests d'intrusion.

Ce laboratoire présente une méthodologie permettant d'analyser le trafic HTTPS d'une application Android protégée par SSL Pinning sans modifier son code source ni son fichier APK. L'approche repose sur l'utilisation conjointe de Frida, Objection et Burp Suite afin de neutraliser dynamiquement les mécanismes de validation des certificats et d'observer les communications réseau en temps réel.

L'expérimentation a été réalisée sur l'application pédagogique OWASP MSTG UnCrackable Level 1 exécutée dans un environnement Android émulé.

---

# Table des matières

1. Introduction
2. Objectifs du laboratoire
3. Environnement expérimental
4. Architecture de la solution
5. Méthodologie
6. Déploiement et configuration
7. Validation des résultats
8. Commandes utiles
9. Analyse des résultats
10. Conclusion
11. Références

---

# 1. Introduction

La majorité des applications Android s'appuie aujourd'hui sur le protocole HTTPS afin de garantir la confidentialité et l'intégrité des données échangées avec les serveurs distants.

Afin de renforcer davantage cette sécurité, certaines applications implémentent le SSL Pinning, une technique qui consiste à associer explicitement un certificat ou une clé publique à l'application. Cette vérification supplémentaire empêche l'utilisation de certificats intermédiaires non autorisés, même lorsqu'ils sont installés sur l'appareil.

Dans le cadre d'un audit de sécurité mobile, il est souvent nécessaire d'observer le trafic réseau généré par une application. Le SSL Pinning constitue alors un mécanisme de protection qu'il convient de contourner temporairement afin d'effectuer des analyses dynamiques.

---

# 2. Objectifs du laboratoire

Les objectifs poursuivis au cours de ce laboratoire sont les suivants :

* Comprendre le principe de fonctionnement du SSL Pinning.
* Mettre en place un environnement de test Android dédié à l'analyse réseau.
* Configurer un proxy d'interception HTTPS.
* Installer et utiliser Frida pour l'instrumentation dynamique.
* Exploiter Objection pour désactiver le SSL Pinning à l'exécution.
* Intercepter et analyser le trafic HTTPS généré par une application Android.
* Vérifier l'efficacité des hooks appliqués par Frida.

---

# 3. Environnement expérimental

## 3.1 Poste d'analyse

| Composant                  | Version           |
| -------------------------- | ----------------- |
| Système d'exploitation     | Windows 10 Pro    |
| Python                     | 3.13.5            |
| Pip                        | 25.3              |
| Android Debug Bridge (ADB) | 37.0.0            |
| Frida                      | 17.9.1            |
| Objection                  | 1.12.5            |
| Burp Suite                 | Community Edition |

## 3.2 Plateforme Android

| Élément         | Valeur                 |
| --------------- | ---------------------- |
| Type            | Android Virtual Device |
| Version Android | 8.1.0 (Oreo)           |
| Identifiant ADB | emulator-5554          |
| frida-server    | 17.9.1                 |

## 3.3 Application cible

| Paramètre | Valeur                                       |
| --------- | -------------------------------------------- |
| Nom       | OWASP MSTG UnCrackable Level 1               |
| Package   | owasp.mstg.uncrackable1                      |
| Catégorie | Application de démonstration sécurité mobile |

---

# 4. Architecture de la solution

L'environnement de test repose sur trois composants principaux :

* La machine d'analyse exécutant Burp Suite, Frida et Objection.
* L'émulateur Android contenant l'application cible.
* Le canal ADB permettant la communication et l'instrumentation dynamique.

## Schéma logique

```text
+------------------------------------------------------+
|                 Poste d'analyse                      |
|                                                      |
|  Burp Suite    Objection    Frida    ADB             |
+----------------------+-------------------------------+
                       |
                       |
                       v
+------------------------------------------------------+
|                Émulateur Android                     |
|                                                      |
|  OWASP UnCrackable Level 1                           |
|  Frida Server                                        |
+------------------------------------------------------+
```

## Flux réseau

```text
Application Android
        │
        ▼
Hooks Frida / Objection
        │
        ▼
Burp Suite (Proxy HTTPS)
        │
        ▼
Serveur distant
```

---

# 5. Méthodologie

L'approche adoptée repose sur l'instrumentation dynamique de l'application cible.

Contrairement à une modification statique de l'APK, cette méthode injecte du code directement dans le processus en mémoire grâce à Frida.

Les principales étapes sont :

1. Préparer l'environnement Android.
2. Déployer Frida Server.
3. Configurer le proxy Burp Suite.
4. Connecter Objection à l'application.
5. Désactiver dynamiquement le SSL Pinning.
6. Observer le trafic HTTPS intercepté.

---

# 6. Déploiement et configuration

## Étape 1 : Vérification des outils

```powershell
python --version
pip --version
adb version
frida --version
objection version
```

---

## Étape 2 : Vérification de l'émulateur Android

```powershell
adb devices
```

Résultat attendu :

```text
List of devices attached
emulator-5554    device
```

---

## Étape 3 : Installation et démarrage de Frida Server

Copie du serveur Frida vers l'émulateur :

```powershell
adb push frida-server /data/local/tmp/
adb shell "chmod 755 /data/local/tmp/frida-server"
```

Démarrage du service :

```powershell
adb shell "/data/local/tmp/frida-server &"
```

Vérification :

```powershell
adb shell "ps -A | grep frida"
```

---

## Étape 4 : Validation de la communication Frida

Lister les appareils détectés :

```powershell
frida-ls-devices
```

Lister les applications installées :

```powershell
frida-ps -Uai
```

Vérifier la présence du package :

```text
owasp.mstg.uncrackable1
```

---

## Étape 5 : Configuration de Burp Suite

Configurer un écouteur proxy :

```text
Adresse : 0.0.0.0
Port : 8080
```

Configurer ensuite le proxy sur l'émulateur :

```text
Host : 10.0.2.2
Port : 8080
```

Appliquer également la configuration via ADB :

```powershell
adb shell "settings put global http_proxy 10.0.2.2:8080"
```

Installer ensuite le certificat Burp sur l'émulateur afin d'autoriser l'inspection HTTPS.

---

## Étape 6 : Mise à jour d'Objection

```powershell
pip install objection --upgrade
```

Vérification :

```powershell
objection version
```

Version recommandée :

```text
1.12.5
```

---

## Étape 7 : Désactivation du SSL Pinning

Lancer l'application avec Objection :

```powershell
objection -n owasp.mstg.uncrackable1 start --startup-command "android sslpinning disable"
```

Cette commande installe automatiquement plusieurs hooks permettant de neutraliser les mécanismes de validation TLS.

---

## Étape 8 : Exploration de l'application

Une fois connecté :

```bash
android root disable
```

Lister les activités :

```bash
android hooking list activities
```

Lister les classes :

```bash
android hooking list classes
```

Lister les services :

```bash
android hooking list services
```

Afficher les hooks actifs :

```bash
jobs list
```

---

# 7. Validation des résultats

Les résultats obtenus démontrent que :

* Frida Server fonctionne correctement sur l'émulateur.
* Objection établit une connexion avec l'application cible.
* Les hooks SSL sont injectés avec succès.
* Le trafic HTTPS devient visible dans Burp Suite.
* Les requêtes réseau peuvent être observées et analysées.

Les indicateurs suivants confirment le succès de l'opération :

```text
Custom TrustManager ready
SSLContext.init() overridden
verifyChain() overridden
checkTrustedRecursive() overridden
```

La présence de ces messages indique que les mécanismes de vérification des certificats ont été neutralisés.

---

# 8. Commandes utiles

## Android Debug Bridge

```powershell
adb devices
adb shell
adb push
adb pull
adb install
adb uninstall
adb logcat
```

## Frida

```powershell
frida --version
frida-ls-devices
frida-ps -U
frida-ps -Uai
frida -U -f owasp.mstg.uncrackable1
```

## Objection

```powershell
android sslpinning disable
android root disable
android hooking list activities
android hooking list classes
android hooking list services
jobs list
```

---

# 9. Analyse des résultats

L'utilisation combinée de Frida et Objection permet de contourner efficacement les protections SSL Pinning sans altérer l'application.

Cette approche présente plusieurs avantages :

* Aucune modification de l'APK.
* Aucun processus de recompilation.
* Instrumentation en temps réel.
* Réversibilité immédiate.
* Compatibilité avec de nombreuses applications Android.

Toutefois, certaines applications implémentent des mécanismes de protection avancés pouvant nécessiter des scripts Frida personnalisés ou des techniques complémentaires.

---

# 10. Conclusion

Ce laboratoire a permis de démontrer la faisabilité de l'interception du trafic HTTPS d'une application Android protégée par SSL Pinning grâce à une instrumentation dynamique.

L'association de Frida, Objection et Burp Suite constitue une solution particulièrement efficace pour les audits de sécurité mobile et les analyses comportementales.

Les principaux résultats obtenus sont :

* Mise en place complète de l'environnement d'analyse.
* Déploiement réussi de Frida Server.
* Désactivation dynamique du SSL Pinning.
* Interception fonctionnelle du trafic HTTPS.
* Validation du fonctionnement des hooks de sécurité.

Cette méthodologie représente aujourd'hui l'une des approches les plus utilisées lors des évaluations de sécurité des applications Android.

---

# 11. Références

OWASP Mobile Application Security Testing Guide (MASTG)

https://mas.owasp.org/

Frida Documentation

https://frida.re/docs/

Objection Framework

https://github.com/sensepost/objection

PortSwigger Burp Suite Documentation

https://portswigger.net/burp/documentation

Android Developers Documentation

https://developer.android.com/

Android Debug Bridge (ADB)

https://developer.android.com/tools/adb
