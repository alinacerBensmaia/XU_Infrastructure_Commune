# Documentation - Système de Validation Périodique de Sécurité
Date: 2025-11-30  
Contacts: Martin Desmeules, Fabiano Liberati

## 📋 Vue d'ensemble

Ce système fournit des templates Azure DevOps pour effectuer des scans de sécurité périodiques sur les repositories. Il combine plusieurs outils d'analyse :
- **GHAS (GitHub Advanced Security)** : CodeQL + Dependency Scanning
- **DevSkim** : Détection de patterns de sécurité

## 📁 Structure des fichiers

### Dans ST3.Outils.DevOps (à créer)
```
ST3.Outils.DevOps/
└── Securite/
    ├── ST3.Securite.Extends.ScanPeriodique.yml  # Template principal (Extends)
    ├── ST3.Securite.ScanGHAS.yml                # Template GHAS
    └── ST3.Securite.ScanDevSkim.yml             # Template DevSkim
```

### Dans les repositories à scanner
```
VotreRepository/
└── azure-pipelines-security-scan.yml  # Fichier de configuration
```

## 🚀 Utilisation

### 1. Configuration dans un repository

Créer un fichier `azure-pipelines-security-scan.yml` à la racine du repository :

```yaml
trigger:
  - main

schedules:
  - cron: "0 2 * * 0"  # Dimanche à 2h
    displayName: 'Scan hebdomadaire'
    branches:
      include:
        - main

resources:
  repositories:
    - repository: ST3.Outils.DevOps
      type: git
      name: ST3.Outils.DevOps
      ref: main

extends:
  template: Securite/ST3.Securite.Extends.ScanPeriodique.yml@ST3.Outils.DevOps
  parameters:
    solutionPath: '**/*.sln'
    enableGHAS: true
    enableDevSkim: true
    codeQLLanguages: 'csharp'
```

### 2. Paramètres configurables

| Paramètre | Type | Défaut | Description |
|-----------|------|--------|-------------|
| `solutionPath` | string | `**/*.sln` | Chemin vers les solutions à compiler |
| `buildConfiguration` | string | `Release` | Configuration de build |
| `buildPlatform` | string | `Any CPU` | Plateforme de build |
| `enableGHAS` | boolean | `true` | Activer GitHub Advanced Security |
| `enableDevSkim` | boolean | `true` | Activer DevSkim |
| `codeQLLanguages` | string | `csharp` | Langages pour CodeQL |
| `directoryExclusionList` | string | `''` | Dossiers à exclure du scan |

## 🔍 Détails des scans

### GHAS - GitHub Advanced Security

**CodeQL** : Analyse statique du code source
- Détecte : injections SQL, XSS, buffer overflows, etc.
- Supporte : C#, JavaScript, TypeScript, Python, Java, C/C++, Go

**Dependency Scanning** : Analyse des dépendances
- Scanne les packages NuGet/npm/pip
- Identifie les CVE connus
- Recommande des versions sécurisées

### DevSkim

**Analyse de patterns** : Détection de code non sécurisé
- Mots de passe en clair
- Algorithmes cryptographiques faibles
- Patterns de code dangereux
- Pratiques de sécurité obsolètes

## 📊 Résultats

Tous les scans alimentent le référentiel **Advanced Security** d'Azure DevOps :
- Tableau de bord centralisé
- Suivi des vulnérabilités
- Historique et tendances
- Alertes et notifications

### Accès aux résultats
1. Azure DevOps > Votre projet
2. Pipelines > Exécution du pipeline
3. Advanced Security > Alertes

## 🔧 Prérequis

### Dans Azure DevOps
- Extension **GitHub Advanced Security** installée
- Permissions appropriées sur le repository
- Agent avec accès au tool cache

### Pour CodeQL
- Binaire CodeQL pré-installé dans `C:\Temp\CodeQL\codeql-bundle-win64`
- OU installation automatique via le script PowerShell

### Pour DevSkim
- Installation automatique via dotnet tool
- Pas de configuration préalable nécessaire

## 📅 Recommandations de scheduling

### Scan hebdomadaire (recommandé)
```yaml
schedules:
  - cron: "0 2 * * 0"  # Dimanche 2h
```

### Scan quotidien (projets critiques)
```yaml
schedules:
  - cron: "0 3 * * *"  # Tous les jours 3h
```

### Scan mensuel (projets stables)
```yaml
schedules:
  - cron: "0 2 1 * *"  # 1er du mois 2h
```

## ⚠️ Notes importantes

1. **Durée d'exécution** : Les scans peuvent prendre 10-30 minutes selon la taille du projet
2. **CodeQL** : Nécessite une compilation complète du code
3. **Faux positifs** : Certains résultats peuvent nécessiter une revue manuelle
4. **Performance** : `WaitForProcessing: false` évite d'attendre le traitement complet

## 🐛 Dépannage

### CodeQL non trouvé
- Vérifier le chemin `C:\Temp\CodeQL\codeql-bundle-win64`
- Vérifier les permissions de l'agent
- Consulter les logs de l'étape d'installation

### DevSkim échoue
- Vérifier la connexion internet (téléchargement du package)
- Vérifier que .NET SDK est installé sur l'agent

### Résultats non visibles dans Advanced Security
- Vérifier que l'extension est activée
- Vérifier les permissions du pipeline
- Attendre quelques minutes (traitement asynchrone)

## 📞 Support

Pour toute question ou problème :
- **Martin Desmeules** : Architecture et GHAS
- **Fabiano Liberati** : DevSkim et intégration

## 🔄 Migration depuis l'ancien système

Si vous utilisez actuellement `XU_AnalysePeriodique.yml` :

1. Créer le nouveau fichier avec le template Extends
2. Migrer vos paramètres spécifiques
3. Tester sur une branche de test
4. Remplacer l'ancien pipeline une fois validé

## 📝 Exemple complet

Voir le fichier `EXEMPLE-azure-pipelines-security-scan.yml` pour un exemple complet et commenté.
