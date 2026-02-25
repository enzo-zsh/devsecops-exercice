# exercice-devsecops-1
Exercice Gitleaks - DevSecOps

## Exercice : 

Objectifs :
- Créer un dépôt et y commettre (pousser) intentionnellement deux types de secrets différents pour compromettre l'historique :
    - Un secret standard (ex: clé AWS)
    - Un secret au format custom (défini par une règle spécifique)
- Mettre en place la configuration Gitleaks via un fichier .gitleaks.toml pour :
    - Ajouter une règle personnalisée ([[rules]]) qui détecte le secret custom
    - Ajouter une entrée à la liste blanche ([allowlist]) pour exclure un faux positif
- Intégrer Gitleaks dans la chaîne de production :
    - Via un pipeline CI/CD (GitLab ou GitHub Actions) en mode détection rétrospective
    - Via un pre-commit hook Git local pour la prévention proactive (bloquer le secret avant le commit)
- Valider que les secrets initiaux sont détectés par le CI/CD et que les tentatives de nouveaux commits secrets sont bloquées localement par le hook
