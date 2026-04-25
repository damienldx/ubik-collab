# RFC-004 : Migration et Coexistence Git

## Vision
UBIK-COLLAB ne remplace pas Git, il le **surplombe**. Git gère les octets (snapshots), UBIK-COLLAB gère les intentions (ledger).

## Stratégie de Coexistence : "Sidecar Ledger"
Le repo reste un repo Git standard. UBIK-COLLAB ajoute un répertoire `.ubik/` (ignoré ou versionné selon le mode) contenant le `decision-ledger.db`.

### Workflow de Migration
1. **Initialisation** : `ubik-collab init` scanne l'historique Git récent.
2. **Mapping** : Il tente de lier les derniers commits à des intentions via les messages de commit (rétro-ingénierie).
3. **Shadowing** : À chaque `git commit` fait par un agent, une entrée riche est créée dans le Ledger UBIK.

### Ledger vs Git Log
- **Git Log** : "Quoi" a changé (diff).
- **UBIK Ledger** : "Pourquoi" (Intention), "Qui" (Agent ID), "Comment" (Tests passés, alternatives rejetées).

Le Ledger remplace le `git log` pour la compréhension humaine et agentique, mais Git reste la source de vérité pour le déploiement et la CI/CD.
