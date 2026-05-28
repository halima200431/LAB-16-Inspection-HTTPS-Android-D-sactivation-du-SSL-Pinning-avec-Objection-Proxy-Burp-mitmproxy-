# LAB 16 : Inspection HTTPS Android avec Objection et Burp Suite

## 1. Introduction

Ce laboratoire a pour objectif de réaliser une inspection du trafic HTTPS d’une application Android protégée par SSL Pinning.

L’analyse est réalisée dans un environnement contrôlé à l’aide de :

- Burp Suite comme proxy HTTP/HTTPS ;
- Frida pour l’instrumentation dynamique ;
- Objection comme surcouche Frida ;
- une application Android de test volontairement conçue pour le SSL Pinning.

Application cible utilisée :

```text
HTTP Toolkit SSL Pinning Demo
Package : tech.httptoolkit.pinning_demo
```

Le but principal est de montrer que certaines requêtes HTTPS bloquées par le SSL Pinning peuvent être interceptées après l’injection des hooks Objection.

---

## 2. Objectifs du lab

Les objectifs de ce lab sont :

- installer et utiliser Frida côté PC et côté Android ;
- installer et utiliser Objection ;
- configurer Burp Suite comme proxy HTTPS ;
- installer le certificat CA de Burp sur l’émulateur Android ;
- lancer l’application cible avec Objection ;
- exécuter la commande de désactivation du SSL Pinning ;
- valider que les requêtes HTTPS deviennent visibles dans Burp Suite ;
- documenter les problèmes rencontrés et leurs solutions.

Commande principale utilisée dans Objection :

```bash
android sslpinning disable
```

---

## 3. Environnement de travail

| Élément | Valeur |
|---|---|
| Système hôte | Windows |
| Terminal | PowerShell |
| Appareil Android | Émulateur Android |
| Version Android finale utilisée | Android 11 |
| Proxy | Burp Suite |
| Instrumentation dynamique | Frida |
| Surcouche Frida | Objection |
| Application cible | HTTP Toolkit SSL Pinning Demo |
| Package cible | `tech.httptoolkit.pinning_demo` |

---

## 4. Outils utilisés

Les outils utilisés dans ce lab sont :

- Android Debug Bridge ADB ;
- Frida ;
- Frida Server ;
- Objection ;
- Burp Suite ;
- Android Emulator ;
- Application de test SSL Pinning Demo.

---

## 5. Vérification des outils

Avant de commencer, les versions des outils ont été vérifiées avec les commandes suivantes :

```powershell
python --version
pip --version
frida --version
objection --version
```

Ces commandes permettent de vérifier que Python, pip, Frida et Objection sont correctement installés côté PC.


<img width="959" height="348" alt="image" src="https://github.com/user-attachments/assets/55988a84-4a8c-41bd-b395-b0654a059665" />

---

## 6. Vérification de l’appareil Android

L’émulateur Android a été vérifié avec ADB :

```powershell
$ADB = "$env:LOCALAPPDATA\Android\Sdk\platform-tools\adb.exe"
& $ADB devices
```

Résultat attendu :

```text
List of devices attached
emulator-5554    device
```


## 7. Installation et lancement de Frida Server

L’architecture de l’émulateur a été identifiée avec la commande suivante :

```powershell
& $ADB shell getprop ro.product.cpu.abi
```

Dans notre cas, l’architecture utilisée est :

```text
x86_64
```

Le binaire `frida-server` correspondant à la version Frida côté PC et à l’architecture Android a ensuite été copié vers l’émulateur :

```powershell
& $ADB push .\frida-server /data/local/tmp/frida-server
& $ADB shell chmod 755 /data/local/tmp/frida-server
```

Frida Server a ensuite été lancé :

```powershell
& $ADB shell "/data/local/tmp/frida-server -l 0.0.0.0:27042"
```

Dans un autre terminal PowerShell, la connexion avec Frida a été vérifiée :

```powershell
frida-ps -Uai
```

Exemple de résultat observé :

```text
PID   Name                    Identifier
----  ----------------------  ---------------------------------------
8455  Chrome                  com.android.chrome
...   ...
```


<img width="939" height="450" alt="image" src="https://github.com/user-attachments/assets/ca4ef4c5-9f3b-4de1-b4be-614ca91f4a66" />


---

## 8. Configuration de Burp Suite

Burp Suite a été lancé sur le PC avec un proxy configuré sur le port suivant :

```text
127.0.0.1:8080
```

Pour que l’émulateur Android puisse accéder au proxy Burp du PC, l’adresse suivante a été utilisée :

```text
10.0.2.2:8080
```

Le proxy a été configuré sur l’émulateur avec la commande suivante :

```powershell
& $ADB shell settings put global http_proxy 10.0.2.2:8080
```

La configuration a été vérifiée avec :

```powershell
& $ADB shell settings get global http_proxy
```

Résultat obtenu :

```text
10.0.2.2:8080
```

Capture recommandée :

```md
![Configuration du proxy Android](images/05-proxy-config.png)
```

---

## 9. Installation du certificat CA Burp

Le certificat CA de Burp Suite a été installé dans l’émulateur Android afin de permettre l’interception HTTPS.

L’installation du certificat a été réalisée depuis les paramètres Android :

```text
Settings
→ Security
→ Encryption & credentials
→ Install a certificate
→ CA certificate
```

Le certificat installé a été nommé :

```text
BurpCA
```

Capture recommandée :

```md
![Certificat Burp installé](images/04-burp-ca-installed.png)
```

Après l’installation du certificat, le navigateur de l’émulateur a été utilisé pour vérifier que le trafic HTTP/HTTPS pouvait passer par Burp.

---

## 10. Installation de l’application cible

L’application de test utilisée est :

```text
HTTP Toolkit SSL Pinning Demo
```

Elle a été installée avec la commande suivante :

```powershell
& $ADB install -r .\android-ssl-pinning-demo.apk
```

Le package de l’application a été identifié comme suit :

```text
tech.httptoolkit.pinning_demo
```

La présence de l’application a été confirmée avec Frida :

```powershell
frida-ps -Uai | Select-String -Pattern "pinning"
```

---

## 11. Test avant désactivation du SSL Pinning

Avant l’utilisation d’Objection, certaines requêtes de l’application échouaient à cause du SSL Pinning.

Dans l’application, les requêtes suivantes affichaient une erreur :

```text
VOLLEY PINNED REQUEST
TRUSTKIT PINNED REQUEST
```

Une erreur SSL était visible :

```text
javax.net.ssl.SSLHandshakeException
```

Cette erreur confirme que l’application refuse le certificat présenté par Burp Suite, même si le certificat CA Burp est installé sur Android.

<img width="390" height="783" alt="Capture d&#39;écran 2026-05-28 230157" src="https://github.com/user-attachments/assets/d08f7c94-660c-4673-8779-5988ed481ce6" />

Interprétation :

```text
L’application applique une vérification stricte du certificat serveur.
Le proxy Burp est détecté et la connexion HTTPS est refusée par certains modules de l’application.
```

---

## 12. Problème rencontré avec Android 17

Lors du premier essai, l’application a été testée sur un environnement Android récent affiché dans Objection comme :

```text
Android: 17
```

La commande suivante a été exécutée :

```bash
android sslpinning disable
```

Cependant, une erreur Frida/Objection est apparue :

```text
A Frida agent exception has occurred.
Error: Unable to find copied methods in java/lang/Thread; please file a bug
```


Analyse :

```text
Cette erreur est liée à une incompatibilité entre Objection/Frida et certaines versions Android récentes.
Pour stabiliser le lab, l’environnement a été remplacé par un émulateur Android 11.
```

Solution appliquée :

```text
Création d’un nouvel émulateur Android 11 / API 30.
Réinstallation de l’application cible.
Réinstallation du certificat CA Burp.
Relancement de frida-server.
Relancement d’Objection.
```

---

## 13. Désactivation du SSL Pinning avec Objection

Après passage à Android 11, Objection a été lancé sur l’application cible avec la commande suivante :

```powershell
objection -g tech.httptoolkit.pinning_demo explore
```

Dans la console Objection, la commande suivante a été exécutée :

```bash
android sslpinning disable
```

Résultat obtenu :

```text
(agent) Custom TrustManager ready, overriding SSLContext.init()
(agent) Found okhttp3.CertificatePinner, overriding CertificatePinner.check()
(agent) Found okhttp3.CertificatePinner, overriding CertificatePinner.check$okhttp()
(agent) Found com.android.org.conscrypt.TrustManagerImpl, overriding TrustManagerImpl.verifyChain()
(agent) Found com.android.org.conscrypt.TrustManagerImpl, overriding TrustManagerImpl.checkTrustedRecursive()
(agent) Registering job ... Name: android-sslpinning-disable
```
<img width="1046" height="477" alt="image" src="https://github.com/user-attachments/assets/744f99f3-38f6-4768-b3bb-76fdb471f1f5" />


Interprétation :

```text
Objection a correctement injecté des hooks dans l’application.
Les mécanismes de vérification TLS classiques ont été neutralisés.
Les classes OkHttp CertificatePinner et TrustManagerImpl ont été hookées.
```

---

## 14. Validation dans l’application Android

Après l’exécution de la commande `android sslpinning disable`, les requêtes précédemment bloquées ont été relancées depuis l’application.

Les boutons suivants sont passés en vert :

```text
CONFIG-PINNED REQUEST
CONTEXT-PINNED REQUEST
OKHTTP PINNED REQUEST
VOLLEY PINNED REQUEST
TRUSTKIT PINNED REQUEST
```

<img width="564" height="487" alt="Capture d&#39;écran 2026-05-28 232113" src="https://github.com/user-attachments/assets/4ebdc2f2-51d3-4b4c-8cbb-5c82f106ac07" />


Interprétation :

```text
Les requêtes qui échouaient avant l’injection Objection réussissent maintenant.
Cela confirme que le SSL Pinning a été désactivé pour les mécanismes principaux utilisés par l’application.
```

---

## 15. Validation dans Burp Suite

Après la désactivation du SSL Pinning, le trafic HTTPS de l’application est apparu dans Burp Suite dans :

```text
Proxy → HTTP history
```

Les requêtes HTTPS suivantes ont été observées :

```text
https://sha256.badssl.com
https://ecc384.badssl.com
https://amiusing.httptoolkit...
```

Les réponses HTTP étaient valides :

```text
Status : 200
Type   : HTML
Port   : 8080
```

<img width="1912" height="180" alt="Capture d&#39;écran 2026-05-28 232103" src="https://github.com/user-attachments/assets/cb3735b2-1185-45bb-aed5-8dfde0248201" />

Interprétation :

```text
Le trafic HTTPS de l’application passe maintenant par Burp Suite.
Les requêtes sont visibles dans l’historique HTTP.
Le proxy reçoit les réponses du serveur avec le code HTTP 200.
```

Cette étape valide la réussite du lab.

---

## 16. Résumé des résultats

| Test | Avant Objection | Après Objection |
|---|---|---|
| Proxy Burp configuré | Oui | Oui |
| Certificat CA installé | Oui | Oui |
| Requêtes non pinnées | Fonctionnelles | Fonctionnelles |
| Requêtes pinnées | Échec `SSLHandshakeException` | Succès |
| Hooks Objection | Non appliqués | Appliqués |
| Trafic visible dans Burp | Partiel | Oui |
| Code HTTP observé | Non applicable pour les requêtes bloquées | 200 |

---

## 17. Commandes principales utilisées

### Vérifier ADB

```powershell
$ADB = "$env:LOCALAPPDATA\Android\Sdk\platform-tools\adb.exe"
& $ADB devices
```

### Configurer le proxy Android

```powershell
& $ADB shell settings put global http_proxy 10.0.2.2:8080
& $ADB shell settings get global http_proxy
```

### Lancer Frida Server

```powershell
& $ADB shell "/data/local/tmp/frida-server -l 0.0.0.0:27042"
```

### Vérifier Frida

```powershell
frida-ps -Uai
```

### Lancer Objection

```powershell
objection -g tech.httptoolkit.pinning_demo explore
```

### Désactiver le SSL Pinning

```bash
android sslpinning disable
```

### Nettoyer le proxy après le lab

```powershell
& $ADB shell settings put global http_proxy :0
```

---

## 18. Problèmes rencontrés et solutions

### Problème 1 : pas d’accès Internet dans l’émulateur

Cause possible :

```text
Proxy mal configuré ou Burp non prêt.
```

Solution :

```powershell
& $ADB shell settings put global http_proxy :0
```

Puis test sans proxy. Ensuite, le proxy a été réactivé :

```powershell
& $ADB shell settings put global http_proxy 10.0.2.2:8080
```

---

### Problème 2 : le lien http://burp ne fonctionne pas

Explication :

```text
http://burp n’est pas un vrai site Internet.
Il fonctionne uniquement lorsque le navigateur passe déjà par le proxy Burp.
```

Solution :

```text
Exporter directement le certificat CA depuis Burp Suite.
Installer manuellement le certificat dans Android.
```

---

### Problème 3 : erreur Objection sur Android récent

Erreur observée :

```text
Unable to find copied methods in java/lang/Thread
```

Solution :

```text
Utilisation d’un émulateur Android 11/API 30.
Relancement du lab dans un environnement plus compatible avec Frida et Objection.
```

---

### Problème 4 : certains boutons restent violets

Certains boutons liés à des mécanismes avancés peuvent ne pas être neutralisés par Objection standard :

```text
APPMATTUS CT REQUEST
APPMATTUS+OKHTTP CT REQUEST
APPMATTUS+RAW TLS CT
APPMATTUS+WEBVIEW CT
FLUTTER REQUEST
```

Ces cas peuvent impliquer :

```text
Certificate Transparency
Raw TLS
Flutter
Pinning personnalisé
```

Cela ne bloque pas la validation du lab, car les mécanismes principaux de SSL Pinning ont bien été désactivés et le trafic HTTPS est visible dans Burp.

---


## 19. Conclusion

Ce lab a permis de mettre en place une chaîne complète d’analyse dynamique HTTPS sur Android.

Dans un premier temps, Burp Suite a été configuré comme proxy et son certificat CA a été installé sur l’émulateur. Ensuite, Frida Server a été lancé sur Android et Objection a été utilisé pour injecter des hooks dans l’application cible.

Avant le bypass, certaines requêtes échouaient avec une erreur `SSLHandshakeException`, ce qui confirmait la présence du SSL Pinning. Après l’exécution de la commande `android sslpinning disable`, Objection a hooké plusieurs mécanismes TLS, notamment `okhttp3.CertificatePinner` et `TrustManagerImpl`.

Les requêtes pinnées sont ensuite passées avec succès dans l’application, et le trafic HTTPS correspondant est devenu visible dans Burp Suite avec des réponses HTTP 200.

Le lab est donc validé.

---

## 20. Remarque éthique

Les techniques présentées dans ce lab doivent être utilisées uniquement dans un environnement autorisé, sur des applications de test ou dans le cadre d’un audit de sécurité encadré.

Elles ne doivent pas être appliquées à des applications tierces sans autorisation explicite.
