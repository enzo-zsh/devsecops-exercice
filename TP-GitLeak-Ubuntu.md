Ozen
ozen_lz
Invisible

C'est le début du salon privé #devsecops. 
Ozen

 — 10:32
Le cours :

https://bastienmaurice.fr/slides/devsecops/from-devops-to-devsecops/00-introduction-au-devsecops/index.html 
Momotoculteur — 14:00
TP : 

Exercice
Objectifs :

Créer un dépôt et y commettre (pousser) intentionnellement deux types de secrets différents pour compromettre l'historique :
Un secret standard (ex: clé AWS)
Un secret au format custom (défini par une règle spécifique)
Mettre en place la configuration Gitleaks via un fichier .gitleaks.toml pour :
Ajouter une règle personnalisée ([[rules]]) qui détecte le secret custom
Ajouter une entrée à la liste blanche ([allowlist]) pour exclure un faux positif
Intégrer Gitleaks dans la chaîne de production :
Via un pipeline CI/CD (GitLab ou GitHub Actions) en mode détection rétrospective
Via un pre-commit hook Git local pour la prévention proactive (bloquer le secret avant le commit)
Valider que les secrets initiaux sont détectés par le CI/CD et que les tentatives de nouveaux commits secrets sont bloquées localement par le hook
Momotoculteur — 14:28
https://blog.stephane-robert.info/docs/securiser/analyser-code/gitleaks/#configuration
Accueil
Gitleaks : scanner vos dépôts Git pour détecter les secrets expo...
Gitleaks scanne l'historique Git, les fichiers et les archives pour détecter les secrets. Rapide, configurable, idéal pour les pre-commit hooks et CI/CD. Guide complet avec exemples testés.
Gitleaks : scanner vos dépôts Git pour détecter les secrets exposés
Float representing the minimum shannon entropy a regex group must have to be considered a secret.
Momotoculteur — 14:53
Supprimer temporairement la protection : 
SKIP=gitleaks git commit -m "skip check"
Autoriser un secret dans le code directement : 
AWS_KEY = "AKIAIOSFODNN7EXAMPLE"  # gitleaks:allow
Thibault — 15:14
# TP Gitleaks — Guide Complet Pas à Pas
> VM Ubuntu vierge (Noble 24.04) — tout est installé depuis zéro

---

## 🗺️ Vue d'ensemble

TP_Gitleaks_Guide.md
13 Ko
﻿
# TP Gitleaks — Guide Complet Pas à Pas
> VM Ubuntu vierge (Noble 24.04) — tout est installé depuis zéro

---

## 🗺️ Vue d'ensemble

```
ÉTAPE 1 → Installer les outils (Git, Gitleaks, GitHub CLI)
ÉTAPE 2 → Créer le dépôt GitHub + commettre les secrets intentionnels
ÉTAPE 3 → Configurer .gitleaks.toml (règle custom + allowlist)
ÉTAPE 4 → Intégrer le CI/CD GitHub Actions (détection rétrospective)
ÉTAPE 5 → Mettre en place le pre-commit hook local (prévention)
ÉTAPE 6 → Valider tout fonctionne
```

---

## ÉTAPE 1 — Installation des outils sur la VM

### 1.1 — Mettre à jour le système
```bash
apt update && apt upgrade -y
```

### 1.2 — Installer Git
```bash
apt install git -y
git --version
# git version 2.43.x
```

### 1.3 — Installer Gitleaks
```bash
# Télécharger la dernière version de Gitleaks (binaire Linux amd64)
curl -LO https://github.com/gitleaks/gitleaks/releases/download/v8.21.2/gitleaks_8.21.2_linux_x64.tar.gz

# Extraire et installer
tar -xzf gitleaks_8.21.2_linux_x64.tar.gz
mv gitleaks /usr/local/bin/
chmod +x /usr/local/bin/gitleaks

# Vérifier
gitleaks version
# v8.21.2
```

### 1.4 — Installer GitHub CLI (gh) — pour créer le repo depuis la VM
```bash
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | \
  dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] \
  https://cli.github.com/packages stable main" | \
  tee /etc/apt/sources.list.d/github-cli.list > /dev/null

apt update && apt install gh -y
gh --version
```

### 1.5 — Configurer Git (identité locale)
```bash
git config --global user.name "TonNom"
git config --global user.email "ton@email.com"
```

### 1.6 — S'authentifier sur GitHub
```bash
gh auth login
# Choisir : GitHub.com → HTTPS → Login with a web browser
# Suivre les instructions (token affiché à coller dans le navigateur)
```

---

## ÉTAPE 2 — Créer le dépôt et commettre les secrets intentionnels

### 2.1 — Créer le dépôt local
```bash
mkdir ~/tp-gitleaks && cd ~/tp-gitleaks
git init
```

### 2.2 — Créer un fichier avec un secret AWS (secret standard)
```bash
cat > config.py << 'EOF'
# Configuration de l'application
DATABASE_URL = "postgresql://user:password@localhost/db"

# Clé AWS - NE PAS PARTAGER
AWS_ACCESS_KEY_ID = "AKIAIOSFODNN7EXAMPLE"
AWS_SECRET_ACCESS_KEY = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
AWS_REGION = "eu-west-1"
EOF
```

> ℹ️ `AKIAIOSFODNN7EXAMPLE` est une fausse clé AWS au format valide — Gitleaks la détectera comme un vrai secret.

### 2.3 — Créer un fichier avec un secret custom (format maison)
```bash
cat > app_secrets.env << 'EOF'
# Tokens internes de l'application
APP_NAME=MonApplication
APP_ENV=production

# Token interne custom — format : MON_TOKEN_xxxxxxxxxxxx (12 chars alphanumériques)
INTERNAL_API_TOKEN=MON_TOKEN_aZ3bY7cX9dW2
EOF
```

> ℹ️ Le format `MON_TOKEN_[12 chars alphanumériques]` est notre secret custom que nous allons détecter avec une règle Gitleaks personnalisée.

### 2.4 — Créer un faux positif (pour l'allowlist)
```bash
cat > tests/test_tokens.py << 'EOF'
# Tests unitaires - ces valeurs sont des EXEMPLES FICTIFS pour les tests
# Ne contiennent pas de vrais secrets

def test_token_format():
    """Vérifie que le format du token est correct"""
    fake_token = "MON_TOKEN_FAKEFAKEFAKE"  # Faux positif intentionnel
    assert fake_token.startswith("MON_TOKEN_")
    assert len(fake_token) == 22
    print("Format du token validé")
EOF
mkdir -p tests
cat > tests/test_tokens.py << 'EOF'
def test_token_format():
    fake_token = "MON_TOKEN_FAKEFAKEFAKE"
    assert fake_token.startswith("MON_TOKEN_")
    print("Format OK")
EOF
```

### 2.5 — Premier commit (avec les secrets — historique compromis)
```bash
git add .
git commit -m "feat: configuration initiale de l'application"
```

### 2.6 — Créer le repo sur GitHub et pousser
```bash
gh repo create tp-gitleaks --public --source=. --remote=origin --push
# Le repo est maintenant sur GitHub avec les secrets dans l'historique
```

---

## ÉTAPE 3 — Configurer Gitleaks (.gitleaks.toml)

### 3.1 — Créer le fichier de configuration
```bash
cat > .gitleaks.toml << 'EOF'
# =============================================================
# Configuration Gitleaks — TP Sécurité
# =============================================================

title = "Gitleaks Config — TP"

# -------------------------------------------------------------
# RÈGLE CUSTOM : Détection du token interne "MON_TOKEN_"
# -------------------------------------------------------------
[[rules]]
id          = "internal-api-token"
description = "Détecte les tokens internes au format MON_TOKEN_xxxxxxxxxxxx"
regex       = '''MON_TOKEN_[A-Za-z0-9]{12}'''
severity    = "HIGH"
tags        = ["token", "custom", "internal"]

  # Contexte : chercher autour du match pour confirmer que c'est un vrai secret
  [rules.allowlist]
  description = "Ignorer les fichiers de tests unitaires (faux positifs)"
  paths       = ['''tests/.*\.py$''']

# -------------------------------------------------------------
# ALLOWLIST GLOBALE : Exclusions générales
# -------------------------------------------------------------
[allowlist]
description = "Exclusions globales du projet"

# Ignorer les fichiers de documentation contenant des exemples
paths = [
  '''docs/.*''',
  '''README\.md'''
]

# Ignorer les commits qui contiennent explicitement la mention "FAKE" ou "EXAMPLE"
# (utile pour la doc et les tests)
regexes = [
  '''FAKEFAKEFAKE''',
  '''EXAMPLE KEY'''
]
EOF
```

### 3.2 — Tester Gitleaks localement (avant CI/CD)
```bash
# Scanner tout l'historique git
gitleaks detect --source . -v

# Tu dois voir :
# ✗ leaks found : 2 secrets détectés
# - aws-access-token  → config.py
# - internal-api-token → app_secrets.env
```

### 3.3 — Committer la configuration Gitleaks
```bash
git add .gitleaks.toml
git commit -m "chore: ajout configuration Gitleaks"
git push origin main
```

---

## ÉTAPE 4 — Pipeline CI/CD GitHub Actions (détection rétrospective)

### 4.1 — Créer le workflow GitHub Actions
```bash
mkdir -p .github/workflows
cat > .github/workflows/gitleaks.yml << 'EOF'
# =============================================================
# GitHub Actions — Détection de secrets avec Gitleaks
# Mode : Rétrospectif (scan de tout l'historique du repo)
# =============================================================

name: 🔐 Gitleaks - Détection de Secrets

on:
  push:
    branches: [ "**" ]
  pull_request:
    branches: [ main, develop ]
  # Permet de lancer manuellement depuis l'interface GitHub
  workflow_dispatch:

jobs:
  gitleaks:
    name: Scanner les secrets
    runs-on: ubuntu-latest

    steps:
      # 1. Récupérer TOUT l'historique git (fetch-depth: 0 = obligatoire)
      - name: Checkout du code (historique complet)
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # 2. Lancer Gitleaks en mode détection rétrospective
      - name: Gitleaks — Scan de l'historique complet
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          args: detect --source . --config .gitleaks.toml --redact -v --log-level debug

      # 3. Upload du rapport si des secrets sont trouvés (même en cas d'échec)
      - name: Upload rapport Gitleaks
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: gitleaks-report
          path: results.sarif
          retention-days: 7
EOF
```

### 4.2 — Pousser le workflow
```bash
git add .github/
git commit -m "ci: ajout pipeline GitHub Actions Gitleaks"
git push origin main
```

### 4.3 — Vérifier que le CI échoue (comportement attendu !)
```
1. Aller sur GitHub → ton repo → onglet "Actions"
2. Voir le workflow "Gitleaks - Détection de Secrets" s'exécuter
3. ✅ Il doit ÉCHOUER — c'est normal, il a trouvé les 2 secrets !
4. Cliquer sur le job pour voir le détail des secrets détectés
```

> ✅ **Résultat attendu** : Le CI/CD détecte les 2 secrets dans l'historique et marque le pipeline en rouge (FAILED).

---

## ÉTAPE 5 — Pre-commit Hook Git Local (prévention proactive)

Le hook va **bloquer tout commit** contenant un secret, AVANT qu'il soit enregistré.

### 5.1 — Créer le hook pre-commit
```bash
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/bash
# =============================================================
# Pre-commit Hook — Gitleaks
# Bloque les commits contenant des secrets détectés
# =============================================================

echo "🔐 [pre-commit] Scan Gitleaks en cours..."

# Scanner uniquement les fichiers stagés (pas tout l'historique)
gitleaks protect --staged --config .gitleaks.toml -v

EXIT_CODE=$?

if [ $EXIT_CODE -ne 0 ]; then
  echo ""
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "❌ COMMIT BLOQUÉ — Secret détecté par Gitleaks !"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "👉 Retire le secret du fichier avant de committer."
  echo "👉 Pour ignorer (DANGEREUX) : git commit --no-verify"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  exit 1
fi

echo "✅ [pre-commit] Aucun secret détecté. Commit autorisé."
exit 0
EOF

# Rendre le hook exécutable
chmod +x .git/hooks/pre-commit
```

### 5.2 — Tester que le hook bloque un nouveau secret
```bash
# Créer un fichier avec un nouveau secret
cat > nouveau_secret.txt << 'EOF'
TOKEN_PROD=MON_TOKEN_xK2mL9nP0qR5
EOF

git add nouveau_secret.txt
git commit -m "test: ajout token de prod"

# Résultat attendu :
# 🔐 [pre-commit] Scan Gitleaks en cours...
# ❌ COMMIT BLOQUÉ — Secret détecté par Gitleaks !
```

### 5.3 — Tester que le faux positif (allowlist) passe
```bash
# Le fichier tests/test_tokens.py contient MON_TOKEN_FAKEFAKEFAKE
# mais il est dans l'allowlist de la règle → doit passer

git add tests/test_tokens.py
git commit -m "test: ajout test unitaire format token"

# Résultat attendu :
# ✅ [pre-commit] Aucun secret détecté. Commit autorisé.
```

---

## ÉTAPE 6 — Validation complète

### Récapitulatif des validations

| Test | Commande | Résultat attendu |
|------|----------|-----------------|
| Scan historique local | `gitleaks detect --source . -v` | 2 secrets trouvés |
| CI/CD pipeline | Push sur GitHub | Pipeline ❌ FAILED |
| Hook bloque secret | `git commit` avec secret | Commit ❌ BLOQUÉ |
| Hook laisse passer | `git commit` fichier allowlist | Commit ✅ OK |

### Scan local final pour confirmer
```bash
# Rapport détaillé en JSON
gitleaks detect --source . --config .gitleaks.toml --report-format json --report-path rapport-secrets.json -v

cat rapport-secrets.json | python3 -m json.tool
```

### Structure finale du projet
```
tp-gitleaks/
├── .github/
│   └── workflows/
│       └── gitleaks.yml          ← Pipeline CI/CD
├── .git/
│   └── hooks/
│       └── pre-commit            ← Hook local
├── tests/
│   └── test_tokens.py            ← Faux positif (allowlist)
├── .gitleaks.toml                ← Config Gitleaks (règle + allowlist)
├── config.py                     ← Secret AWS (intentionnel)
└── app_secrets.env               ← Secret custom (intentionnel)
```

---

## 🧠 Résumé pédagogique

**Ce que tu as mis en place :**

1. **Détection rétrospective (CI/CD)** — Gitleaks scanne TOUT l'historique git à chaque push. Il détecte les secrets déjà commis dans le passé. Le pipeline échoue pour alerter l'équipe.

2. **Prévention proactive (pre-commit hook)** — Gitleaks intercepte les commits AVANT qu'ils soient enregistrés. Un secret ne peut pas entrer dans le repo si le hook est en place.

3. **Règle custom** — La regex `MON_TOKEN_[A-Za-z0-9]{12}` détecte les tokens internes spécifiques à ton organisation que Gitleaks ne connaît pas par défaut.

4. **Allowlist** — Les fichiers `tests/*.py` sont exclus de la règle custom car ils contiennent des valeurs fictives pour les tests (faux positifs).

> ⚠️ **Important** : Le pre-commit hook est LOCAL (dans `.git/hooks/`). Il ne se partage pas automatiquement via git. Pour le partager en équipe, utiliser un outil comme **pre-commit** (`pip install pre-commit`) avec un fichier `.pre-commit-config.yaml`.
TP_Gitleaks_Guide.md
13 Ko