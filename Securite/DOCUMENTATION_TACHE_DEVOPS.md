# Documentation - Tâche DevOps : Système de Validation Périodique de Sécurité

**Responsable** : Responsable des outils de soutien au développement  
**Contacts** : Martin Desmeules, Fabiano Liberati  
**Date de création** : 30 novembre 2025  
**Statut** : ✅ Complété

---

## 📌 Contexte et Objectif

### Besoin métier
Valider de façon **périodique** la sécurité des différents repositories de l'organisation pour :
- Détecter les vulnérabilités dans le code source
- Identifier les dépendances avec des CVE connus
- Automatiser la surveillance de la sécurité
- Centraliser les résultats dans Advanced Security

### Solution développée
Un système de templates Azure DevOps modulaires permettant d'exécuter des scans de sécurité automatisés et périodiques sur n'importe quel repository.

---

## 🏗️ Architecture de la solution

### Structure créée dans ST3.Outils.DevOps

```
ST3.Outils.DevOps/
└── Securite/
    ├── ST3.Securite.Extends.ScanPeriodique.yml    # Template principal (Extends)
    ├── ST3.Securite.ScanGHAS.yml                  # Template pour GHAS
    └── ST3.Securite.ScanDevSkim.yml               # Template pour DevSkim
```

### Principe de fonctionnement

```mermaid
┌─────────────────────────────────────────────────────────────┐
│                    Repository à scanner                      │
│         (ex: XU_Infrastructure_Commune)                      │
│                                                              │
│  azure-pipelines-security-scan.yml                          │
│  └── extends: ST3.Securite.Extends.ScanPeriodique.yml      │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│        ST3.Securite.Extends.ScanPeriodique.yml              │
│  ┌────────────────────────────────────────────────────┐     │
│  │ 1. Installation CodeQL                             │     │
│  │ 2. Initialisation CodeQL (PreBuild)                │     │
│  │ 3. Restauration NuGet                              │     │
│  │ 4. Compilation toutes les solutions                │     │
│  │ 5. Appel template GHAS                             │─────┼──┐
│  │ 6. Appel template DevSkim                          │─────┼──┼──┐
│  └────────────────────────────────────────────────────┘     │  │  │
└─────────────────────────────────────────────────────────────┘  │  │
                                                                  │  │
       ┌──────────────────────────────────────────────────────────┘  │
       │                                                              │
       ▼                                                              ▼
┌──────────────────────────┐                          ┌──────────────────────────┐
│ ST3.Securite.ScanGHAS    │                          │ ST3.Securite.ScanDevSkim │
├──────────────────────────┤                          ├──────────────────────────┤
│ • CodeQL Analyze         │                          │ • Installation DevSkim   │
│ • Dependency Scanning    │                          │ • Scan patterns sécurité │
│ • Publish GHAS           │                          │ • Génération SARIF       │
└────────────┬─────────────┘                          └────────────┬─────────────┘
             │                                                      │
             └──────────────────────┬───────────────────────────────┘
                                    ▼
                    ┌───────────────────────────────┐
                    │   Advanced Security - Azure   │
                    │   Référentiel centralisé      │
                    │   des vulnérabilités          │
                    └───────────────────────────────┘
```

---

## 📁 Détails des fichiers créés

### 1. ST3.Securite.Extends.ScanPeriodique.yml

**Type** : Template Extends (template principal orchestrateur)

**Responsabilités** :
- Orchestrer l'ensemble du processus de scan
- Compiler toutes les solutions du repository
- Gérer l'installation et l'initialisation de CodeQL
- Appeler les templates de scan (GHAS et DevSkim)

**Paramètres** :
| Paramètre | Type | Défaut | Description |
|-----------|------|--------|-------------|
| `solutionPath` | string | `**/*.sln` | Chemin vers les solutions à compiler |
| `buildConfiguration` | string | `Release` | Configuration de build |
| `buildPlatform` | string | `Any CPU` | Plateforme de compilation |
| `enableGHAS` | boolean | `true` | Activer/désactiver GHAS |
| `enableDevSkim` | boolean | `true` | Activer/désactiver DevSkim |
| `codeQLLanguages` | string | `csharp` | Langages pour CodeQL |
| `directoryExclusionList` | string | `''` | Dossiers à exclure du scan |

**Étapes d'exécution** :
1. Installation NuGet
2. Restauration des packages
3. Installation CodeQL (script PowerShell personnalisé)
4. Initialisation CodeQL (avant compilation)
5. Compilation VSBuild
6. Appel du template GHAS (si activé)
7. Appel du template DevSkim (si activé)

**Particularité** : Utilise un script PowerShell pour installer CodeQL depuis un bundle pré-téléchargé (`C:\Temp\CodeQL\codeql-bundle-win64`)

---

### 2. ST3.Securite.ScanGHAS.yml

**Type** : Template régulier (composant réutilisable)

**Responsabilités** :
- Exécuter l'analyse CodeQL du code source
- Scanner les dépendances pour les CVE
- Publier les résultats dans Advanced Security

**Paramètres** :
| Paramètre | Type | Défaut | Description |
|-----------|------|--------|-------------|
| `directoryExclusionList` | string | `''` | Répertoires à exclure du scan de dépendances |
| `waitForProcessing` | boolean | `false` | Attendre le traitement complet des résultats |

**Tâches Azure DevOps** :
1. `AdvancedSecurity-Codeql-Analyze@1` : Analyse le code avec CodeQL
2. `AdvancedSecurity-Dependency-Scanning@1` : Scanne les packages NuGet/npm/etc.
3. `AdvancedSecurity-Publish@1` : Publie tous les résultats

**Outils de sécurité** :
- **CodeQL** : SAST (Static Application Security Testing)
  - Injections SQL, XSS, path traversal, etc.
- **Dependency Scanning** : SCA (Software Composition Analysis)
  - CVE dans les packages NuGet, npm, pip, etc.

---

### 3. ST3.Securite.ScanDevSkim.yml

**Type** : Template régulier (composant réutilisable)

**Responsabilités** :
- Installer DevSkim CLI via dotnet tool
- Scanner le code pour détecter des patterns de sécurité dangereux
- Générer un rapport SARIF
- Publier les artefacts et les résultats

**Paramètres** :
| Paramètre | Type | Défaut | Description |
|-----------|------|--------|-------------|
| `sourcesDirectory` | string | `$(Build.SourcesDirectory)` | Répertoire à scanner |
| `continueOnError` | boolean | `false` | Continuer si le scan trouve des problèmes |
| `publishToAdvancedSecurity` | boolean | `true` | Publier vers Advanced Security |

**Étapes** :
1. **Installation** : `dotnet tool install --global Microsoft.CST.DevSkim.CLI`
2. **Analyse** : Scan avec sortie SARIF, niveaux Critical/Important/Moderate
3. **Publication artefacts** : Upload du fichier SARIF dans les artefacts du build
4. **Upload Advanced Security** : Intégration des résultats dans le référentiel

**Détections DevSkim** :
- Mots de passe en clair
- Algorithmes cryptographiques faibles (MD5, SHA1)
- Clés API hardcodées
- Pratiques de sécurité obsolètes
- Patterns de code dangereux

---

## 🚀 Guide d'utilisation

### Étape 1 : Déploiement dans ST3.Outils.DevOps

1. Créer le dossier `Securite/` à la racine de ST3.Outils.DevOps
2. Copier les 3 fichiers de template :
   - `ST3.Securite.Extends.ScanPeriodique.yml`
   - `ST3.Securite.ScanGHAS.yml`
   - `ST3.Securite.ScanDevSkim.yml`
3. Commit et push vers la branche appropriée (main ou testGHAS)

### Étape 2 : Configuration dans un repository

Créer un fichier `azure-pipelines-security-scan.yml` dans le repository à scanner :

```yaml
trigger:
  - main

# Scan périodique hebdomadaire
schedules:
  - cron: "0 2 * * 0"  # Dimanche à 2h du matin
    displayName: 'Scan de sécurité hebdomadaire'
    branches:
      include:
        - main
    always: true

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
    buildConfiguration: 'Release'
    enableGHAS: true
    enableDevSkim: true
    codeQLLanguages: 'csharp'
    directoryExclusionList: 'bin,obj,packages'
```

### Étape 3 : Configuration du pipeline dans Azure DevOps

1. Aller dans **Pipelines** > **New Pipeline**
2. Sélectionner le repository
3. Choisir **Existing Azure Pipelines YAML file**
4. Sélectionner `azure-pipelines-security-scan.yml`
5. **Save and Run**

---

## 📊 Résultats et monitoring

### Accès aux résultats

**Dans Azure DevOps** :
1. Projet > **Pipelines** > Sélectionner le pipeline de sécurité
2. Onglet **Advanced Security**
3. Visualiser les alertes par sévérité

**Informations disponibles** :
- Nombre total de vulnérabilités
- Répartition par sévérité (Critical, High, Medium, Low)
- Détails de chaque vulnérabilité (description, chemin, ligne de code)
- Recommandations de correction
- Historique et tendances

### Types de rapports

| Outil | Format | Localisation |
|-------|--------|--------------|
| CodeQL | Dashboard GHAS | Advanced Security > Code scanning |
| Dependency Scanning | Dashboard GHAS | Advanced Security > Dependencies |
| DevSkim | SARIF + Artefacts | Pipeline Artifacts > DevSkim-SecurityResults |

---

## ⚙️ Configuration avancée

### Scheduling personnalisé

**Scan quotidien** (projets critiques) :
```yaml
schedules:
  - cron: "0 3 * * *"  # Tous les jours à 3h
```

**Scan mensuel** (projets stables) :
```yaml
schedules:
  - cron: "0 2 1 * *"  # 1er du mois à 2h
```

**Multiple schedules** :
```yaml
schedules:
  - cron: "0 2 * * 0"   # Scan complet le dimanche
    displayName: 'Scan complet hebdomadaire'
  - cron: "0 4 * * 1-5" # Scan rapide en semaine
    displayName: 'Scan rapide quotidien'
```

### Désactiver certains scans

```yaml
parameters:
  enableGHAS: true      # Garder GHAS
  enableDevSkim: false  # Désactiver DevSkim temporairement
```

### Exclure des dossiers

```yaml
parameters:
  directoryExclusionList: 'bin,obj,packages,node_modules,TestResults'
```

### Langages multiples

```yaml
parameters:
  codeQLLanguages: 'csharp,javascript'  # Projets hybrides
```

---

## 🔧 Prérequis techniques

### Dans Azure DevOps

- ✅ Extension **GitHub Advanced Security for Azure DevOps** installée
- ✅ Permissions suffisantes pour créer des pipelines
- ✅ Licence Advanced Security activée pour le projet

### Sur les agents de build

- ✅ Windows avec PowerShell 5.1+
- ✅ Visual Studio Build Tools
- ✅ NuGet CLI
- ✅ .NET SDK (pour DevSkim)
- ✅ CodeQL bundle pré-installé dans `C:\Temp\CodeQL\codeql-bundle-win64` OU installation automatique via le script

### Permissions requises

- **Repository** : Lecture du code source
- **Pipelines** : Création et exécution
- **Advanced Security** : Écriture des résultats

---

## 🐛 Dépannage

### Problème : CodeQL non trouvé

**Symptôme** : `codeql.exe est introuvable`

**Solution** :
1. Vérifier que le bundle existe : `C:\Temp\CodeQL\codeql-bundle-win64\codeql\codeql.exe`
2. Vérifier les permissions de l'agent sur ce dossier
3. Alternative : Activer `enableAutomaticCodeQLInstall: true` dans la tâche Init

### Problème : DevSkim échoue à l'installation

**Symptôme** : `dotnet tool install` échoue

**Solution** :
1. Vérifier que .NET SDK est installé sur l'agent
2. Vérifier la connexion internet (téléchargement NuGet)
3. Vérifier les proxies/firewalls

### Problème : Résultats non visibles dans Advanced Security

**Symptôme** : Pipeline réussit mais aucune alerte

**Solution** :
1. Attendre 5-10 minutes (traitement asynchrone)
2. Vérifier que l'extension Advanced Security est activée
3. Vérifier les permissions du service de build
4. Consulter les logs de la tâche `AdvancedSecurity-Publish@1`

### Problème : Pipeline trop long

**Symptôme** : Le pipeline prend plus de 30 minutes

**Solution** :
1. Vérifier que `WaitForProcessing: false` (ne pas attendre le traitement)
2. Réduire le scope avec `directoryExclusionList`
3. Désactiver temporairement DevSkim si non critique
4. Compiler en mode Release (plus rapide que Debug)

### Problème : Trop de faux positifs

**Symptôme** : Beaucoup d'alertes non pertinentes

**Solution** :
1. **CodeQL** : Utiliser `querysuite: code-scanning` (moins de faux positifs)
2. **DevSkim** : Ajuster `severity-level` pour exclure les Low
3. Créer des suppressions dans Advanced Security pour les faux positifs confirmés
4. Documenter les raisons des suppressions

---

## 📈 Métriques et KPI

### Indicateurs de performance

- **Couverture** : % de repositories avec scan périodique activé
- **Fréquence** : Nombre de scans par semaine/mois
- **Détection** : Nombre de vulnérabilités détectées
- **Résolution** : Temps moyen de correction
- **Tendance** : Évolution du nombre de vulnérabilités dans le temps

### Rapports recommandés

1. **Dashboard mensuel** : Vulnérabilités critiques par projet
2. **Rapport trimestriel** : Amélioration de la posture de sécurité
3. **Alerte temps réel** : Nouvelles vulnérabilités critiques détectées

---

## 🔄 Maintenance et évolution

### Mises à jour des templates

**Processus** :
1. Modifier le template dans ST3.Outils.DevOps
2. Tester sur un repository pilote
3. Déployer vers la branche main
4. Les repositories utilisateurs récupèrent automatiquement la nouvelle version

**Fréquence recommandée** : Revue trimestrielle des templates

### Évolutions futures possibles

- ✨ Intégration de nouveaux scanners (Trivy, Semgrep)
- ✨ Scan des images Docker
- ✨ Scan des infrastructures (Terraform, ARM templates)
- ✨ Notifications Slack/Teams en cas de vulnérabilité critique
- ✨ Tableau de bord PowerBI centralisé
- ✨ Auto-création de work items pour les vulnérabilités

---

## 📞 Support et contacts

### Contacts principaux

- **Architecture sécurité et GHAS** : Martin Desmeules
- **DevSkim et intégration** : Fabiano Liberati
- **Support général** : Équipe Outils de soutien au développement

### Ressources additionnelles

- [Documentation GitHub Advanced Security](https://docs.github.com/en/code-security)
- [Documentation CodeQL](https://codeql.github.com/docs/)
- [Documentation DevSkim](https://github.com/microsoft/DevSkim)
- [Azure DevOps Advanced Security](https://learn.microsoft.com/en-us/azure/devops/repos/security/configure-github-advanced-security)

---

## ✅ Checklist de déploiement

### Phase 1 : Préparation
- [ ] Créer le dossier `Securite/` dans ST3.Outils.DevOps
- [ ] Copier les 3 templates
- [ ] Vérifier avec Fabiano pour l'exemple DevSkim
- [ ] Commit et push vers la branche appropriée

### Phase 2 : Test pilote
- [ ] Choisir 1-2 repositories pilotes
- [ ] Créer les fichiers de pipeline de sécurité
- [ ] Exécuter un premier scan manuel
- [ ] Valider que les résultats apparaissent dans Advanced Security
- [ ] Corriger les problèmes identifiés

### Phase 3 : Déploiement
- [ ] Documenter les bonnes pratiques (ce document)
- [ ] Communiquer auprès des équipes de développement
- [ ] Déployer progressivement sur tous les repositories
- [ ] Configurer les schedules appropriés

### Phase 4 : Monitoring
- [ ] Configurer les alertes pour les vulnérabilités critiques
- [ ] Créer un dashboard de suivi
- [ ] Planifier des revues mensuelles
- [ ] Former les équipes à l'interprétation des résultats

---

## 📋 Conclusion

Ce système de validation périodique de sécurité fournit une **approche standardisée et automatisée** pour maintenir un haut niveau de sécurité dans tous les repositories de l'organisation.

**Avantages clés** :
- ✅ Détection proactive des vulnérabilités
- ✅ Centralisation des résultats dans Advanced Security
- ✅ Standardisation des processus de scan
- ✅ Automatisation complète via scheduling
- ✅ Architecture modulaire et maintenable
- ✅ Couverture multiple (SAST + SCA + Pattern analysis)

**Prochaines étapes** : Déployer sur tous les repositories critiques et établir un processus de revue régulier des vulnérabilités détectées.

---

**Document maintenu par** : Équipe Outils de soutien au développement  
**Dernière mise à jour** : 30 novembre 2025  
**Version** : 1.0
