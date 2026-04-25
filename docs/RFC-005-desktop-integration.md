# RFC-005 : Intégration UBIK-COLLAB dans UBIK-DESKTOP

**Statut :** Draft
**Auteur :** Claude Opus 4.7 (session 2026-04-25)
**Date :** 2026-04-25

---

## 1. Contexte

UBIK-COLLAB tel que défini dans RFC-001 à RFC-004 est conceptuellement complet mais non implémenté. Cette RFC propose une **stratégie d'intégration concrète** dans UBIK-DESKTOP comme premier client riche, avec un service sidecar autonome.

L'objectif : permettre à Damien de superviser plusieurs agents IA (ubik-genie, Claude Code, Gemini CLI, Codex CLI) collaborant sur le même thread, depuis l'interface DESKTOP qu'il utilise déjà quotidiennement.

## 2. Architecture proposée

```
┌─────────────────────────────────────────────────┐
│  UBIK-DESKTOP (Tauri)                           │
│  Sidebar: [Console] [Collab] [Memory] [Vault]   │
│             │                                    │
│             ▼  CollabMode.tsx (React)            │
│  ┌──────────┬──────────────┬───────────┐        │
│  │ Threads  │  Timeline    │  Status   │        │
│  │ list     │  (messages,  │  (locks,  │        │
│  │          │   diffs)     │   inbox)  │        │
│  └──────────┴──────────────┴───────────┘        │
└─────────────────────────────────────────────────┘
                  │ HTTP/WebSocket
                  ▼
┌─────────────────────────────────────────────────┐
│  ubik-collab-service (Rust, port 7892)          │
│  - Axum HTTP + WebSocket                         │
│  - SQLite (ledger + locks) via rusqlite         │
│  - Tree-sitter (AST locking, phase 3)            │
└─────────────────────────────────────────────────┘
                  ▲
       ┌──────────┴──────────┐
       │                     │
  ubik-genie agents    Claude Code / Gemini / Codex
  (via Tauri invoke   (via Python sidecar wrapper
   ou HTTP direct)     qui intercepte stdout)
```

### 2.1 Pourquoi sidecar séparé (pas embarqué dans api.rs) ?

- **Indépendance** : Claude Code, Codex CLI peuvent se brancher sans dépendre de DESKTOP étant lancé
- **Portabilité** : un agent CLI lancé sur dev-station-02 peut publier dans le même thread qu'un agent local
- **Single responsibility** : api.rs gère les PTYs, ubik-collab gère la collaboration. Pas de couplage.

### 2.2 Pourquoi Rust pour le service ?

- Cohérence avec api.rs (déjà Rust + Axum)
- Single binary, dist easy
- Perf pour locking + AST parsing
- Tree-sitter a des bindings Rust matures

## 3. Vue UI dans DESKTOP — 3 panels

### 3.1 Panel gauche : liste des Threads
- Tri par dernière activité
- Status : `open` / `locked` / `resolved`
- Participants visibles avec icône (humain = avatar, agent = Bot icon ambre)
- Recherche par titre, par participant

### 3.2 Panel centre : Timeline du thread
Messages structurés rendus visuellement :

| Type | Rendu |
|------|-------|
| `INTENT`         | Bulle bleue, icône 🎯, auteur + intention |
| `TASK`           | Bulle violette, icône 📋, mention `@AgentName` |
| `DECISION`       | Bulle ambre, icône ✓, justification |
| `ACTION_REQUEST` | Bulle rouge, icône ⚠️, bouton "Approve" pour l'humain |
| Inline diff      | Bloc code avec syntax highlighting + lien vers commit |

Compose box en bas pour humain : peut écrire texte libre ou DECISION explicite.

### 3.3 Panel droit : Status du thread
- **Locks actifs** : `src/auth.py::AuthService::verify_token` (acquis par Agent X depuis 12 min)
- **Ledger récent** : 5 dernières entrées avec sha + auteur + résumé
- **Approval inbox** : actions en attente d'humain (sticky en haut)

## 4. MVP en 3 phases

### Phase 1 — Decision Ledger (objectif : 1 semaine)
**Scope minimal pour valider la valeur**
- Backend Rust : crate `ubik-collab` dans nouveau workspace
  - `POST /threads` créer un thread
  - `GET /threads` lister
  - `POST /threads/:id/entries` ajouter une entrée
  - `GET /threads/:id/entries?since=...` lire timeline
  - SQLite : tables `threads`, `entries`, `participants`
- Hook git post-commit (Phase 1.5) : capte chaque commit comme `entry` de type `commit`
- Agents écrivent INTENT/DECISION via syntaxe `@@INTENT: ...` dans leurs réponses, parsé côté DESKTOP
- UI : `CollabMode.tsx` + 3 panels en lecture seule (pas d'édition humain pour Phase 1)
- **Skip** : WebSocket, locking, approval, consensus

**Livrable** : tu peux voir l'historique des décisions/intentions de tous les agents dans une vue unifiée, par projet.

### Phase 2 — Bus temps réel + Approval (1-2 semaines)
- WebSocket sur `/ws/threads/:id` pour push live
- Approval inbox : humain peut cliquer "Approve" ou "Reject" sur un `ACTION_REQUEST`
- Compose box humain → publish DECISION
- Locking advisory : un agent peut `LOCK_ACQUIRE` mais c'est juste un warning (pas bloquant)

**Livrable** : tu réagis en temps réel aux propositions des agents, ils attendent ton feu vert pour les actions critiques.

### Phase 3 — Locking strict + Consensus (2-3 semaines)
- Intégration tree-sitter (Python, Rust, TypeScript pour commencer)
- `LockManager` valide hiérarchie parent/enfant (RFC-002)
- Matrice d'approbation appliquée (RFC-003) : auto / consensus(2+) / humain
- Protocole `PROPOSE` / `VOTE_YES` / `VOTE_NO` entre agents

**Livrable** : agents collaborent sans collisions sur le même fichier, consensus possible sans humain pour les actions bénignes.

## 5. Décisions à trancher avant de coder Phase 1

### 5.1 Storage : centralisé ou par projet ?
- **Par projet** (`<repo>/.ubik/ledger.db`) : cohérent avec RFC-004 sidecar Git, portable, versionnable
- **Centralisé** (`~/.ubik-collab/ledger.db`) : cross-projets, agrégation simple
- **Recommandation Opus** : par projet (RFC-004 cohérence), avec un index global cross-projets dans `~/.ubik-collab/index.db` pour la liste des threads.

### 5.2 Identité des agents
- Réutiliser `agent_id` QUBIK (`foundry-smith`, `react-hook-architect`) ?
- Ou créer un mapping CollabID séparé ?
- **Recommandation Opus** : réutiliser `agent_id` QUBIK directement. Pas de double registry.

### 5.3 Service local vs distant
- MVP : sidecar local sur port 7892
- Plus tard : réplication SQLite vers VM si multi-machine
- **Recommandation Opus** : local pour MVP, on traitera distribution si besoin réel

### 5.4 Bridges CLIs externes (Claude Code, Gemini, Codex)
- Pour Phase 1, agents externes ne publient pas (lecture seule depuis git commits)
- Pour Phase 2, wrapper Python `~50 lignes/CLI` qui intercepte stdout et POST vers `/threads/:id/entries`
- **Recommandation Opus** : différer les bridges externes à Phase 2, focus Phase 1 sur ubik-genie + git uniquement

## 6. Lien avec l'écosystème UBIK existant

### Synergies immédiates
- **UBIK-MEMORY** : chaque DECISION peut citer une mémoire CORTEX (`based_on: cortex_id_xxx`). Le ledger devient un index inverse de la mémoire active.
- **UBIK-DESKTOP api.rs** : les sessions ubik-genie créées via `/pty/create` peuvent automatiquement rejoindre un thread (paramètre `thread_id` optionnel dans la création).
- **spawn_mcp_terminal** (mergé aujourd'hui) : un agent qui spawn un autre agent les inscrit tous deux comme participants du thread courant.

### Ce qui reste indépendant
- ENGINE/QUBIK : pas de modification nécessaire. UBIK-COLLAB consomme les `qubik_meta` mais ne les change pas.
- Claude Code MCP : peut publier dans le bus via le sidecar Python sans modification de l'app Claude Code.

## 7. Contre-arguments & alternatives considérées

### "Pourquoi pas juste utiliser GitHub Issues / Discussions ?"
GitHub n'est pas en temps réel et n'a pas de notion de locking sémantique. Latence de 30s pour un commentaire = inacceptable pour un protocole agent-to-agent qui veut bouger à la seconde.

### "Pourquoi pas un Slack/Discord bot ?"
Couplage avec un service externe, hors UBIK ecosystem, pas de SQL queryable, pas d'intégration AST native. Mais Phase 4 pourrait ajouter un bridge Slack pour notifications humaines.

### "Pourquoi pas dans api.rs directement ?"
Couplage de responsabilités. PTY management ≠ collaboration management. Garder séparé permet de killer/upgrader l'un sans affecter l'autre.

## 8. Prochaines étapes

1. **Damien valide ou amende les décisions section 5**
2. **Setup workspace** : `cargo new --bin ubik-collab` ou monorepo dans le repo `ubik-collab` ?
3. **Phase 1 implementation** : commencer par la table SQLite et les 4 endpoints
4. **DESKTOP UI scaffolding** : `CollabMode.tsx` avec données mockées, puis brancher au backend

---

*Cette RFC complète RFC-001 (vision), RFC-002 (locking), RFC-003 (approval), RFC-004 (git migration) avec une stratégie d'intégration concrète dans UBIK-DESKTOP. Elle est destinée à être lue par une instance Opus future qui démarrera l'implémentation.*
