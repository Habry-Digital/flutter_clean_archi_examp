# Architecture — monorepo talent / recruiter

Monorepo Melos avec 2 apps (`talent`, `recruiter`) partageant leur logique métier
et leur design system via des packages locaux, en clean architecture.

## Arborescence

```
mobile_app/                              # racine du monorepo (Melos)
├── melos.yaml
├── analysis_options.yaml
├── pubspec.yaml                         # workspace root (Melos)
│
├── apps/
│   ├── talent/
│   │   ├── lib/
│   │   │   ├── main.dart
│   │   │   ├── app.dart                 # MaterialApp(theme: appTheme), routing racine
│   │   │   ├── core/
│   │   │   │   ├── config/              # env_config.dart, app_keys.dart — spécifiques talent
│   │   │   │   └── di/
│   │   │   │       └── injection.dart   # composition root : domain+core+data + modules talent
│   │   │   ├── features/                # features propres au talent (non partagées)
│   │   │   │   └── <feature>/
│   │   │   │       ├── domain/          # usecases LOCAUX, propres à cette app seulement
│   │   │   │       └── presentation/
│   │   │   └── presentation/            # écrans non partagés
│   │   ├── assets/
│   │   └── pubspec.yaml                 # deps: domain, core, data, design_system, shared_screens
│   │
│   └── recruiter/
│       ├── lib/
│       │   ├── main.dart
│       │   ├── app.dart
│       │   ├── core/
│       │   │   ├── config/              # env_config.dart, app_keys.dart — spécifiques recruiter
│       │   │   └── di/
│       │   │       └── injection.dart
│       │   ├── features/                # features propres au recruiter (non partagées)
│       │   └── presentation/
│       ├── assets/
│       └── pubspec.yaml                 # deps: domain, core, data, design_system, shared_screens
│
└── packages/
    ├── domain/                          # DOMAIN — zéro dépendance externe (sauf ex. fpdart)
    │   ├── lib/src/
    │   │   ├── entities/                # candidate.dart, job_offer.dart, application.dart...
    │   │   ├── repositories/            # interfaces abstraites
    │   │   └── usecases/                # UNIQUEMENT les usecases communs aux 2 apps
    │   └── pubspec.yaml
    │
    ├── core/                            # infra technique transverse (pas de logique métier)
    │   ├── lib/src/
    │   │   ├── di/
    │   │   │   └── network_module.dart
    │   │   ├── error/                   # error_helper.dart, error_messages.dart, network_exceptions.dart
    │   │   ├── network/                 # api_kit.dart (Dio)
    │   │   ├── enums/                   # form_submission_status.dart, loading_status.dart
    │   │   ├── common/                  # type.dart
    │   │   ├── helpers/                 # utils purs
    │   │   ├── navigation/              # navigation_service.dart
    │   │   └── logger.dart
    │   └── pubspec.yaml
    │
    ├── data/                            # dépend de domain + core
    │   ├── lib/src/
    │   │   ├── datasources/
    │   │   │   ├── local/
    │   │   │   └── remote/
    │   │   ├── mappers/
    │   │   └── repositories_impl/       # implémente domain/repositories
    │   └── pubspec.yaml
    │
    ├── design_system/                   # UI partagée — même thème/boutons pour les 2 apps
    │   ├── lib/src/
    │   │   ├── components/
    │   │   │   ├── buttons/
    │   │   │   ├── inputs/
    │   │   │   ├── texts/
    │   │   │   ├── bottom_sheet/
    │   │   │   ├── bottom_safe_padding.dart
    │   │   │   ├── list_loading_effect.dart
    │   │   │   └── platform_bottom_spacing.dart
    │   │   ├── loader/
    │   │   │   └── simple_circular_loader.dart
    │   │   ├── templates/
    │   │   │   └── reusable_paged_list_view.dart
    │   │   └── theme/
    │   │       ├── app_colors.dart      # constantes uniques (même marque pour les 2 apps)
    │   │       └── app_theme.dart       # un seul ThemeData
    │   └── pubspec.yaml
    │
    └── shared_screens/                  # PRESENTATION partagée — dépend de domain+data+design_system
        ├── lib/src/
        │   └── <feature_commune>/       # ex: auth, chat, notifications
        │       ├── presentation/        # screens + bloc/cubit
        │       └── di.dart
        └── pubspec.yaml
```

**Graphe de dépendances :** `domain` (rien) ← `core` (rien) ← `data` (domain + core) ←
`shared_screens` (domain + data + design_system) ← `apps/talent` / `apps/recruiter` (tout).

## Règles établies

### 1. `domain` vs `core`
- **`domain`** = règles métier pures (entities, usecases communs, interfaces de repositories).
  Aucune dépendance à Flutter, Dio ou get_it — le domaine ne doit rien savoir des couches
  externes (dependency rule de la clean architecture).
- **`core`** = plomberie technique transverse (DI, logger, client réseau, gestion d'erreurs
  génériques, enums techniques). C'est de l'infra, pas du métier.

### 2. Usecases partagés vs locaux
- Un usecase va dans `packages/domain/usecases/` **uniquement** s'il est réellement utilisé
  par les 2 apps (ex: `GetCandidateProfileUseCase`, `RefreshTokenUseCase`).
- Un usecase propre à une seule app (ex: `PostJobOfferUseCase` côté recruiter,
  `ApplyToJobUseCase` côté talent) reste local, dans `apps/<app>/lib/features/.../domain/usecases/`.
- Ces usecases locaux peuvent réutiliser les **entities** et les **interfaces de repository**
  de `packages/domain` sans que la classe usecase elle-même y soit centralisée.
- Objectif : garder le partage honnête — seul ce qui est vraiment commun est mutualisé,
  le reste évite de gonfler le package partagé avec de la logique à usage unique.

### 3. Partage `domain` + `data` entre les 2 apps
- Légitime ici : talent et recruiter sont deux faces de la **même plateforme**, même backend,
  mêmes entités métier (candidature, offre, candidat).
- Risque à surveiller : couplage. Un changement dans `domain`/`data` doit être pensé pour
  les 2 apps à la fois. Garder les usecases granulaires limite l'impact des changements.
- À reconsidérer uniquement si un jour les 2 apps parlent à des backends différents ou que
  leurs modèles de données divergent fortement (auquel cas `data` se forkerait par app).

### 4. `design_system`
- Un seul thème (`AppColors`, `AppTheme`) car les 2 apps partagent exactement la même
  identité visuelle (mêmes couleurs, mêmes boutons) — pas besoin de `ThemeExtension`
  paramétrable par app, ce serait une abstraction inutile ici.
- Les assets de marque (logo, polices) qui différeraient malgré tout resteraient dans
  `apps/<app>/assets/`, pas dans `design_system`.

