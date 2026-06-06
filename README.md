# LAB-16-Inspection-HTTPS-Android-D-sactivation-du-SSL-Pinning-avec-Objection-Proxy-Burp-mitmproxy-
We need to provide a version ready to copy-paste to GitHub. The user wants the reformulated README.md content, 100% in French (but can be bilingual? The original was French with some English. The instruction: "reformule ce readme 100% en l'ameliorant et change tout pour eviter le plagiat" and then "donne moi une version prete à copier coller sur github". So output the full README.md content in a code block, ready to copy. Ensure it's well-structured, improved, and original phrasing to avoid plagiarism. Also maintain markdown formatting.

Voici le contenu du fichier README.md entièrement reformulé, amélioré et prêt à être copié-collé sur GitHub :

markdown
# LAB 16 — Neutralisation du SSL Pinning sur Android  
## Avec Frida, Objection et un proxy intercepteur (Burp Suite / mitmproxy)

---

## 📑 Sommaire

1. [Objectifs pédagogiques](#objectifs)
2. [Environnement technique](#environnement)
3. [Architecture de la solution](#architecture)
4. [Procédure pas à pas](#procedure)
5. [Captures d’écran](#captures)
6. [Commandes utiles](#commandes)
7. [Résolution des problèmes](#depannage)
8. [Bilan et enseignements](#bilan)

---

## 🎯 Objectifs pédagogiques <a name="objectifs"></a>

- Comprendre le mécanisme de **certificate pinning** (ou SSL Pinning) sous Android et pourquoi il bloque l’interception du trafic HTTPS.
- Configurer un **proxy d’interception** (Burp Suite) sur un émulateur Android.
- Utiliser **Frida** et **Objection** pour contourner dynamiquement le SSL Pinning **sans recompiler ni modifier l’APK**.
- Visualiser en temps réel le flux HTTPS d’une application Android dans Burp Suite.
- Application d’étude : **OWASP MSTG UnCrackable Level 1** (`owasp.mstg.uncrackable1`).

---

## 🛠️ Environnement technique <a name="environnement"></a>

### Machine hôte (Windows)

| Composant | Version |
|-----------|---------|
| Système d’exploitation | Windows 10.0.19045 |
| Python | 3.13.5 / 3.11 (pip) |
| ADB (Android Debug Bridge) | 1.0.41 — build 37.0.0-14910828 |
| Frida (client) | 17.9.1 |
| Objection | 1.12.5 |
| Burp Suite | Community / Professional |

### Émulateur Android

| Propriété | Valeur |
|-----------|--------|
| Identifiant ADB | `emulator-5554` |
| Version Android | 8.1.0 (Oreo) |
| Type | AVD (Android Virtual Device) |
| frida-server | 17.9.1 (version identique au client) |

### Application cible

| Champ | Information |
|-------|-------------|
| Nom | OWASP MSTG UnCrackable Level 1 |
| Package | `owasp.mstg.uncrackable1` |
| Source officielle | [OWASP MASTG Crackmes](https://github.com/OWASP/owasp-mastg/tree/master/Crackmes/Android/Level_01) |

---

## 🏗️ Architecture de la solution <a name="architecture"></a>
┌─────────────────────────────────────────────────────────────┐
│ HÔTE WINDOWS │
│ │
│ ┌─────────────┐ ┌──────────────┐ ┌───────────────┐ │
│ │ Burp Suite │ │ Objection │ │ ADB │ │
│ │ :8080 │ │ v1.12.5 │ │ v1.0.41 │ │
│ └──────┬──────┘ └──────┬───────┘ └───────┬───────┘ │
│ │ │ │ │
└─────────┼──────────────────┼────────────────────┼───────────┘
│ Proxy HTTP/HTTPS │ Frida RPC │ USB/TCP
│ │ │
┌─────────┼──────────────────┼────────────────────┼───────────┐
│ │ ÉMULATEUR ANDROID (emulator-5554) │ │
│ │ │ │ │
│ ┌──────▼──────────────────▼──────────────┐ │ │
│ │ owasp.mstg.uncrackable1 │ │ │
│ │ (SSL Pinning neutralisé dynamiquement)│ │ │
│ └─────────────────────────────────────────┘ │ │
│ │ │
│ ┌───────────────────────────────────────────┐ │ │
│ │ frida-server 17.9.1 │◄─┘ │
│ │ /data/local/tmp/frida-server │ │
│ └───────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘

text

**Cheminement des paquets interceptés :**  
App Android ──HTTPS──► frida-server (bypass SSL) ──HTTP──► Burp Suite :8080 ──► Internet

text

---

## 📝 Procédure pas à pas <a name="procedure"></a>

### 1. Vérification des prérequis

```powershell
python --version        # Python 3.13.5
pip --version           # pip 25.3
adb version             # Android Debug Bridge 1.0.41
frida --version         # 17.9.1
objection version       # 1.12.5
2. Démarrage de l’émulateur et test ADB
powershell
adb devices
Résultat attendu :

text
List of devices attached
emulator-5554   device
3. Installation et lancement de frida-server sur l’émulateur
powershell
adb push frida-server /data/local/tmp/
adb shell "chmod 755 /data/local/tmp/frida-server"
adb shell "/data/local/tmp/frida-server &"
adb shell "ps -A | grep frida"
Sortie typique :

text
root  5846  5452  921280  89640  poll_schedule_timeout  ...  S  frida-server
4. Test de Frida : lister les applications
powershell
frida-ls-devices
frida-ps -Uai
Vérifier que owasp.mstg.uncrackable1 apparaît dans la liste.

5. Configuration du proxy Burp Suite
Burp Suite → Proxy → Proxy Settings → ajouter un listener sur 0.0.0.0:8080

Sur l’émulateur : Paramètres → Wi-Fi → Proxy manuel

Hôte : 10.0.2.2 (IP de la machine hôte vue par l’émulateur)

Port : 8080

Installer le certificat Burp :

Depuis le navigateur de l’émulateur, accéder à http://burp et télécharger le certificat

Ou via ADB :
adb shell "settings put global http_proxy 10.0.2.2:8080"

6. Mise à jour d’Objection (indispensable pour Frida 17.x)
⚠️ Avec Frida 17.x, Objection 1.12.5 ou supérieur est obligatoire.

powershell
pip install objection --upgrade
objection version   # Doit afficher 1.12.5 ou plus
7. Lancement d’Objection avec désactivation automatique du SSL Pinning
powershell
objection -n owasp.mstg.uncrackable1 start --startup-command "android sslpinning disable"
Retour attendu :

text
Running a startup command... android sslpinning disable
(agent) Custom TrustManager ready, overriding SSLContext.init()
(agent) Found com.android.org.conscrypt.TrustManagerImpl, overriding TrustManagerImpl.verifyChain()
(agent) Found com.android.org.conscrypt.TrustManagerImpl, overriding TrustManagerImpl.checkTrustedRecursive()
(agent) Registering job 32535. Name: android-sslpinning-disable

owasp.mstg.uncrackable1 (run) on (Android: 8.1.0) [usb] #
8. Commandes d’exploration dans l’invite Objection
Une fois dans l’invite owasp.mstg.uncrackable1 (run) on (Android: 8.1.0) [usb] # :

bash
# Neutraliser la détection de root
android root disable

# Lister les activités
android hooking list activities

# Lister toutes les classes de l’application
android hooking list classes

# Afficher les jobs actifs (le bypass SSL doit apparaître)
jobs list

# Lister les services
android hooking list services
📸 Captures d’écran <a name="captures"></a>
✅ Capture 1 – Versions de l’environnement
<img width="1156" height="385" alt="image" src="https://github.com/user-attachments/assets/46fe1f7c-4982-453f-a74f-865487a1660f" />
text
Python 3.13.5 | pip 25.3 | ADB 1.0.41 (Windows 10.0.19045)
✅ Capture 2 – Versions Frida & Objection
<img width="1125" height="490" alt="image" src="https://github.com/user-attachments/assets/f0daa91e-af91-4207-a33c-14fd265d6392" />
text
Frida : 17.9.1 → Objection : 1.12.5 (après mise à jour)
✅ Capture 3 – Périphériques ADB et processus Frida
text
emulator-5554   device (Android 8.1.0)
frida-ps -Uai → affiche correctement owasp.mstg.uncrackable1
<img width="972" height="813" alt="image" src="https://github.com/user-attachments/assets/2056f22c-8284-475c-8db6-9f58068af1be" />
✅ Capture 4 – Trafic intercepté dans Burp Suite
text
Historique HTTP → requêtes Android visibles sur le port 8080
Hôtes : connectivitycheck.gstatic.com, play.googleapis.com, www.google.com
Le proxy fonctionne et le trafic est bien capturé
<img width="1597" height="397" alt="image" src="https://github.com/user-attachments/assets/3273d186-3b0d-4c1d-885b-ab7d869ccf33" />
✅ Capture 5 – Objection connecté + SSL Pinning désactivé (au lancement)
text
objection -n owasp.mstg.uncrackable1 start --startup-command "android sslpinning disable"

(agent) Custom TrustManager ready, overriding SSLContext.init()         ✅
(agent) TrustManagerImpl.verifyChain() → overridden                     ✅
(agent) TrustManagerImpl.checkTrustedRecursive() → overridden           ✅
(agent) Job 32535 enregistré : android-sslpinning-disable               ✅
Invite : owasp.mstg.uncrackable1 (run) on (Android: 8.1.0) [usb] #    ✅
<img width="1459" height="541" alt="image" src="https://github.com/user-attachments/assets/1f5d8e1d-feb2-4548-83c3-dfb6867f0f0a" />
✅ Capture 6 – Désactivation manuelle du SSL Pinning depuis l’invite
text
owasp.mstg.uncrackable1 (run) on (Android: 8.1.0) [usb] # android sslpinning disable

(agent) Custom TrustManager ready, overriding SSLContext.init()         ✅
(agent) TrustManagerImpl.verifyChain() → overridden                     ✅
(agent) TrustManagerImpl.checkTrustedRecursive() → overridden           ✅
(agent) Job 800451 enregistré : android-sslpinning-disable              ✅
<img width="1414" height="631" alt="image" src="https://github.com/user-attachments/assets/ea74e303-696d-454f-a34d-b4777b18868b" />
📚 Commandes utiles <a name="commandes"></a>
ADB
powershell
adb devices                                          # Lister les périphériques
adb shell "ps -A | grep frida"                       # Vérifier frida-server
adb shell "/data/local/tmp/frida-server &"           # Lancer frida-server
adb shell pm list packages | findstr uncrackable     # Vérifier l’installation de l’app
adb shell monkey -p owasp.mstg.uncrackable1 1        # Démarrer l’application
adb shell "settings put global http_proxy 10.0.2.2:8080"  # Configurer le proxy système
Frida
powershell
frida-ls-devices                        # Lister les périphériques Frida
frida-ps -U                             # Processus actifs
frida-ps -Uai                           # Toutes les applications + identifiants
frida -U -f owasp.mstg.uncrackable1     # Spawner l’app avec Frida
frida --version                         # Afficher la version
Objection
powershell
# Lancement avec bypass SSL automatique
objection -n owasp.mstg.uncrackable1 start --startup-command "android sslpinning disable"

# Commandes internes à l’invite Objection :
android sslpinning disable              # Désactiver le pinning SSL
android root disable                    # Contourner la détection root
android hooking list activities         # Lister les activités
android hooking list classes            # Lister les classes
android hooking list services           # Lister les services
jobs list                               # Voir les hooks actifs
🔧 Résolution des problèmes <a name="depannage"></a>
Message d’erreur	Cause probable	Solution
Unable to find target application	frida-server non démarré ou app non lancée	Exécuter adb shell "/data/local/tmp/frida-server &" puis ouvrir l’application
DeprecationWarning: 'gadget' is deprecated	Ancienne syntaxe -g	Remplacer par -n
DeprecationWarning: 'explore' is deprecated	Commande obsolète	Remplacer par start
Incompatibilité Frida/Objection	Versions incohérentes	Mettre à jour Objection : pip install objection --upgrade
Architecture incorrecte de frida-server	Binaire Frida inadapté au CPU de l’émulateur	Vérifier avec adb shell getprop ro.product.cpu.abi puis télécharger le bon binaire
🏁 Bilan et enseignements <a name="bilan"></a>
Ce laboratoire démontre une approche non intrusive pour neutraliser le SSL Pinning sous Android.

Ce qui a été accompli
Aucune modification de l’APK : tout le contournement s’effectue à l’exécution par injection via Frida.

SSL Pinning désactivé sur trois niveaux : SSLContext, TrustManagerImpl.verifyChain() et TrustManagerImpl.checkTrustedRecursive().

Interception HTTPS fonctionnelle dans Burp Suite sur le port 8080.

Environnement opérationnel complet : Frida 17.9.1 + Objection 1.12.5 + Android 8.1.0.

Points clés à retenir
La version de frida-server doit être rigoureusement identique à celle du client frida sur le poste de travail.

Objection 1.12.5 est nécessaire pour une compatibilité parfaite avec Frida 17.x.

L’application doit être spawnée (pas simplement installée) pour que l’injection réussisse.

L’option --startup-command permet d’activer le hook SSL dès la première requête réseau.

Références
OWASP MASTG — Testing Network Communication

Objection sur GitHub

Documentation Frida

PortSwigger — Tests mobiles avec Burp Suite
