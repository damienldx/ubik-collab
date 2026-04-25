# RFC-002 : Locking Sémantique

## Problématique
Comment empêcher deux agents de modifier simultanément la même logique sans les figer dans des conflits de merge Git classiques ?

## Approche retenue : Lock par Chemin AST (Abstract Syntax Tree)
L'approche par **AST Path** (`file.py::Class::method`) est choisie pour sa robustesse face aux déplacements de code.

### Pourquoi pas les autres ?
- **Lignes** : Trop fragile. Un ajout de 10 lignes en haut du fichier décale tous les locks inférieurs.
- **Zone d'intention** : Trop flou pour une validation machine. Utile pour la discussion, pas pour le verrouillage.

### Mécanisme technique
1. **Analyse** : Avant modification, l'agent parse le fichier (via `tree-sitter` ou `ast` Python).
2. **Claim** : L'agent publie un message `LOCK_ACQUIRE` avec l'identifiant unique de l'objet (ex: `src/auth.py::AuthService::verify_token`).
3. **Validation** : Le serveur UBIK-COLLAB vérifie qu'aucun lock parent (la classe) ou enfant (une sous-fonction) n'est actif.
4. **Release** : Le lock est libéré au commit ou après un timeout d'inactivité.

### Edge Cases
- **Renommage** : Si un agent renomme une fonction lockée, le lock suit l'identifiant AST si possible, sinon il invalide les locks dépendants.
- **Refactor global** : Nécessite un `LOCK_ACQUIRE` sur le fichier entier ou le module.
- **Fonctions imbriquées** : Le lock est hiérarchique. Locker une classe locke implicitement toutes ses méthodes.

## Implémentation
Un service `LockManager` en Rust (intégré à UBIK-COLLAB) maintient une table en mémoire des locks actifs par thread.
