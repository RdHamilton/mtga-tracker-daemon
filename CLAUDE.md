# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Vue d'ensemble du projet

Un démon HTTP en C# qui extrait des données du processus MTG Arena en cours d'exécution via l'inspection de la mémoire Unity. Fournit une API REST pour accéder aux informations de jeu (cartes, inventaire, état des matchs, etc.).

## Architecture technique

### Structure en deux projets

1. **HackF5.UnitySpy** : Bibliothèque de bas niveau pour l'inspection de mémoire Unity
   - Support multi-plateforme (Windows, Linux, macOS)
   - Lecture de la mémoire des processus Unity via différentes façades (`ProcessFacade`)
   - Accès aux structures de données Mono/Unity (classes, champs, types)
   - Gestion des offsets spécifiques aux différentes versions Unity

2. **mtga-tracker-daemon** : Serveur HTTP principal
   - Démarre un `HttpListener` sur un port configurable (défaut : 6842)
   - Utilise UnitySpy pour lire les données du processus MTGA
   - Sérialise les réponses en JSON avec Newtonsoft.Json
   - Système de mise à jour automatique (Linux uniquement)

### Points clés de l'architecture

- **Injection de dépendances Unity** : Les données sont récupérées via des chemins d'accès aux objets Unity (ex: `WrapperController["<Instance>k__BackingField"]["<InventoryManager>k__BackingField"]`)
- **Façades spécifiques à la plateforme** :
  - Windows : `ProcessFacadeWindows` via Win32 API
  - Linux : `ProcessFacadeLinuxDirect` via `/proc/{pid}/mem`
  - macOS : `ProcessFacadeMacOSDirect`
- **Base de données SQLite** : Lecture de la base de données des cartes MTGA pour `/allCards`

## Commandes de développement

### Build
```bash
# Build standard
dotnet build

# Build Release avec self-contained runtime
dotnet publish ./src/mtga-tracker-daemon -r win-x64 --self-contained    # Windows
dotnet publish ./src/mtga-tracker-daemon -r linux-x64 --self-contained  # Linux
```

### Exécution
```bash
# Exécuter avec le port par défaut (6842)
dotnet run --project src/mtga-tracker-daemon

# Exécuter avec un port personnalisé
dotnet run --project src/mtga-tracker-daemon -- -p 9000
```

### Solution Visual Studio
```bash
# Ouvrir la solution
dotnet sln mtga-tracker-daemon.sln
```

## API Endpoints (HttpServer.cs)

- `GET /status` : État du processus MTGA, version du démon, mise à jour en cours
- `GET /cards` : Collection de cartes du joueur (grpId + quantité possédée)
- `GET /allCards` : Toutes les cartes disponibles via la DB SQLite de MTGA
- `GET /playerId` : ID du compte Wizards, nom d'affichage, PersonaID
- `GET /inventory` : Gemmes et or du joueur
- `GET /events` : Événements actifs
- `GET /matchState` : État du match en cours (ID, rangs des joueurs)
- `POST /checkForUpdates` : Vérifie les mises à jour disponibles
- `POST /shutdown` : Arrête le démon

## Détails techniques importants

### Détection du processus MTGA
- Windows : cherche `MTGA` dans la liste des processus
- Linux : cherche `MTGA.exe` ET vérifie que `/proc/{pid}/maps` n'est pas vide

### Système de mise à jour automatique (Linux)
- Vérifie les releases GitHub au démarrage
- Télécharge et extrait automatiquement les nouvelles versions
- Redémarre via `systemctl restart mtga-trackerd.service`
- Utilise `/tmp/mtga-tracker-dameon` comme répertoire temporaire

### Gestion des chemins de base de données
La connexion SQLite subit des transformations :
```csharp
connectionString = connectionString.Replace("Data Source=Z:", "Data Source=");
connectionString = connectionString.Replace("\\", "/");
```
Ceci adapte le chemin Windows de MTGA au système hôte.

### Échappement des noms de cartes
`StringUtils.JsonEscape()` est utilisé pour les titres de cartes car certains contiennent des caractères spéciaux JSON.

## Notes de développement

- Le projet cible **.NET 8.0** (voir `mtga-tracker-daemon.csproj`)
- HackF5.UnitySpy cible **.NET Standard 2.0** pour la compatibilité
- Les builds de release activent `TreatWarningsAsErrors`
- Le code utilise `unsafe` blocks dans UnitySpy (`AllowUnsafeBlocks=true`)
