# LAB-9 — Analyse de Surface d'Attaque Android avec Drozer

**Auteur :** Mohamed Amine Znady  
**Filière :** Cycle Ingénieur — CIR (Systèmes d'Information Distribués), EMSI Marrakech  
**Module :** Sécurité Mobile & Audit Android  

> Lab réalisé dans un cadre d'audit défensif et autorisé. Aucune donnée réelle n'a été utilisée ou extraite durant ce lab.

---

## Objectif du lab

Explorer la **surface d'attaque** d'une application Android volontairement vulnérable (**AndroGoat**) via **Drozer**, un framework d'audit Android permettant d'interagir directement avec les composants exposés d'une application.

On s'intéresse à tout ce qu'une application offre involontairement à l'extérieur :

- Activités accessibles sans authentification
- Services déclenchables par d'autres apps
- Broadcast receivers écoutant des intents non filtrés
- Content providers exposant des données sans restriction

---

## Environnement

| Élément | Détail |
|---|---|
| **Système hôte** | Windows |
| **Environnement Android** | Android Studio Emulator |
| **Drozer Agent** | Installé et lancé dans l'émulateur |
| **Drozer Console** | Exécutée depuis Docker |
| **Application cible** | AndroGoat |
| **Package Android** | `owasp.sat.agoat` |

---

## Structure des preuves

```
preuves/
├── activities/
│   └── exported_activities.txt
├── services/
│   └── exported_services.txt
├── receivers/
│   └── exported_receivers.txt
└── providers/
    └── exported_providers.txt
```

---

## Étape 1 — Démarrage du Drozer Agent

Lancer l'application **Drozer Agent** dans l'émulateur et activer son serveur embarqué. Ce serveur permet à la console Drozer (côté PC) de communiquer avec l'émulateur.

---

## Étape 2 — Redirection de port ADB

Pour que la console Drozer (Docker) atteigne le Drozer Agent (émulateur), on met en place une redirection de port via ADB :

```bash
adb forward tcp:31416 tcp:31415
adb forward --list
```

> Le port `31416` côté hôte est mappé vers le port `31415` du Drozer Agent dans l'émulateur.

---

## Étape 3 — Connexion de la console Drozer

```bash
drozer console --server host.docker.internal:31417 connect
```

L'apparition du prompt `dz>` confirme que la connexion est établie.

---

## Étape 4 — Validation de la connexion

```bash
run information.deviceinfo
```

> L'émulateur peut retourner une erreur d'accès à `/proc/version` — comportement normal lié aux restrictions Android, cela ne bloque pas l'analyse.

---

## Étape 5 — Identification du package cible

```bash
run app.package.list -f goat
```

Package identifié : `owasp.sat.agoat`

---

## Étape 6 — Informations générales du package

```bash
run app.package.info -a owasp.sat.agoat
```

Récupération des métadonnées de l'application : version, permissions déclarées, chemins de données.

---

## Étape 7 — Vue d'ensemble de la surface d'attaque

```bash
run app.package.attacksurface owasp.sat.agoat
```

Une seule commande pour obtenir le nombre de composants exportés (activités, services, receivers, providers) et le statut `debuggable` de l'application.

---

## Étape 8 — Analyse des activités exportées

```bash
run app.activity.info -a owasp.sat.agoat
```

Une activité exportée peut être démarrée par n'importe quelle application installée. Si elle affiche un écran sensible ou ne vérifie pas l'état d'authentification, elle devient une porte d'entrée directe.

📄 Preuves : `preuves/activities/exported_activities.txt`

---

## Étape 9 — Analyse des services exportés

```bash
run app.service.info -a owasp.sat.agoat
```

Un service exporté sans contrôle de permissions peut être déclenché par une application tierce pour exécuter des opérations internes à l'insu de l'utilisateur.

📄 Preuves : `preuves/services/exported_services.txt`

---

## Étape 10 — Analyse des broadcast receivers

```bash
run app.broadcast.info -a owasp.sat.agoat
```

Un receiver exporté peut recevoir des intents envoyés par n'importe quelle application. Sans validation de la source ou de l'action reçue, il peut provoquer des comportements non prévus.

📄 Preuves : `preuves/receivers/exported_receivers.txt`

---

## Étape 11 — Analyse des content providers

```bash
run app.provider.info -a owasp.sat.agoat
```

Les content providers exposent des données via des URI. Un provider exporté sans permission de lecture/écriture peut permettre à d'autres apps de consulter ou modifier des données internes.

📄 Preuves : `preuves/providers/exported_providers.txt`

---

## Étape 12 — Analyse du manifest Android

```bash
run app.package.manifest owasp.sat.agoat
```

Le manifest est la carte d'identité d'une application Android : déclaration de tous les composants, permissions, intent-filters et attributs d'export. C'est souvent là que les mauvaises configurations apparaissent.

---

## Étape 13 — Découverte des URI de content providers

```bash
run scanner.provider.finduris -a owasp.sat.agoat
```

Recherche des URI de content providers référencées dans l'application pour identifier lesquelles sont accessibles sans authentification.

---

## Résumé de l'analyse

```
AndroGoat (owasp.sat.agoat)
   │
   ├─ [Étape 1-3]  Drozer Agent + ADB forwarding + console connectée
   ├─ [Étape 5-6]  Package identifié + métadonnées récupérées
   ├─ [Étape 7]    Surface d'attaque globale → composants exportés recensés
   ├─ [Étape 8]    Activités exportées → accès sans auth possible
   ├─ [Étape 9]    Services exportés → déclenchement par app tierce
   ├─ [Étape 10]   Broadcast receivers → intents non filtrés
   ├─ [Étape 11]   Content providers → données accessibles sans permission
   ├─ [Étape 12]   Manifest analysé → mauvaises configurations identifiées
   └─ [Étape 13]   URI providers découvertes → accès sans authentification ✓
```

---

*Mohamed Amine Znady — EMSI Marrakech, 2024-2025*
