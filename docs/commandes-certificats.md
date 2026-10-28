# 💻 Commandes : création de la CA et des certificats

Commandes utilisées sur le serveur **Red Hat Linux** pour mettre en place l'authentification par certificat avec **OpenSSL**.

> ⚠️ **Sécurité :** les valeurs entre `< >` et `CHANGE_ME` sont à remplacer. Ne publiez **jamais** les fichiers `.key`, `.p12` ni les vrais mots de passe.

## 📌 Sommaire

1. [Préparer le dossier de la CA](#1-préparer-le-dossier-de-la-ca)
2. [Créer l'autorité de certification (TekUp-CA)](#2-créer-lautorité-de-certification-tekup-ca)
3. [Créer le certificat du serveur](#3-créer-le-certificat-du-serveur)
4. [Créer un certificat client](#4-créer-un-certificat-client)
5. [Transférer le certificat vers Windows](#5-transférer-le-certificat-vers-windows)
6. [Créer un faux certificat (test)](#6-créer-un-faux-certificat-test)
7. [Commandes de vérification](#7-commandes-de-vérification)

---

## 1. Préparer le dossier de la CA

```bash
sudo mkdir -p /etc/nginx/ca /etc/nginx/ssl
cd /etc/nginx/ca
```

| Commande | Rôle |
|---|---|
| `mkdir -p` | Crée les dossiers (`ca` pour la CA et les clients, `ssl` pour le serveur) |
| `cd` | Entre dans le dossier de la CA |

---

## 2. Créer l'autorité de certification (TekUp-CA)

La CA est l'autorité qui **signe** tous les certificats.

```bash
# Clé privée de la CA
sudo openssl genrsa -out ca.key 4096

# Certificat de la CA (auto-signé, valable 10 ans)
sudo openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 \
  -out ca.crt -subj "/C=TN/ST=Tunis/O=TekUp/CN=TekUp-CA"

# Protéger la clé privée
sudo chmod 600 ca.key
```

| Fichier | Rôle |
|---|---|
| `ca.key` | Clé **privée** de la CA : à garder secrète 🔒 |
| `ca.crt` | Certificat **public** de la CA : utilisé par Nginx pour vérifier les clients |

---

## 3. Créer le certificat du serveur

Il prouve que le site `odoo.local` est le vrai serveur (HTTPS).

```bash
cd /etc/nginx/ssl

# Clé privée du serveur
sudo openssl genrsa -out odoo.local.key 2048

# Demande de certificat (CSR)
sudo openssl req -new -key odoo.local.key -out odoo.local.csr \
  -subj "/C=TN/ST=Tunis/O=TekUp/CN=odoo.local"

# Nom du site accepté par le navigateur (SAN)
echo "subjectAltName=DNS:odoo.local" | sudo tee odoo.local.ext

# Signature par la CA
sudo openssl x509 -req -in odoo.local.csr \
  -CA /etc/nginx/ca/ca.crt -CAkey /etc/nginx/ca/ca.key -CAcreateserial \
  -out odoo.local.crt -days 365 -sha256 -extfile odoo.local.ext
```

---

## 4. Créer un certificat client

Un certificat par utilisateur. Le **CN** (nom) doit être **identique au login Odoo** de la personne, car le module `cert_auth_check` compare les deux.

```bash
cd /etc/nginx/ca
NOM="nozha"        # à remplacer par le login Odoo de l'utilisateur

# 1. Clé privée de l'utilisateur
sudo openssl genrsa -out $NOM.key 2048

# 2. Demande de certificat (CN = login Odoo)
sudo openssl req -new -key $NOM.key -out $NOM.csr \
  -subj "/C=TN/ST=Tunis/O=TekUp/CN=$NOM"

# 3. Signature par la CA (valable 365 jours)
sudo openssl x509 -req -in $NOM.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out $NOM.crt -days 365 -sha256

# 4. Export en .p12 (certificat + clé, à importer dans le navigateur)
sudo openssl pkcs12 -export -out $NOM.p12 -inkey $NOM.key -in $NOM.crt \
  -name "$NOM" -passout pass:CHANGE_ME

# 5. Donner le fichier .p12 à l'utilisateur du serveur
sudo chown <utilisateur>:<utilisateur> $NOM.p12
```

| Étape | Fichier créé | Rôle |
|---|---|---|
| 1 | `.key` | Clé privée de l'utilisateur 🔒 |
| 2 | `.csr` | Demande de signature envoyée à la CA |
| 3 | `.crt` | Certificat **signé par TekUp-CA** |
| 4 | `.p12` | Paquet (certificat + clé) protégé par mot de passe, à importer dans le navigateur |

---

## 5. Transférer le certificat vers Windows

Depuis **PowerShell** ou **cmd** sur Windows :

```bash
cd Desktop
scp <utilisateur>@<IP_SERVEUR>:/etc/nginx/ca/nozha.p12 .
```

Puis sur Windows :

1. Double-cliquer sur le fichier `.p12`
2. Choisir **Utilisateur actuel** → **Suivant**
3. Saisir le **mot de passe** du `.p12` → **Suivant**
4. **Sélectionner automatiquement le magasin** → **Suivant** → **Terminer**
5. **Fermer complètement Chrome**, le rouvrir, puis aller sur `https://odoo.local`

---

## 6. Créer un faux certificat (test)

Pour tester le **scénario 3** : un certificat **non signé par TekUp-CA**.

```bash
cd /etc/nginx/ca

# Certificat auto-signé (la CA TekUp-CA n'intervient pas)
sudo openssl req -x509 -newkey rsa:2048 -nodes -keyout fake.key \
  -out fake.crt -days 365 -subj "/C=TN/O=Fake/CN=fake"

# Export en .p12
sudo openssl pkcs12 -export -out fake.p12 -inkey fake.key -in fake.crt \
  -name "fake" -passout pass:CHANGE_ME
```

---

## 7. Commandes de vérification

### 📋 Lister les certificats existants

```bash
ls -la /etc/nginx/ca/*.crt
```

### 🔎 Voir les informations d'un certificat (nom, signataire, dates)

```bash
openssl x509 -in /etc/nginx/ca/nozha.crt -text -noout | grep -E "Subject:|Issuer:|Not Before|Not After"
```

| Ligne affichée | Signification |
|---|---|
| `Subject` | À qui appartient le certificat (CN) |
| `Issuer` | Qui l'a signé (doit être `TekUp-CA`) |
| `Not Before` / `Not After` | Dates de début et de fin de validité |

### ✅ Vérifier qu'un certificat est signé par la CA

```bash
openssl verify -CAfile /etc/nginx/ca/ca.crt /etc/nginx/ca/nozha.crt
```

- Certificat valide : affiche **`OK`**
- Faux certificat (`fake.crt`) : affiche une **erreur** (non signé par TekUp-CA)

### 📅 Voir la date d'expiration

```bash
openssl x509 -in /etc/nginx/ca/nozha.crt -noout -enddate
```

### 🌐 Tester l'accès avec un certificat (curl)

```bash
sudo curl -k --cert /etc/nginx/ca/nozha.crt --key /etc/nginx/ca/nozha.key https://odoo.local
```

### ⚙️ Vérifier la configuration Nginx et relancer

```bash
sudo nginx -t
sudo systemctl restart nginx
```

### 🐳 Vérifier les conteneurs

```bash
sudo docker ps
cd ~/odoo-mfa && sudo docker compose ps
```

---

## 🔒 Rappel de sécurité

| À ne **jamais** publier | Pourquoi |
|---|---|
| `ca.key` | Permet de fabriquer de faux certificats valides |
| `*.key`, `*.p12` | Clés privées des utilisateurs |
| Mots de passe réels | Accès aux certificats |

Ajoutez dans `.gitignore` :

```
*.key
*.p12
*.csr
.env
```
