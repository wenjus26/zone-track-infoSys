# zone-track-infoSys
ZTIS - Suivi et gestion des opérations logistiques


# Dictionnaire de données — Zone Track Information Systems

**Version :** v1.0  
**Date :** 2025-08-25  
**Objectif :** Documenter les entités, attributs, contraintes et relations du SI Zone Track Information Systems.  
**Périmètre :** Gestion multi-tenant des opérations logistiques (camions, conducteurs, entrepôts, opérations, pesées, scellés, laissez-passer, documents).

---

## 1) Conventions
- **Nommage** : Modèles en PascalCase (Django), champs en snake_case.
- **Dates** : `YYYY-MM-DD` ; Date+heure en UTC `YYYY-MM-DD HH:MM:SS`.
- **Audit** (si présent) : `created_at`, `created_by`, `edited_at`, `edited_by`.
- **Rôles clés** : `approved_by/at/me`, `unapproved_by/at/me` pour les validations.

---

## 2) Vue d’ensemble des entités

| Modèle | Description | Clé primaire | Relations clés |
|---|---|---|---|
| UserProfile | Secret OTP TOTP par utilisateur | id | 1–1 User |
| Profile | Signature et poste utilisateur | id | 1–1 User |
| UserDevice | Limite d’appareils par utilisateur | id | 1–1 User ; 1–N Device |
| Device | Appareil associé à un utilisateur | id | N–1 UserDevice |
| DriverInfo | Conducteur | id | N–M TruckInfo via DriverTruckAssignment |
| TruckInfo | Camion | id | N–M DriverInfo ; audit User |
| DriverTruckAssignment | Affectation conducteur–camion | id | FK DriverInfo, TruckInfo |
| Tenant | Locataire (multi-tenant) | id | N–M Warehouse ; 1–N référentiels & opérations |
| UserTenant | Association user ↔ tenant | id | FK User, Tenant (unique) |
| Warehouse | Entrepôt/magasin | id | N–M Tenant ; 1–N Lot |
| Lot | Lot/stack | id | FK Warehouse, Tenant |
| UserWarehouse | Accès user ↔ warehouse | id | FK User, Warehouse (unique) |
| Handler | Manutentionnaire | id | FK Tenant |
| DepartmentEmailConfig | Routage e-mail par département | id | FK Tenant |
| Product | Référentiel produit | id | FK Tenant |
| Destination | Référentiel destination | id | FK Tenant |
| Origin | Référentiel origine | id | FK Tenant |
| PackagingType | Référentiel emballage | id | FK Tenant |
| Partner | Partenaire (client/fournisseur) | id | FK Tenant |
| Cha | Agent en douane | id | FK Tenant (unique par tenant+nom) |
| Booking | Réservation | id | FK Tenant, FK Cha (opt) |
| Operation | Opération logistique (centrale) | id | Multiples FK (tenant, assignment, partner, etc.) |
| OperationInitialWeight | Poids initial & WS | id | 1–1 Operation |
| OperationFinalWeight | Poids final & net | id | 1–1 Operation |
| OperationFinalWeightImage | Image ticket poids final | id | 1–1 OperationFinalWeight |
| OperationContent | Contenu (packaging/lot) | id | 1–1 Operation ; FK Warehouse, Lot, PackagingType, Handler |
| OperationSeal | Scellé (n° + image) | id | 1–1 Operation |
| GatePass | Laissez-passer | id | 1–1 Operation ; FK DepartmentEmailConfig |
| OperationSupportingDocument | Pièces jointes d’une opération | id | N–1 Operation (unique par type) |
| AccessAttempt | Tentatives d’accès | id | — |

---

## 3) Détail des entités et champs

> Abréviations : **NN** = Non Null, **UQ** = Unique, **IDX** = Index, **FK** = Clé étrangère, **DF** = Valeur par défaut.

### 3.1 UserProfile
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| user | FK(User) | ✓ | 1–1 | Utilisateur lié |
| otp_secret | Char(32) | ✓ | DF généré ; TOTP interval=60s | Secret TOTP |

---

### 3.2 Profile
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| user | FK(User) | ✓ | 1–1 | Utilisateur |
| user_signature | Image | ✓ | DF `user_signatures/default_signature.jpg` | Signature |
| position | Char(100) |  |  | Fonction/position |

---

### 3.3 UserDevice
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| user | FK(User) | ✓ | 1–1 | Utilisateur |
| device_limit | Integer | ✓ | DF=3 | Nombre max d’appareils |

**Device**
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| device_id | Char(255) | ✓ | UQ | Identifiant unique appareil |
| user_device | FK(UserDevice) |  |  | Groupe d’appareils de l’utilisateur |

---

### 3.4 DriverInfo
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| name | Char(255) | ✓ |  | Nom |
| phone_number | Char(20) | ✓ | IDX | Téléphone |
| license_number | Char(100) |  |  | Numéro permis |
| created_at | DateTime | ✓ | DF=auto | Création |
| created_by | FK(User) |  | IDX | Auteur création |
| edited_at | DateTime |  |  | Dernière maj |
| edited_by | FK(User) |  | IDX | Auteur maj |

---

### 3.5 TruckInfo
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| truck_number | Char(100) | ✓ | UQ, IDX | Identifiant camion |
| plate_number | Char(100) |  |  | Immatriculation |
| brand | Char(100) |  |  | Marque |
| model | Char(100) |  |  | Modèle |
| created_at | DateTime |  |  | Création |
| created_by | FK(User) |  | IDX | Auteur création |
| edited_at | DateTime |  |  | Maj |
| edited_by | FK(User) |  | IDX | Auteur maj |

---

### 3.6 DriverTruckAssignment
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| driver | FK(DriverInfo) | ✓ | IDX | Conducteur |
| truck | FK(TruckInfo) | ✓ | IDX | Camion |
| assigned_at | DateTime | ✓ | IDX | Date d’affectation |
| status | Choice | ✓ | DF=`assigned` | `assigned` \| `unassigned` |

---

### 3.7 Tenant
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| name | Char(255) | ✓ | IDX | Dénomination |
| contact_person | Char(255) |  |  | Contact |
| tenant_code | Char(10) |  | IDX | Code court |
| email | Email |  | IDX | Email |
| address | Text |  |  | Adresse |
| logo | Image |  | chemin: `tenant_logos/<code>_logo.ext` | Logo |
| created_at | DateTime | ✓ | IDX | Création |

**UserTenant**
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| user | FK(User) | ✓ |  | Utilisateur |
| tenant | FK(Tenant) | ✓ |  | Locataire |
| (contrainte) | — | — | Unique (`user`,`tenant`) | Unicité |

---

### 3.8 Warehouse / Lot / UserWarehouse
**Warehouse**
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| name | Char(255) | ✓ |  | Nom |
| location | Char(255) | ✓ | IDX | Localisation |
| tenants | M2M(Tenant) |  |  | Locataires |
| created_at | DateTime | ✓ |  | Création |

**Lot**
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| name | Char(255) | ✓ |  | Nom du lot |
| description | Text | ✓ |  | Description |
| warehouse | FK(Warehouse) | ✓ | IDX | Entrepôt |
| tenant | FK(Tenant) | ✓ | IDX | Locataire |
| created_at | DateTime | ✓ |  | Création |

**UserWarehouse**
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| user | FK(User) | ✓ |  | Utilisateur |
| warehouse | FK(Warehouse) | ✓ |  | Entrepôt |
| (contrainte) | — | — | Unique (`user`,`warehouse`) | Unicité |

---

### 3.9 Handler
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| name | Char(255) | ✓ |  | Nom |
| contact_info | Char(255) |  |  | Contact |
| email | Email |  |  | Email |
| role | Char(255) |  |  | Rôle |
| tenant | FK(Tenant) | ✓ | IDX | Locataire |
| created_at | DateTime | ✓ |  | Création |

---

### 3.10 DepartmentEmailConfig
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| tenant | FK(Tenant) | ✓ | IDX | Locataire |
| department_name | Char(100) | ✓ | IDX | Département |
| report_type | Char(100) | ✓ | IDX | Type de rapport |
| recipients | Text | ✓ |  | Emails (séparés par virgules) |
| cc | Text |  |  | Copie |
| active | Bool | ✓ | DF=True | Actif ? |
| created_at | DateTime | ✓ | DF=now | Création |

---

### 3.11 Référentiels : Product / Destination / Origin / PackagingType
Champs communs :
- `tenant` FK(Tenant) **NN**, IDX  
- `name` Char(255) **NN**, IDX

---

### 3.12 Partner
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| tenant | FK(Tenant) | ✓ | IDX | Locataire |
| name | Char(255) | ✓ | IDX | Nom |
| contact | Char(255) |  |  | Coordonnées |
| partner_type | Choice | ✓ | IDX | `client` \| `supplier` |

---

### 3.13 Cha (Customs House Agent)
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| tenant | FK(Tenant) | ✓ | IDX | Locataire |
| name | Char(255) | ✓ | IDX | Nom |
| contact | Char(255) |  |  | Contact |
| (contrainte) | — | — | Unique (`tenant`,`name`) | Unicité par tenant |

---

### 3.14 Booking
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| uuid | UUID | ✓ | UQ | Identifiant |
| booking_number | Char(100) | ✓ | UQ | Numéro de booking |
| is_active | Bool | ✓ | DF=True | Actif ? |
| tenant | FK(Tenant) | ✓ | IDX | Locataire |
| cha | FK(Cha) |  | IDX, nullable | Agent douane |
| contract_number | Char(100) |  | IDX | Contrat |
| teu_plan | PositiveInteger |  | DF=0 | Plan TEU |
| quantity_plan | Decimal(15,2) |  |  | Quantité planifiée |

---

### 3.15 Operation (entité centrale)
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| uuid | UUID | ✓ | UQ | Identifiant |
| token_number | Char(50) |  | UQ, IDX | Généré à l’approbation (`<TENANT>-####`) |
| tenant | FK(Tenant) | ✓ | IDX | Locataire |
| driver_truck_assignment | FK(DriverTruckAssignment) |  | IDX | Affectation camion/conducteur |
| partner | FK(Partner) |  | IDX | Partenaire |
| product | FK(Product) |  | IDX | Produit |
| origin | FK(Origin) |  | IDX | Origine |
| destination | FK(Destination) |  | IDX | Destination |
| cha | FK(Cha) |  | IDX | Agent douane |
| no_bags | PositiveInteger | ✓ | DF=0 | Nombre de sacs |
| operation_type | Choice | ✓ |  | `export shipment` \| `import shipment` \| `exit` \| `loading` \| `unloading` \| `inspection` \| `storage` \| `shipment preparation` \| `return` \| `maintenance` \| `transfer` |
| booking | FK(Booking) |  | IDX | Réservation |
| container_number | Char(100) |  | IDX | Conteneur |
| interchange_weight | Decimal(10,2) |  |  | Poids interchange |
| vessel | Char(100) |  |  | Navire |
| container_tare | Decimal(10,2) |  |  | Tare conteneur |
| shipping_line | Choice |  | IDX | `MAERSK` \| `MSC` \| `CMA_CGM` \| `HAPAG_LLOYD` |
| container_type | Choice |  | IDX | `20GP` \| `40GP` \| `40HC` \| `20RF` \| `40RF` \| `OT` \| `FR` |
| container_condition | Choice |  | IDX | `EMPTY` \| `FULL` \| `GOOD` \| `DAMAGED` \| `MINOR_DAMAGE` \| `UNKNOWN` |
| interchange_date | Date |  |  | Date interchange |
| interchange_number | Char(100) |  |  | Réf. interchange |
| approved_by / at / me | FK(User) / DateTime / Bool |  |  | Approbation |
| unapproved_by / at / me | FK(User) / DateTime / Bool |  |  | Désapprobation |
| created_at / by | DateTime / FK(User) |  |  | Audit |
| edited_at / by | DateTime / FK(User) |  |  | Audit |

**Règles clés :**
- Token généré unique par tenant au moment de l’approbation.
- Statut d’opération dérivé de la complétude/approbation des blocs Poids, Contenu, Scellé et Gate Pass.

---

### 3.16 OperationInitialWeight
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| uuid | UUID | ✓ | UQ | Identifiant |
| operation | O2O FK(Operation) | ✓ | IDX | Opération |
| initial_weight | Float | ✓ |  | Poids initial (kg) |
| ws_number | Char(255) |  | IDX | N° ticket de pesée |
| initial_weight_edit | Float |  |  | Correction |
| created_at / by | DateTime / FK(User) |  | IDX (created_at) | Audit |
| edited_at / by | DateTime / FK(User) |  |  | Audit |

---

### 3.17 OperationFinalWeight
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| uuid | UUID | ✓ | UQ | Identifiant |
| operation | O2O FK(Operation) | ✓ | IDX | Opération |
| final_weight | Float | ✓ |  | Poids final (kg) |
| net_weight | Float |  | Calculé | \|final − initial\| |
| approved_by / at / me | FK(User) / DateTime / Bool |  |  | Approbation |
| unapproved_by / at / me | FK(User) / DateTime / Bool |  |  | Désapprobation |
| created_at / by | DateTime / FK(User) |  |  | Audit |
| edited_at / by | DateTime / FK(User) |  |  | Audit |

---

### 3.18 OperationFinalWeightImage
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| uuid | UUID | ✓ | UQ | Identifiant |
| final_weight | O2O FK(OperationFinalWeight) | ✓ |  | Lien au poids final |
| image | Image |  | upload renommé | Ticket scanné |
| cropped_image | Image |  | upload renommé | Recadrage |
| uploaded_at | DateTime | ✓ |  | Téléversement |
| final_weight_images_agent | Char(255) |  |  | Agent de saisie |

---

### 3.19 OperationContent
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| uuid | UUID | ✓ | UQ | Identifiant |
| operation | O2O FK(Operation) | ✓ |  | Opération |
| warehouse | FK(Warehouse) |  | IDX | Entrepôt |
| lot | FK(Lot) |  | IDX | Lot |
| packaging_type | FK(PackagingType) |  | IDX | Type d’emballage |
| description | Text | ✓ |  | Détails |
| content | PositiveInteger | ✓ | DF=0 | Contenu (nb unités) |
| content_leaved | PositiveInteger | ✓ | DF=0 | Restant |
| handler | FK(Handler) |  | IDX | Manutentionnaire |
| approved_by / at / me | FK(User) / DateTime / Bool |  |  | Approbation |
| unapproved_by / at / me | FK(User) / DateTime / Bool |  |  | Désapprobation |
| created_at / by | DateTime / FK(User) |  |  | Audit |
| edited_at / by | DateTime / FK(User) |  |  | Audit |

---

### 3.20 OperationSeal
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| uuid | UUID | ✓ | UQ | Identifiant |
| operation | O2O FK(Operation) | ✓ | IDX | Opération |
| seal_number | Char(100) |  | IDX | N° de scellé |
| seal_image | Image |  | upload renommé | Photo scellé |
| created_at / by | DateTime / FK(User) |  | IDX (created_at) | Audit |
| edited_at / by | DateTime / FK(User) |  |  | Audit |

---

### 3.21 GatePass
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| uuid | UUID | ✓ | UQ | Identifiant |
| operation | O2O FK(Operation) | ✓ | IDX | Opération |
| gate_pass_number | Char(255) |  | UQ, IDX | `{TENANT}_GP-YYYYMMDD-###` |
| valid_until | DateTime | ✓ |  | Validité |
| department | FK(DepartmentEmailConfig) |  | IDX | Dépt. émetteur |
| remarks | Text |  |  | Remarques |
| validity_status | Char(50) |  | IDX | `Valide` \| `Expiré` (à l’approbation) |
| approved_by / at / me | FK(User) / DateTime / Bool |  |  | Approbation |
| unapproved_by / at / me | FK(User) / DateTime / Bool |  |  | Désapprobation |
| created_at / by | DateTime / FK(User) |  |  | Audit |
| edited_at / by | DateTime / FK(User) |  |  | Audit |

---

### 3.22 OperationSupportingDocument
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| uuid | UUID | ✓ | UQ | Identifiant |
| operation | FK(Operation) | ✓ | IDX | Opération |
| document_name | Choice | ✓ | Unique par opération | Type de pièce |
| document_number | Char(100) |  | IDX | Référence |
| document_file | File |  | upload renommé | Fichier |
| created_at / by | DateTime / FK(User) |  | IDX (created_at) | Audit |
| edited_at / by | DateTime / FK(User) |  |  | Audit |

**DOCUMENT_NAME_CHOICES** : `LOADING_SLIP`, `QUALITY_SLIP`, `INTERCHANGE`, `EMPTY_CONTAINER`, `DRESSING`, `LOADING`.

---

### 3.23 AccessAttempt
| Champ | Type | NN | Contraintes / DF | Description |
|---|---|---|---|---|
| username | Char(255) | ✓ |  | Nom d’utilisateur |
| attempt_time | DateTime | ✓ | DF=now | Horodatage |
| failed_attempt | Bool | ✓ | DF=True | Échec ? |
| ip_address | IP | ✓ |  | Adresse IP |
| user_agent | Char(255) | ✓ |  | Navigateur/appareil |

---

## 4) Règles métier transverses
- **Génération du token d’opération** : unique par tenant (`<TENANT>-####`), créé lors de l’approbation.
- **Poids net** : calculé automatiquement à l’enregistrement du poids final à partir du poids initial.
- **Statut d’une opération** : dépend de la complétude et de l’approbation des blocs Poids initial/final, Contenu, Scellé et Gate Pass.
- **Documents d’une opération** : au plus **un** document par `document_name` et par opération.

---

## 5) Sécurité & Sensibilité (recommandations)
- **Données personnelles** : `DriverInfo.phone_number`, signatures, emails → accès restreint.
- **Journalisation** : exploiter `created_*`, `edited_*`, `approved_*`, `unapproved_*` pour traçabilité.
- **Multi-tenant** : filtrage systématique par `tenant`.

---

## 6) Glossaire
| Terme | Définition |
|---|---|
| Gate Pass | Laissez-passer autorisant la sortie |
| WS | Weighing Slip (ticket de pesée) |
| CHA | Customs House Agent (agent en douane) |
| TEU | Twenty-foot Equivalent Unit |

