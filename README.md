# macos-platform-sso-entra

Profils de configuration pour déployer **Apple Platform SSO** adossé à **Microsoft Entra ID** sur un parc macOS, en deux vagues.

Version publique et anonymisée d'un déploiement de production : identifiants de charge utile neutralisés, aucun identifiant de locataire, UUID factices à régénérer.

## Ce que ça apporte

Sans Platform SSO, l'utilisateur a deux mots de passe qui divergent : celui de son compte local macOS et celui d'Entra ID. Les outils de synchronisation tiers comblent le trou imparfaitement — pas de notification d'expiration, synchronisation lente, désynchronisations régulières, pas de réinitialisation en libre-service.

Platform SSO en méthode `Password` fait du mot de passe Entra **la source de vérité du compte local** : l'utilisateur se connecte à son Mac avec son mot de passe d'entreprise, le changement se propage, et l'authentification unique couvre les applications Microsoft sans nouvelle saisie.

## Les deux vagues

Le déploiement est séparé en deux, parce que la première apporte déjà l'essentiel sans toucher à l'ouverture de session.

### Vague 1 : `01-sso-extension-workplace-join.mobileconfig`

Extension SSO Microsoft et Workplace Join, **sans** le bloc `PlatformSSO`.

Résultat : authentification unique silencieuse sur Microsoft 365 et Outlook, et enregistrement de l'appareil dans Entra. Le mot de passe du compte local n'est pas touché, donc aucun risque sur l'ouverture de session.

C'est l'état intermédiaire à déployer en premier et à laisser vivre. Il ne casse rien.

### Vague 2 : `02-platform-sso-full.mobileconfig`

Le même profil, **plus** le bloc `PlatformSSO`. C'est lui qui active la synchronisation du mot de passe, la notification d'expiration et la réinitialisation en libre-service.

Passer de la vague 1 à la vague 2 n'exige pas un nouveau profil : il suffit d'ajouter le dictionnaire `PlatformSSO` à l'existant. **Les postes déjà enrôlés en vague 1 n'ont rien à refaire.**

## Le point qui trompe tout le monde

```sh
app-sso platform -s
```

Cette commande renvoie **vide** tant que le bloc `PlatformSSO` n'est pas présent, même quand l'extension SSO fonctionne parfaitement et que le Mac est bien enregistré dans Entra.

Beaucoup en concluent que leur déploiement a échoué et recommencent. Ce n'est pas un échec : c'est exactement le comportement attendu en vague 1. Workplace Join et Platform SSO sont deux mécanismes distincts, seul le second alimente cette commande.

## Paramètres retenus

| Clé | Valeur | Pourquoi |
|---|---|---|
| `AuthenticationMethod` | `Password` | Le mot de passe Entra devient celui du compte local. Les méthodes `UserSecureEnclaveKey` et `SmartCard` suppriment le mot de passe mais demandent une maturité supérieure |
| `UseSharedDeviceKeys` | `false` | À activer sur les postes partagés uniquement |
| `EnableCreateUserAtLogin` | `false` | Empêche la création d'un compte local depuis l'écran de connexion tant que le pilote n'est pas validé |
| `NewUserAuthorizationMode` | `Standard` | Les comptes créés ne sont pas administrateurs |
| `TokenToUserMapping` | `preferred_username` / `name` | Correspondance entre les revendications du jeton et le compte local |

## Prérequis

- **macOS 14 minimum**, 15 recommandé.
- Application **Company Portal** installée : l'extension `com.microsoft.CompanyPortalMac.ssoextension` (Team ID `UBF8T346G9`) en dépend.
- Profil livré par un MDM. Un profil installé à la main ne déclenche pas l'enrôlement.

## Réinitialisation en libre-service : le prérequis oublié

Activer le SSPR sur des postes Platform SSO en environnement **hybride** exige que **Password Writeback soit actif dans Entra Connect**.

Sans lui, l'utilisateur réinitialise son mot de passe côté Entra, celui-ci ne redescend jamais vers l'annuaire sur site, et le compte se retrouve avec deux mots de passe différents selon la ressource consultée. Le symptôme est déroutant et la cause est très en amont.

Déployez le SSPR sur un groupe pilote restreint avant de généraliser.

## Vérification sur un poste

```sh
profiles -P | grep -i sso                       # profil présent
app-sso platform -s                             # vide en vague 1, renseigné en vague 2
sudo log show --predicate 'subsystem == "com.apple.AppSSO"' --last 10m
```

Test de la synchronisation du mot de passe : cocher **« L'utilisateur doit changer de mot de passe à la prochaine ouverture de session »** côté annuaire, puis ouvrir une session sur le Mac.

## Avant de déployer

1. **Régénérer tous les `PayloadUUID`** (`uuidgen`). Ceux fournis sont des valeurs factices lisibles, pas des identifiants valides pour un parc.
2. Adapter les `PayloadIdentifier` en `com.company.*` à votre domaine inversé.
3. Renseigner `PayloadOrganization`.

## Licence

MIT
