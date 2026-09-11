# Prospection Agri Santerre

Outil de prospection terrain avec podium d'équipe partagé.
Page statique unique, hébergeable sur GitHub Pages ; les statuts de prospection
sont synchronisés entre tous les commerciaux via Supabase.

---

## Mise en service — 10 minutes, 3 étapes

### 1. Créer la base Supabase

1. Créer un compte sur [supabase.com](https://supabase.com) puis un projet (offre gratuite).
2. Ouvrir **SQL Editor → New query**, coller tout le contenu de [`supabase.sql`](supabase.sql), cliquer **Run**.
3. Aller dans **Project Settings → Data API** et noter :
   - **Project URL** → ressemble à `https://abcdefgh.supabase.co`
   - **API Key « anon / public »** → longue chaîne commençant par `eyJ…`

> La clé `anon` est faite pour vivre dans une page web publique.
> Les règles RLS posées par le script autorisent la lecture, la création et la
> modification des fiches de suivi — et **rien d'autre** : aucune suppression
> n'est possible, et la clé ne donne accès à aucune autre table.

### 2. Choisir le code d'accès

Ouvrir [`code-acces.html`](code-acces.html) dans un navigateur (double-clic suffit),
taper le code voulu, copier l'empreinte affichée.

### 3. Renseigner `index.html`

Ouvrir `index.html`, tout en haut du second `<script>` :

```js
const CONFIG = {
  SUPABASE_URL:       "https://abcdefgh.supabase.co",
  SUPABASE_ANON_KEY:  "eyJhbGciOi…",
  ACCESS_CODE_SHA256: "3a7bd3e2360a…"
};
```

Pousser sur GitHub, activer **Settings → Pages → Deploy from branch**.
C'est en ligne.

Pour retirer le code d'accès, laisser `ACCESS_CODE_SHA256: ""`.

---

## Ce qui est partagé, ce qui ne l'est pas

| Donnée | Portée |
|---|---|
| Contacté / réponse / projet / notes | **partagé** — visible par toute l'équipe |
| Podium et KPIs | **partagé** — calculés sur les données de tous |
| Nom du vendeur actif | local à l'appareil |
| Fichier clients (`DATA`) | figé dans la page, identique pour tous |

## Comment marche la synchronisation

- **Au démarrage** : affichage immédiat du dernier état connu (cache local),
  puis rattrapage complet depuis Supabase.
- **Ensuite** : relecture des seules fiches modifiées toutes les 15 secondes,
  plus un rafraîchissement au retour sur l'onglet. Le point vert à droite des
  onglets indique l'état.
- **À chaque saisie** : la fiche part dans une file d'envoi. En cas de coupure
  réseau — fréquent sur le terrain — elle est conservée et repart toute seule
  au retour de la connexion, y compris après fermeture du navigateur.
- **Pendant qu'on saisit** : la liste ne se redessine jamais sous les doigts.
  Tant qu'une fiche est ouverte, les mises à jour des collègues sont chargées
  en arrière-plan et appliquées à la fermeture de la fiche.

## Le podium

Chaque action est créditée à celui qui l'a réellement saisie, et non au dernier
qui a touché la fiche : `contacte_par`, `reponse_par` et `projet_par` sont
enregistrés séparément. Si Marc contacte un client et que Julie lève le projet,
Marc garde son contact et Julie obtient le projet.

Le classement est trié par projets levés, puis réponses obtenues, puis contacts.

## Limites connues

- **Notes simultanées** : si deux vendeurs modifient les notes de la *même*
  fiche en même temps, la dernière saisie enregistrée écrase l'autre.
- **Code d'accès** : il filtre les curieux, ce n'est pas un chiffrement. Le
  fichier clients reste techniquement lisible par qui possède l'URL du site.
  Pour un vrai cloisonnement, il faut passer au login par e-mail (Supabase Auth)
  ou héberger le site en privé.
- **`code-acces.html`** est un utilitaire de configuration : inutile de le publier.

## Mettre à jour le fichier clients

Les clients sont figés dans le premier `<script>` d'`index.html` (constante `DATA`).
Remplacer ce bloc met à jour le fichier ; les suivis déjà saisis restent liés par
`client_id`, à condition que les identifiants clients ne changent pas.
