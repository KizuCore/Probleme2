# Projet – Problème 2  
## Déploiement d’une application web sécurisée sous Debian 13 (VirtualBox)

---

# Objectif

L’objectif de ce projet est de :

- Installer un serveur Debian 13 dans une machine virtuelle
- Déployer une application web (Flask)
- Mettre en place un reverse proxy (Caddy)
- Configurer une protection contre les attaques brute force (Fail2ban)
- Tester et valider le fonctionnement

---

# 1️⃣ Installation de la machine virtuelle

## ISO utilisée

Debian 13.3.0 amd64 – version Netinst  
https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-13.3.0-amd64-netinst.iso

> La version **Netinst** permet une installation légère et propre, en téléchargeant uniquement les paquets nécessaires.

---

## Création de la VM (VirtualBox)

[Lien téléchargement VirtualBox](https://www.virtualbox.org/wiki/Downloads)

### Étape 1 – Nouvelle machine

Dans VirtualBox :

- VM Name : Debian-Probleme2
- Image ISO : sélectionner le fichier netinst téléchargé

---

### Étape 2 – Configuration utilisateur

- Username : user
- Password : password
- Hostname : Debian-Probleme2
- Domain : laisser par défaut

---

### Étape 3 – Configuration matérielle

| Paramètre | Valeur |
|------------|--------|
| RAM | 2048 MB |
| CPU | 2 |
| Disque | 20 Go |
| Use EFI | Désactivé |

> Cette configuration permet un fonctionnement fluide tout en restant réaliste pour un serveur léger.

Cliquez ensuite sur **Finish**.

---

# 2️⃣ Configuration réseau (IMPORTANT)

Avant de lancer la machine virtuelle :

VirtualBox → Configuration → Réseau → Adapter 1

Changer :

NAT → **Accès par pont (Bridged Adapter)**

Puis cliquer sur **OK**.

> Ce mode permet :
> 
> - D’obtenir une adresse IP locale
> - D’accéder au serveur depuis la machine hôte
> - De tester Fail2ban dans des conditions réelles

---

# 3️⃣ Installation de Debian

Démarrer la VM.

---

# 4️⃣ Vérification de l’installation

Depuis la VM, ouvrez un terminal :

```bash
ip a | grep 192.168
```

Repérer l’adresse IP locale :

```
inet 192.168.x.x
```

Exemple : 192.168.50.89

> Conserver cette IP elle sera utilisée pour accéder à la VM et au site.

---

# 5️⃣ Configuration de sudo

Si `sudo` n’est pas installé :

```bash
su -
apt update
apt install sudo
```

Ajouter l’utilisateur au groupe sudo :

```bash
usermod -aG sudo user
```

> Cette étape permet d’exécuter des commandes administratives sans utiliser directement le compte root.

---

# 6️⃣ Connexion SSH à la machine virtuelle

## Vérifier si SSH est installé

```bash
systemctl status ssh
```

Si absent :

```bash
sudo apt update
sudo apt install openssh-server -y
sudo systemctl start ssh
sudo systemctl enable ssh
```

Puis : 

```bash
sudo systemctl status ssh
```

> Si la commande retourne active (running) le service est bien démarré. 

---

## Connexion depuis la machine hôte

Depuis votre terminal :

```bash
ssh user@IP_DE_LA_VM
```

Exemple :

```bash
ssh user@192.168.1.45
```

---

## Vérification

```bash
whoami
hostname
```

> Si le nom d’utilisateur et le nom de la VM s’affichent, la connexion fonctionne.

---

# 7️⃣ Installation des dépendances du projet

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install python3 python3-pip python3-venv caddy fail2ban curl -y
```

### Rôle des dépendances

> - **python3** : exécution de l’application web  
> - **python3-pip** : installation des bibliothèques Python  
> - **python3-venv** : création d’un environnement virtuel isolé  
> - **caddy** : reverse proxy  
> - **fail2ban** : protection contre les attaques répétées  
> - **curl** : tests HTTP et simulation d’attaques  

---

# 8️⃣ Création de l’environnement de travail

Nous créons un dossier dédié au projet afin d’organiser proprement les fichiers :

```bash
mkdir projet
cd projet
```

Ce dossier contiendra :

- Le fichier `app.py`
- Le fichier `requirements.txt`
- L’environnement virtuel Python

---

# 9️⃣ Création de l’application web minimale

Le sujet demande un site dynamique (et non un simple fichier HTML statique) avec des identifiants directement intégrés dans le code.

Nous utilisons Flask pour générer dynamiquement la page de connexion et traiter les requêtes POST.

Créer le fichier :

```bash
nano app.py
```

(ou `vim app.py` selon votre préférence)

## Contenu du fichier `app.py`

```python
from flask import Flask, request
import logging

app = Flask(__name__)

# Identifiants codés en dur (conformément au sujet)
VALID_USERNAME = "admin"
VALID_PASSWORD = "password123"

# Configuration des logs pour Fail2ban
logging.basicConfig(
    filename="app_login.log",
    level=logging.INFO,
    format="%(asctime)s - ECHEC LOGIN - ip=%(message)s"
)

@app.route("/")
def home():
    return """
    <h2>Connexion</h2>
    <form method="POST" action="/login">
        <input name="username" placeholder="Utilisateur"><br>
        <input name="password" type="password" placeholder="Mot de passe"><br>
        <input type="submit" value="Se connecter">
    </form>
    """

@app.route("/login", methods=["POST"])
def login():
    username = request.form.get("username")
    password = request.form.get("password")
    ip = request.headers.get("X-Forwarded-For", request.remote_addr)

    if username == VALID_USERNAME and password == VALID_PASSWORD:
        return "Connexion réussie"

    # Log en cas d'échec
    logging.info(ip)
    return "Identifiants incorrects"

if __name__ == "__main__":
    app.run()
```

### Explication

> - Les identifiants sont codés en dur conformément aux consignes.
> - Le contenu HTML est généré dynamiquement par Flask.
> - Chaque échec de connexion génère une entrée dans un fichier de log.
> - Ces logs seront analysés par Fail2ban pour détecter une activité suspecte.

---

# 🔟 Création du fichier requirements.txt

Créer le fichier :

```bash
nano requirements.txt
```

Contenu :

```
flask
gunicorn
```

Ce fichier permet d’installer automatiquement les dépendances Python nécessaires au projet.

---

# 1️⃣1️⃣ Création de l’environnement virtuel

```bash
python3 -m venv venv
source venv/bin/activate
```

> L’apparition de `(venv)` dans le terminal confirme que l’environnement est actif.
> 
> L’utilisation d’un environnement virtuel permet d’isoler les dépendances du projet du reste du système.

---

# 1️⃣2️⃣ Installation des dépendances Python

```bash
pip install -r requirements.txt
```

> Cette commande installe Flask et Gunicorn dans l’environnement isolé.

---


# 1️⃣3️⃣ Lancement de l’application avec Gunicorn

```bash
gunicorn -w 2 -b 127.0.0.1:5000 app:app
```

### Explication

- `-w 2` → lance 2 workers (processus)
- `-b 127.0.0.1:5000` → écoute en local sur le port 5000
- `app:app` → fichier `app.py` et variable `app`

> L’application est maintenant accessible localement.

Ouvrez une deuxième fenêtre SSH, reconnectez-vous, puis vous pouvez tester l'application avec cette commande : 

```bash
ssh user@IP_DE_LA_VM
```

```bash
curl http://127.0.0.1:5000
```

---

# 1️⃣4️⃣ Configuration du Reverse Proxy avec Caddy

Modifier le fichier :

```bash
sudo nano /etc/caddy/Caddyfile
```

Décommentez la ligne reverse_proxy et mettez ceci à la place (la ligne du reverse_proxy) :

Contenu :

```caddy
:80 {
    reverse_proxy 127.0.0.1:5000
}
```

Redémarrer Caddy :

```bash
sudo systemctl reload caddy
```

L’application est maintenant accessible via :

```
http://IP_DE_LA_VM
```


---

# 1️⃣5️⃣ Configuration du bannissement d’IP avec Fail2ban

## Objectif

Mettre en place un mécanisme de bannissement automatique lorsqu’une activité suspecte est détectée sur la ressource d’authentification.

Nous définissons comme activité suspecte :

> Plus de 5 tentatives de connexion échouées en moins de 10 minutes depuis la même adresse IP.

Chaque échec de connexion génère une entrée dans le fichier :

```
/home/user/projet/app_login.log
```

---

# Création du filtre personnalisé

Créer le fichier :

```bash
sudo nano /etc/fail2ban/filter.d/app-login.conf
```

Contenu :

```ini
[Definition]
failregex = ECHEC LOGIN - ip=<HOST>
ignoreregex =
```

> Ce filtre indique à Fail2ban de détecter les lignes contenant "ECHEC LOGIN" et d’extraire l’adresse IP.

---

# Configuration de la jail

Modifier (ou créer) le fichier :

```bash
sudo nano /etc/fail2ban/jail.local
```

Ajouter :

```ini
[app-login]
enabled = true
port = http,https
filter = app-login
logpath = /home/user/projet/app_login.log
maxretry = 5
findtime = 600
bantime = 600
```

### Explication des paramètres

> - **maxretry = 5** → bannissement après 5 échecs
> - **findtime = 600** → période d’observation de 10 minutes
> - **bantime = 600** → durée de bannissement de 10 minutes

---

# Redémarrage de Fail2ban

```bash
sudo systemctl restart fail2ban
```

Vérifier que la jail est active :

```bash
sudo fail2ban-client status
```

---

# Test du mécanisme

## Simulation d’une attaque brute force

Depuis la machine hôte :

```bash
for i in {1..10}; do \
curl -X POST -d "username=test&password=wrong" http://IP_DE_LA_VM/login; \
done
```

---

## Vérification du bannissement

Dans la VM :

```bash
sudo fail2ban-client status app-login
```

# Vérification au niveau du pare-feu

Pour vérifier que l’IP est réellement bloquée au niveau système :

```bash
sudo iptables -L
```

ou selon la configuration :

```bash
sudo nft list ruleset
```

On peut observer qu’une règle temporaire a été ajoutée pour bloquer l’adresse IP détectée comme suspecte.

> Si l’adresse IP apparaît dans la liste des IP bannies, cela confirme que le mécanisme fonctionne correctement.

---

# Résultat


Nous avons mis en place :

- Une application web dynamique minimale
- Gunicorn comme gestionnaire de processus
- Caddy comme reverse proxy
- Un mécanisme de bannissement d’IP avec Fail2ban
- Un test permettant de valider le fonctionnement

> Le système bloque automatiquement les adresses IP ayant un comportement suspect sur la page d’authentification, ce qui protège l’application contre les attaques par brute force.
L’ensemble répond aux exigences du problème 2.

---
# Architecture mise en place

```
Navigateur (machine hôte)
        ↓
Caddy (port 80 - VM)
        ↓
Gunicorn (127.0.0.1:5000 - VM)
        ↓
Application Flask
```

