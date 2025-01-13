# VoteMarket-V2

```mermaid
flowchart TB
    %% --- Personnages ---
    A["Manager"]
    X["Utilisateur / Voteur"]
    B["Contrat Votemarket"]
    D["OracleLens"]

    %% --- Flux d'actions ---
    A -->|"(1) createCampaign()"| B
    X -->|"(2) Vote sur un gauge"| D
    X -->|"(3) Claim Reward"| B
    B -.->|"(4) Get Data from Oracle"| D
    
    B --> G{"(5) Éligible ?"}
    G -- "Oui" --> H["(6) Calcule la récompense et des fee"]
    H --> I["(7) Transfert du reward + fee"]

    G -- "Non" --> K["(8) Pas de récompense (ou claim invalide)"]
```

## Étapes

- Clonage du repo ✅
- Compréhension du besoin à l’aide du README ✅
- Lecture du script de déploiement pour connaître rapidement quel contrat dépend de qui ✅
- Génération d’un diagramme de classes pour connaître les interdépendances ✅
- Rédaction d’un flowchart pour avoir une bonne vue d’ensemble ✅
- Lecture en détail du code pour comprendre certains aspects techniques ✅
- Recherche des pistes d’amélioration

## Analyse

L’idée globale est de mettre en place un mécanisme permettant de :

 1. Distribuer une portion de l’inflation journalière d’un token (ou toute autre source de récompense) pour améliorer l’APR des pools de liquidité.
 2. Incentiver les votes dans des gauges, en récompensant proportionnellement la participation au vote.
 3. Coordonner la logique sur plusieurs chaînes (ex. L1 et L2) grâce à un système de cross-chain messaging (via LaPoste, Adapter, TokenFactory).

Les éléments centraux sont donc :

- Des contrats cross-chain (LaPoste, Adapter, TokenFactory) permettant de verrouiller, déplacer, ou burn/mint des tokens entre plusieurs chaînes, et d’envoyer des messages (payload) pour synchroniser l’état ou initier des actions.
- Un module VoteMarket permettant de créer et gérer des campagnes (périodes d’incitation) au vote.
- Un Oracle (avec RLP, Merkle Patricia Proof, etc.) pour valider les votes, associant un bloc L1 (ou un epoch) à l’état du protocole.

La finalité : permettre, par exemple, qu’un vote effectué sur L1 (ou sur un protocole type Curve) déclenche l’attribution de récompenses sur L2, tout en gardant une cohérence temporelle entre les chaînes (via la mise à jour du block hash/time L1 sur L2).

### VoteMarket

Idée : Permettre à des utilisateurs de voter sur des gauges (ou tout autre système de gouvernance). Les votes sont validés par un Oracle, puis un système de campagnes distribue des récompenses en fonction des votes.

Qu’est-ce qu’une campagne ?

- Une campagne a un certain nombre de périodes (epochs) pendant lesquelles on distribue des récompenses.
- Les récompenses sont calculées en fonction de la part de vote de l’utilisateur sur le gauge (ex. proportion du total des votes).
- On peut prélever un frais (ex. 4 %) sur chaque claim, distribué au feeCollector.
- Les utilisateurs peuvent réclamer (claim) pendant la campagne et jusqu’à la fin de la fenêtre de claim. Au‐delà, la campagne est fermée et les jetons restants sont renvoyés à un destinataire (ex. le manager).

Spécificité intéressante : Claim

- Un utilisateur peut réclamer pour lui‐même ou pour quelqu’un d’autre, tant que l’autre n’est pas un compte protégé.
- Idéal pour les contrats ou les plateformes tierces qui veulent faciliter la récupération des rewards pour leurs utilisateurs.

Pourquoi plutôt en L2 ?

- Le contrat manipule beaucoup de données (vérif. de votes, proofs, etc.). Les opérations étant coûteuses, la déployer sur un L2 (plus scalable, frais moindres) est souvent plus sensé.
- On peut, via les contrats cross-chain (LaPoste, Adapter, TokenFactory), approvisionner le VoteMarket en récompenses depuis la L1 ou d’autres chaînes.

### Oracle & StateProofVerifier

Rôle principal de l’Oracle

- Associer un epoch (période) à un block (ou un timestamp) spécifique.
- Enregistrer des données de vote (point, slope/bias) pour chaque epoch, fournies par un dataProvider autorisé (prévention des écritures malveillantes).
- Gérer un mapping epoch → blockHeader (un bloc L1 donné est ainsi relié à un epoch sur L2). Cela permet de prouver qu’un vote a été effectué avant/à un certain bloc L1, et donc pendant la bonne période.

Slope & Bias

- slope : taux de décroissance de la puissance de vote dans le temps.
- bias : valeur actuelle de la puissance de vote, décroissant linéairement jusqu’à zéro.

Dans le cas d’une veTokenomics (type Curve), la puissance de vote d’un utilisateur (son bias) diminue au fil des blocs jusqu’à l’expiration de son lock. La slope indique la vitesse de cette décroissance.

Pourquoi un stateRoot / RLP / Merkle Patricia ?

- Pour prouver qu’à un certain bloc Ethereum (L1), l’utilisateur avait un certain bias ou avait voté dans un gauge.
- En pratique, on récupère sur L2 des Merkle Patricia Proofs permettant de reconstruire la donnée du compte (ou du slot de stockage) sur L1.
- L’Oracle s’appuie sur ce mécanisme pour associer un bloc L1 (via blockhash, stateRoot) à un epoch L2, garantissant l’alignement temporel et l’authenticité des votes.

#### Zoom sur le StateProofVerifier

Le StateProofVerifier est un module (souvent une bibliothèque) qui facilite :

1. Décodage d’un en-tête de bloc (RLP-encoded)
   - RLP (Recursive Length Prefix) est le format de sérialisation d’Ethereum.
   - Avec des fonctions comme parseBlockHeader(...), on extrait :
   - Le stateRoot (racine Merkle Patricia de l’état),
   - Le number (numéro du bloc),
   - Le timestamp, etc.

2. Vérification de la validité du bloc
   - Après avoir calculé keccak256 de l’header RLP, on le compare à blockhash(header.number) (uniquement possible pour les 256 derniers blocs).
   - Si la comparaison échoue, la preuve est rejetée.

3. Extraction et vérification d’un compte
   - extractAccountFromProof(...) prend en entrée la Merkle Patricia Proof liée à un stateRoot, et la clé (keccak256(address)).
   - Il reconstitue (ou prouve l’absence) du compte dans la state trie :
   - Le nonce,
   - La balance,
   - Le storageRoot,
   - Le codeHash.
   - Si le compte n’existe pas, on en déduit que l’utilisateur n’a pas voté ou n’a pas de balance, par exemple.

4. Extraction et vérification d’une slot de stockage
   - extractSlotValueFromProof(...) reconstitue la valeur stockée dans un slot (variable) d’un contrat (ex. un gauge ou un VotingEscrow).
   - On fournit la preuve Merkle Patricia depuis la racine storageRoot du compte, plus la clé du slot.
   - Le module renvoie la valeur effectivement enregistrée dans ce slot.

Comment cela sert la logique de VoteMarket ?

- Sur L2, on peut fournir (depuis un dataProvider ou un bridging) :
    1. Le bloc header L1 (RLP)
    2. Les preuves Merkle Patricia correspondantes.
- Le StateProofVerifier vérifie ces preuves on-chain, prouvant que, dans le bloc L1 X :
  - L’utilisateur U avait un certain bias/slope, ou
  - Le gauge G mentionne un vote actif pour U, etc.
- L’Oracle enregistre ensuite ces données pour l’epoch associé (ex. _l1Timestamp / 1 weeks * 1 weeks), permettant à VoteMarket de calculer combien de récompenses accorder à l’utilisateur U.

```mermaid
flowchart TB
    A["Bloc Header (RLP-encodé)"] --> B["(1) Contrat reçoit<br/>_headerRlpBytes"]
    B --> C["(2) parseBlockHeader() <br/>via RLPReader"]
    C -->|"Décodage RLP :<br/>- stateRoot<br/>- number<br/>- timestamp<br/>- ..."| D["BlockHeader struct"]
    D --> E["(3) Calcul<br/>header.hash = keccak256(...)"]
    E -->|"(4) Comparaison<br/>hash vs blockhash(number)"| F["Vérification du bloc"]
    F --> G{"BlockHeader<br/>Valide ?"}
    G -- "Oui" --> H["(5) root = stateRootHash<br/>utilisé pour la Merkle Patricia Proof"]
    G -- "Non" --> I["Rejet / Erreur"]
```

Ainsi, StateProofVerifier apporte la brique de confiance nécessaire pour que VoteMarket puisse récompenser les utilisateurs en fonction de leurs votes, sans qu’il faille toute la state trie de L1.

### L1Sender

- **Rôle** : Envoyer depuis L1 vers L2 des informations de bloc (block number, block hash, timestamp).
- **Fonctionnement** :
  - À intervalles réguliers (cf. currentPeriod()), L1Sender appelle broadcastBlock(...).
  - Cela encode (block.number - 1, blockhash(block.number - 1), block.timestamp).
  - Passe le message à LaPoste (ou un système équivalent) pour qu’il soit dispatch sur la chaîne destinataire (L2).
  - Sur L2, un contrat (souvent L1BlockOracleUpdater) reçoit ce message et met à jour un Oracle ou d’autres modules.
- **Rôle** :
  - Permet de s’assurer que L2 connaît le bloc L1 de référence (et son hash). On peut ainsi définir des epochs sur L2 en fonction du block.timestamp ou block.number L1, garantissant la cohérence temporelle ou de consensus.

### L1BlockOracleUpdater

- **Rôle** : Sur L2, réceptionne le bloc hash/number/time envoyé depuis L1 (par L1Sender) et l’insère dans l’Oracle approprié.
- **Fonctionnement** :
  - updateL1BlockNumber :
    - Parfois, si la chaîne L2 a un L1_BLOCK_ORACLE, on peut directement y lire la valeur ; sinon, on compte sur le message de L1Sender.
  - receiveMessage :
    - Appelée quand un message cross-chain arrive (via LaPoste).
    - Décodage (uint256 _l1BlockNumber, bytes32_l1BlockHash, uint256 _l1Timestamp).
    - Stockage dans un contrat Oracle (ex. IOracle(ORACLE).insertBlockNumber(...)), associé à un epoch (souvent arrondi à la semaine).

- **Usage dans VoteMarket** :
  - Permet de déterminer la fin d’une période de vote ou d’une campagne en se basant sur le bloc/timestamp L1, évite la dépendance à la clock L2.
  - Garantit qu’un vote fait après tel bloc L1 n’est pas valide pour l’epoch en cours, par exemple.

## Architecture

```bash
sol2uml class ./packages/votemarket -f png -o ./classDiagram.png --hideInterfaces
```

![class diagram](./asset/classDiagram.png)

### Pistes d'amélioration

1. **Expiration ou archivage des epochs anciens**

- Après un certain délai, il peut être inutile de conserver en permanence dans le storage les données d’epochs trop anciens.
- On pourrait imaginer une fonction d’archivage ou de cleanup pour libérer du stockage (et potentiellement baisser les coûts de lecture/écriture) si l’epoch est suffisamment lointain.

2. **Événements plus granulaires**

- Actuellement, il n’y a pas forcément d’événement (event) détaillant l’insertion d’un nouveau BlockHeader ou de nouveaux points de vote. Ajouter des events pour chaque insertion (PointInserted, SlopeInserted, BlockHeaderInserted) faciliterait la traçabilité.

3. **Gestion avancée des permissions**

- À la place d’un simple mapping(address => bool) pour les dataProviders, on pourrait centraliser un système de rôles via un RoleManager (type AccessControl), permettant une gestion plus fine (ex. ROLE_DATA_PROVIDER, ROLE_BLOCK_PROVIDER).
- Cela pourrait inclure des possibilités de mise à jour ou de révocation plus souples.

4. **Optimisation des structures**

- Les structs Point et VotedSlope sont simples, mais on pourrait envisager d’utiliser des types plus compacts (ex. uint128 si les valeurs ne dépassent jamais 2^128, etc.) pour réduire le coût en gas.
- Vérifier s’il est nécessaire d’avoir des timestamps en uint256 ou s’ils peuvent être en uint40/uint48 (selon la durée d’utilisation prévue).

5. **Contrôle de cohérence**

- À la création ou mise à jour d’un Point ou d’un VotedSlope, ajouter un check pour éviter des incohérences (par exemple, si lastUpdate est inférieur à la lastVote, etc.).
