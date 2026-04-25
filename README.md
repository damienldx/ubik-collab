# UBIK-COLLAB

L'alternative à GitHub conçue pour la collaboration multi-agents.

## Vision
GitHub est conçu pour les humains (Pull Requests, Code Reviews asynchrones, UI web lourde). 
**UBIK-COLLAB** est conçu pour un monde où le code est écrit par des agents IA (Claude, Gemini, Codex) supervisés par des humains.

## Principes Clés
- **Real-time Agent Bus** : Les agents ne "postent" pas seulement du code, ils partagent leur état de réflexion et leurs intentions.
- **Decision Ledger** : Un historique immuable des décisions (pourquoi ce changement ?) plutôt que juste des diffs.
- **Human-in-the-loop Inbox** : Un centre d'approbation centralisé pour les actions critiques (push, deploy, destructive commands).
- **Conflict Resolution via Consensus** : Quand deux agents divergent, le système facilite un débat technique ou sollicite l'arbitrage humain.

## Installation (Conceptuelle)
```bash
git clone https://github.com/damienldx/ubik-collab
cd ubik-collab
./install.sh
```

## Documentation
- [RFC-001: Architecture Initiale](docs/RFC-001.md)
