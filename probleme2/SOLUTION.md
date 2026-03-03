# Projet – Problème 2  
## Déploiement d’une application web sécurisée sous Debian 13 (VirtualBox)

---

# Objectif

L’objectif de ce projet est de :

- Déployer une application web dynamique (Flask)
- Mettre en place un reverse proxy (Caddy)
- Configurer une protection contre les attaques brute force (Fail2ban)
- Tester et valider le fonctionnement

## Pré-requis du sujet
Conformément à l’énoncé, on suppose une machine Debian neuve, un utilisateur `user` dans le groupe `sudo`, et `nftables` installé et actif.

## Périmètre du guide
Le cœur du guide traite la mise en place de l’application, de Caddy et de fail2ban.
Les étapes VirtualBox/installation Debian sont falcutatifs.


---

# 1 - Configuration de la machine virtuelle

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

# 2 - Connexion SSH à la machine virtuelle

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

Ouvrez un terminal :

```bash
ip a | grep 192.168
```

Repérer l’adresse IP locale :

```
inet 192.168.x.x
```

Exemple : 192.168.50.89

> Conserver cette IP elle sera utilisée pour accéder à la VM en SSH et au site.

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

# 3 - Installation des dépendances du projet

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install python3 python3-pip python3-venv caddy fail2ban curl -y
```
Ce guide contient des commandes effectuées avec l’éditeur de texte nano. Il est recommandé pour la suite du SOLUTION.md, mais vous pouvez utiliser l’éditeur de votre choix (par exemple vim). Dans ce cas, il faudra adapter les commandes et les raccourcis en fonction des spécificités de votre éditeur.

Pour installer nano : 

```bash
sudo apt install nano
```

### Rôle des dépendances

> - **python3** : exécution de l’application web  
> - **python3-pip** : installation des bibliothèques Python  
> - **python3-venv** : création d’un environnement virtuel isolé  
> - **caddy** : reverse proxy  
> - **fail2ban** : protection contre les attaques répétées  
> - **curl** : tests HTTP et simulation d’attaques  

---

# 4 - Création de l’environnement de travail

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

# 5 - Création de l’application web minimale

Nous utilisons Flask pour générer dynamiquement la page de connexion et traiter les requêtes POST.
Le code contient les identifiants codés en dur comme demandé sur le sujet.

username : `admin`
password : `password123`


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

> - Le contenu HTML est généré dynamiquement par Flask.
> - Chaque échec de connexion génère une entrée dans un fichier de log.
> - Ces logs seront analysés par Fail2ban pour détecter une activité suspecte.

---

# 6 - Création du fichier requirements.txt

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

# 7 - Création de l’environnement virtuel

```bash
python3 -m venv venv
source venv/bin/activate
```

>

> L’apparition de `(venv)` dans le terminal confirme que l’environnement est actif.
> 
> L’utilisation d’un environnement virtuel permet d’isoler les dépendances du projet du reste du système.

---

# 8 - Installation des dépendances Python

```bash
pip install -r requirements.txt
```

> Cette commande installe Flask et Gunicorn dans l’environnement isolé.

---


# 9 - Lancement de l’application avec Gunicorn

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
> Teste l’accès au serveur local (localhost) sur le port 5000 via une requête HTTP.

---

# 10 - Configuration du Reverse Proxy avec Caddy

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

> Ce bloc configure Caddy pour écouter sur le port 80 et rediriger les requêtes HTTP vers l’application exécutée localement sur le port 5000.

Appliquez la modification avec la commande :

```bash
sudo systemctl reload caddy
```

L’application est maintenant accessible depuis la machine hôte via :

```
http://IP_DE_LA_VM
```

> (Remplacer IP_DE_LA_VM par l’adresse IP de la machine virtuelle.)

---

# 11 - Configuration du bannissement d’IP avec Fail2ban

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

Après modification de la configuration, redémarrez le service afin d’appliquer les changements :

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
> Ici, nous faisons une boucle `for` envoyant 10 requêtes POST successives avec de faux identifiants pour simuler des tentatives de connexion répétées.
---
## Vérification du bannissement

Dans la VM :

```bash
sudo fail2ban-client status app-login
```
> Affiche le statut du jail app-login dans Fail2ban (service actif, IP bannies, etc.).

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

> Si l’adresse IP de votre machine hôte apparaît dans la liste des IP bannies, cela confirme que le mécanisme fonctionne correctement.

---

# Résultat


Nous avons mis en place :

- Une application web dynamique minimale
- Gunicorn comme gestionnaire de processus
- Caddy comme reverse proxy
- Un mécanisme de bannissement d’IP avec Fail2ban
- Un test permettant de valider le fonctionnement

> Le système bloque automatiquement les adresses IP ayant un comportement suspect sur la page d’authentification, ce qui protège l’application contre les attaques par brute force.

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

