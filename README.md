# LAB 16 — Inspection HTTPS Android : Bypass SSL Pinning avec Objection + Burp Suite

## Objectifs

- Installer et configurer Objection et Frida sur PC
- Configurer un proxy Burp Suite et installer le certificat CA sur Android
- Désactiver le SSL Pinning d'une application Android avec Objection
- Capturer et analyser le trafic HTTPS déchiffré

---

## Environnement technique

| Composant | Version / Info |
|---|---|
| OS | Windows 11 |
| Python | 3.12 |
| Objection | 1.12.4 |
| Frida | 17.9.10 |
| ADB | Dernière version |
| Burp Suite Community | v2026.4.3 |
| Émulateur | Pixel 4a — Android 11 (API 30) |

---

## Étape 1 — Installation d'Objection et Frida sur PC

```bash
pip install --upgrade objection frida frida-tools
```

**Vérification :**

```bash
objection version   # → 1.12.4
frida --version     # → 17.9.10
```

---

## Étape 2 — Installation de frida-server sur l'émulateur

Architecture cible : `x86_64`

```bash
# Pousser le binaire sur l'émulateur
adb -s emulator-5556 push frida-server-17.9.10-android-x86_64 /data/local/tmp/frida-server

# Donner les droits d'exécution
adb -s emulator-5556 shell chmod 755 /data/local/tmp/frida-server

# Lancer frida-server (laisser ce terminal ouvert)
adb -s emulator-5556 shell "/data/local/tmp/frida-server -l 0.0.0.0"
```

**Vérification — liste des processus détectés :**

```bash
frida-ps -Uai
```

```
PID   Name      Identifier
----  --------  ---------------------------------------
8674  Chrome    com.android.chrome
9221  Diva      jakhar.aseem.diva
1204  Settings  com.android.settings
```

---

## Étape 3 — Configuration du proxy Burp Suite

### Côté Burp Suite

- Proxy listener : `0.0.0.0:8080`
- Intercept : **OFF**

### Côté émulateur

```bash
# Configurer le proxy
adb -s emulator-5556 shell settings put global http_proxy 10.0.2.2:8080


<img width="366" height="565" alt="Capture d&#39;écran 2026-05-20 140514" src="https://github.com/user-attachments/assets/538cf100-240e-4e94-ab26-0597308e20bd" />


# Vérifier la configuration
adb -s emulator-5556 shell settings get global http_proxy
# → 10.0.2.2:8080
```

### Installation du certificat CA Burp

1. Ouvrir `http://10.0.2.2:8080` dans le navigateur de l'émulateur
2. Cliquer sur **CA Certificate** pour télécharger
3. Installer via : **Paramètres → Sécurité → Installer un certificat → CA certificate**

<img width="340" height="326" alt="Capture d&#39;écran 2026-05-20 134748" src="https://github.com/user-attachments/assets/620b284f-30fc-4d72-aba9-6e85e4aa16b7" />

## Étape 5 — Capture du trafic HTTPS

**Test avec Gmail via Frida :**

```bash
# Récupérer le PID de Gmail
frida-ps -Uai | findstr gm

# Attacher Frida avec le script de bypass
frida -U -p 15339 -l sslpin_bypass_universal.js
```

**Sortie console Frida :**

```
[+] SSL bypass: SSLContext.init patched
[+] SSL bypass: X509TrustManager patches attempted
[+] SSL bypass: com.android.org.conscrypt.TrustManagerImpl patched
[+] Universal SSL pinning bypass installed
```

<img width="936" height="670" alt="Capture d&#39;écran 2026-05-20 173323" src="https://github.com/user-attachments/assets/3d98d6fb-7ea7-4f71-9771-df6a9c4ab446" />

**Trafic capturé dans Burp Suite — HTTP History :**


<img width="1919" height="792" alt="image" src="https://github.com/user-attachments/assets/f4717359-d675-4e48-90d2-6a9ead63506a" />


| Méthode | URL | Statut | TLS |
|---|---|---|---|
| POST | `https://inbox.google.com/sync/...` | 200 | ✓ |
| GET | `https://www.googleapis.com/gmail/v1/...` | 200 | ✓ |
| GET | `https://peoplestack-pa.googleapis.com/...` | 200 | ✓ |

---

## Difficultés rencontrées et solutions

| Problème | Solution |
|---|---|
| `PermissionDeniedError` avec Objection | Utiliser un émulateur rooté (ex : Genymotion) ou passer en mode `attach` |
| Plusieurs émulateurs actifs | Cibler avec `-s emulator-5556` dans les commandes ADB |
| Proxy ne capte pas le trafic | Utiliser `10.0.2.2:8080` (IP passerelle émulateur) plutôt que l'IP du réseau |
| `frida-server` non trouvé | Vérifier le chemin `/data/local/tmp/` et les droits d'exécution |
| Crash des applications Google | Comportement attendu — utiliser DIVA ou AndroGoat à la place |


---

## Commandes de référence

```bash
# Lister les appareils connectés
adb devices

# Lister les applications sur l'appareil
frida-ps -Uai

# Configurer le proxy
adb shell settings put global http_proxy 10.0.2.2:8080

# Désactiver le proxy
adb shell settings put global http_proxy :0

# Lancer Objection sur une application
objection -g nom.du.package explore

# Désactiver le SSL Pinning (dans la console Objection)
android sslpinning disable

# Quitter Objection
exit
```
