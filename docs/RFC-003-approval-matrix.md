# RFC-003 : Matrice d'Approbation

## Philosophie
L'autonomie des agents est proportionnelle au risque de régression ou de coût.

## Matrice d'Approbation

| Type d'Action | Niveau d'Approbation | Justification |
| :--- | :--- | :--- |
| **Refactoring interne** (privé) | Consensus Agents (2+) | Risque faible si les tests passent. |
| **Documentation / Typage** | Auto-validation | Risque nul, valeur immédiate. |
| **Ajout de dépendance** | **Humain Obligatoire** | Impact sur la sécurité et le poids du projet. |
| **Modif API Publique** | **Humain Obligatoire** | Rupture de contrat pour les autres utilisateurs. |
| **Suppression de code** | Consensus Agents + Audit | Éviter les pertes accidentelles. |
| **Fix de Bug Critique** | Auto-validation + Alerte | La vitesse prime, l'humain review a posteriori. |
| **Dépense Cloud / API** | **Humain Obligatoire** | Contrôle financier (Budget). |

## Mécanisme de Consensus
Pour les actions "Consensus Agents", un agent propose (`PROPOSE`), un autre doit valider (`VOTE_YES`) après avoir exécuté les tests localement. Si un agent `VOTE_NO`, l'action est bloquée ou escaladée à l'humain.
