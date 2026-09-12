# Prospection Agri Santerre

Outil de prospection terrain avec podium d'équipe partagé et **cloisonnement par
vendeur** : chacun ne voit que son portefeuille, mais tout le monde voit le classement.

Page statique publiable sur GitHub Pages. Les données vivent dans Supabase, jamais
dans le code.

---

## Ce qui est cloisonné, ce qui est partagé

| | Vendeur | Administrateur |
|---|---|---|
| Ses clients et leur parc matériel | ✅ | ✅ |
| Clients d'un collègue | ❌ jamais transmis | ✅ |
| Clients « Non attribué » / « hors secteur » | ❌ | ✅ |
| Ses suivis et notes | ✅ | ✅ |
| Suivis et notes d'un collègue | ❌ jamais transmis | ✅ |
| **Podium : nom + nombre de projets, réponses, contacts de chacun** | ✅ | ✅ |
| Import du fichier clients | ❌ | ✅ |

Le filtrage est fait par **Postgres**, pas par le navigateur. Un vendeur qui ouvre la
console et trafique la page ne fera pas remonter une ligne de plus : les données d'un
collègue ne quittent jamais le serveur. Le podium passe par une fonction dédiée qui ne
renvoie que des compteurs, jamais de lignes clients.

---

## Mise en service

### 1. Le schéma

Supabase → **SQL Editor** → **New query** → coller tout [`supabase-v2.sql`](supabase-v2.sql) → **Run**.

Le script est ré-exécutable. Il crée les tables, les règles d'accès, le podium, la
fonction d'import — et **supprime les anciennes règles « anon » de la v1**, qui
donnaient accès à tout le monde.

> **Avant de lancer, relisez la section 9 du script** : la liste de vos 16 vendeurs.
> Les e-mails sont **déduits** du format du vôtre (`ldegryck@` → initiale du prénom +
> nom). Un e-mail faux = un vendeur qui ne pourra pas se connecter. Vérifiez aussi
> `sect.nouvion`, qui ressemble à un secteur plutôt qu'à une personne.

### 2. Les comptes

Supabase → **Authentication** → **Users** → **Add user** pour chaque vendeur, avec
l'e-mail exact de la table `vendeurs`. Cochez « Auto Confirm User ».

Le rattachement compte ↔ portefeuille est automatique : un déclencheur relie le compte
dès sa création. Si vous aviez créé des comptes avant d'exécuter le script, lancez une
fois dans le SQL Editor :

```sql
select public.resynchroniser_comptes();
```

Vérifiez ensuite qui est rattaché :

```sql
select vendeur_id, email, nom_affiche, admin, (user_id is not null) as compte_cree
  from public.vendeurs order by admin desc, nom_affiche;
```

**Désactivez l'inscription libre** : Authentication → Providers → Email → décocher
*Enable sign-ups*. Ce n'est pas indispensable — un compte dont l'e-mail est inconnu de
la table `vendeurs` ne voit strictement rien — mais autant fermer la porte.

### 2 bis. Comptes : les vendeurs choisissent leur mot de passe

Vous n'avez plus à créer les comptes un par un. À la première ouverture, le vendeur
clique sur **« Première connexion ? Choisir mon mot de passe »**, saisit son adresse
professionnelle et le mot de passe de son choix. Il est connecté dans la foulée.

Votre seul geste : **que son adresse figure dans la table `vendeurs`**. C'est elle qui
fait office d'invitation.

Dans Supabase, *Authentication* → *Providers* → *Email* : laissez **Enable sign-ups
activé** — sans quoi l'écran de création refusera tout le monde.

> **Le verrou qui rend cela sûr.** Ouvrir l'inscription, c'est normalement laisser
> n'importe qui créer un compte — et surtout laisser un inconnu **prendre l'adresse
> d'un vendeur avant lui**. Le déclencheur `verifier_inscription` refuse, au niveau de
> la base, toute adresse absente de `vendeurs` ou désactivée. Ni l'application, ni un
> appel direct à l'API ne peuvent passer outre.
>
> Conséquence à connaître : la création manuelle d'un compte dans *Authentication →
> Users* obéit au même verrou. Ajoutez la personne à `vendeurs` d'abord, créez son
> compte ensuite.

**Deux niveaux de sécurité, selon votre réglage** *Authentication → Providers → Email
→ Confirm email* :

| Confirm email | Ce que ça donne |
|---|---|
| **Désactivé** | Le vendeur est connecté immédiatement. Aucun e-mail à configurer. Risque résiduel : quelqu'un connaissant l'URL **et** une adresse de la liste pourrait s'inscrire à la place d'un vendeur qui ne s'est pas encore connecté. Le vendeur s'en apercevrait aussitôt (son adresse serait déjà prise). |
| **Activé** | Le vendeur doit ouvrir un lien reçu par e-mail : l'usurpation devient impossible. Exige un **SMTP configuré** — le serveur d'essai de Supabase est bridé à quelques envois par heure et ne tiendra pas 17 inscriptions. |

En pratique : faites créer les comptes pendant une réunion d'équipe, confirmation
désactivée, et le problème ne se pose pas. Pour une mise en service étalée, configurez
un SMTP et activez la confirmation.

**Longueur du mot de passe** : la page n'impose rien. C'est Supabase qui tranche, via
*Authentication* → *Providers* → *Email* → **Minimum password length** (6 par défaut).
Pour être plus ou moins permissif, c'est là que ça se règle — le refus éventuel est
affiché en français au vendeur.

**Mot de passe oublié** : la réinitialisation en autonomie demande elle aussi un SMTP.
Sans cela, vous changez le mot de passe depuis *Authentication → Users → …  → Reset
password*. Le SSO Microsoft ci-dessous supprime entièrement ce problème.

### 2 ter. SSO Microsoft (facultatif mais recommandé)

Les vendeurs se connectent avec leur compte Microsoft 365 : aucun mot de passe à
créer, distribuer ni réinitialiser. C'est inclus dans le plan gratuit de Supabase.
La connexion par mot de passe reste disponible en secours, repliée sous le bouton.

**Côté Microsoft** — portail Azure → *Microsoft Entra ID* → *App registrations* →
*New registration* :

- *Name* : `Prospection Agri Santerre`
- *Supported account types* : **Accounts in this organizational directory only**
  (c'est ce qui interdit l'accès aux comptes Microsoft extérieurs)
- *Redirect URI* : type **Web**, valeur `https://ufpiusdeicxjnctukojq.supabase.co/auth/v1/callback`

Relevez l'**Application (client) ID** et le **Directory (tenant) ID**, puis créez un
secret dans *Certificates & secrets* → *New client secret* et copiez sa **valeur**
(elle n'est affichée qu'une fois). Notez sa date d'expiration : le jour où le secret
expire, le SSO cesse de fonctionner.

**Côté Supabase** — *Authentication* → *Providers* → *Azure* → activer, puis saisir
le client ID, le secret, et dans *Azure Tenant URL* :
`https://login.microsoftonline.com/<votre-tenant-id>`.

**Puis** *Authentication* → *URL Configuration* : renseigner *Site URL* avec l'adresse
GitHub Pages, et ajouter dans *Redirect URLs* :

```
https://<votre-compte>.github.io/<votre-depot>/**
```

Sans cette autorisation, Microsoft renverra les vendeurs vers une erreur de
redirection. L'application demande le retour sur sa propre adresse
(`origin + chemin`), que le joker `/**` couvre dans les deux formes possibles
(`…/` et `…/index.html`).

**Pour changer de fournisseur ou revenir au seul mot de passe**, une ligne suffit en
tête d'`index.html` et d'`admin.html` : `SSO_PROVIDER: "azure"`, `"google"`, ou `""`.

> **Le SSO n'ouvre aucune porte sur les données.** Toute personne du domaine peut
> s'authentifier — la comptabilité, un alternant — mais sans ligne dans `vendeurs`
> elle ne voit **rien**, pas un client, pas un chiffre. Le rattachement se fait par
> e-mail, exactement comme pour un compte à mot de passe.

> **Se déconnecter n'annule que la session locale**, pas la session Microsoft du
> navigateur : un nouveau clic sur le bouton reconnecte sans ressaisie. C'est le
> comportement normal du SSO, à connaître sur un appareil partagé.

### 3. Le fichier clients

Ouvrez [`admin.html`](admin.html) depuis votre poste, connectez-vous avec votre compte
administrateur, et déposez [`clients.json`](clients.json) : 2 704 clients et 3 757
machines, importés par lots de 300.

Ensuite, à chaque mise à jour, vous redéposez simplement votre nouvel export.
L'import est **non destructif** : les suivis de prospection ne sont jamais touchés, et
un champ absent de l'export laisse la valeur en base intacte plutôt que de l'écraser.

### 4. Publication

Poussez sur GitHub — `index.html` et `admin.html` sont déjà renseignés avec votre URL
et votre clé publique Supabase. Puis Settings → Pages → *Deploy from a branch*, `main`,
`/ (root)`.

> ### ⚠️ Ne publiez jamais `clients.json`
> Le [`.gitignore`](.gitignore) l'exclut, ainsi que tout `.csv` et `.xlsx`. GitHub Pages
> sert **tout** ce qui est dans le dépôt : un export déposé là serait téléchargeable par
> quiconque connaît l'URL, et réduirait à néant le cloisonnement. Les données n'ont
> qu'un seul chemin : votre poste → `admin.html` → Supabase.

---

## Format d'import

**JSON** — un tableau d'objets ; seul `id` est obligatoire, c'est lui qui identifie le
client d'un import à l'autre :

```json
[{ "id":"881701056", "nom":"3B CANAM NORMANDIE", "adr":"7 AVENUE… 76250 DEVILLE",
   "naf":"Commerce de voitures…", "tel":["0232767474"],
   "commercial":"verleye.david", "nb_machines":4, "annee_achat":2026,
   "machines":[{ "marque":"BRP","modele":"CAN-AM","categorie":null,
                 "etat":"NEUF","plaque":"HK-728-PF","annee":2026 }] }]
```

**CSV** (séparateur `;` ou `,`) — en-têtes reconnus : `id, nom, adr, naf, tel,
commercial, nb_machines, annee_achat`. Met à jour les clients et leur attribution,
**sans toucher au parc matériel**. Plusieurs téléphones se séparent par un espace.

Le champ `commercial` doit contenir **l'identifiant du vendeur** (`verleye.david`),
exactement comme dans la table `vendeurs`. Toute autre valeur — `Non attribué`, une
faute de frappe — rend le client visible de l'administrateur seul.
Pour transférer un portefeuille, il suffit donc de réimporter avec le nouveau nom.

---

## Le podium

Chaque action est créditée à celui qui l'a réellement saisie : `contacte_par`,
`reponse_par` et `projet_par` sont enregistrés séparément. Si Marc contacte un client
et que Julie lève le projet, Marc garde son contact et Julie obtient le projet.

Classement trié par projets levés, puis réponses, puis contacts. L'administrateur
n'y figure pas (`au_podium = false`).

---

## Synchronisation

- **Au démarrage** : affichage immédiat du dernier état connu (cache local), puis
  rattrapage depuis Supabase.
- **Ensuite** : relecture des seules fiches modifiées toutes les 15 secondes, plus un
  rafraîchissement au retour sur l'onglet. Le point à droite des onglets indique
  l'état : vert = à jour, orange = saisies en attente, rouge = hors connexion.
- **À chaque saisie** : la fiche part dans une file d'envoi. Coupure réseau — fréquent
  sur le terrain — elle est conservée et repart seule au retour de la connexion, même
  après fermeture du navigateur.
- **Pendant qu'on saisit** : la liste ne se redessine jamais sous les doigts.
- **Session** : reste ouverte d'un jour à l'autre, le jeton se renouvelle tout seul.

---

## En cas de problème

La console du navigateur (F12) affiche l'URL appelée et l'erreur exacte.

| Symptôme | Cause |
|---|---|
| « Votre compte n'est rattaché à aucun vendeur » | L'e-mail du compte ne figure pas dans `vendeurs`, ou `resynchroniser_comptes()` n'a pas été lancé |
| Un vendeur ne voit aucun client | Son `vendeur_id` ne correspond à aucune valeur de `clients.commercial` |
| `PGRST125` / *Invalid path* | `SUPABASE_URL` contient un chemin en trop ; attendu : le domaine seul |
| `PGRST205` / *Could not find the table* | `supabase-v2.sql` n'a pas été exécuté |
| `401` / *Invalid API key* | Clé `secret` / `service_role` utilisée au lieu de la clé publique |
| « Réservé aux administrateurs » à l'import | `admin` vaut `false` sur votre ligne de `vendeurs` |

---

## La carte

En ouvrant une fiche client, une carte situe l'exploitation et un bouton
**Itinéraire** passe le relais au GPS du téléphone.

Le fichier ne contient aucune coordonnée : l'adresse est convertie par la
**Base Adresse Nationale** (`api-adresse.data.gouv.fr`), service public gratuit
et sans clé. Le résultat est ensuite gardé sur l'appareil, si bien qu'une fiche
déjà consultée s'affiche sans aucun appel réseau. Le fond de plan vient
d'**OpenStreetMap** via Leaflet, chargé depuis un CDN à la première fiche
ouverte seulement — jamais au démarrage de l'application.

Rien ne casse si un maillon manque : adresse introuvable, réseau coupé ou CDN
injoignable, la fiche reste utilisable et le bouton Itinéraire fonctionne
toujours (il retombe sur l'adresse au lieu des coordonnées). Un échec réseau
n'est pas mémorisé : la carte retentera à la prochaine ouverture.

> **Ce que cela expose** : l'adresse du client est envoyée à la Base Adresse
> Nationale pour être localisée — rien d'autre. Ni le nom du client, ni les
> notes, ni les suivis, ni l'identité du vendeur ne quittent Supabase. Si même
> cela vous gêne, retirez le bloc `.geo` de `cardBody()` : le reste de
> l'application n'en dépend pas.

## Charte graphique

L'interface suit la charte Agri Santerre.

| | |
|---|---|
| Rouge | `#c8102e` — 1ʳᵉ place, projets, actions principales, accents |
| Noir | `#0e0f12` — barre d'application, 2ᵉ place, boutons secondaires |
| Gris | `#5a6068` / `#8a9097` — 3ᵉ place, textes secondaires |
| Gris clair | `#e9ebed` — fonds neutres |
| Papier | `#f2f3f5` — fond de page |

**Typographie** : League Spartan (titres, chiffres, boutons, en majuscules) et
Montserrat (textes courants), chargées depuis Google Fonts.

**Motif** : la barre inclinée à −18°, tirée des sillons du logo, sert de marqueur de
titre, de puce et de témoin de synchronisation. Elle n'est jamais appliquée à un
élément porteur de texte, qui se retrouverait penché.

Le podium reprend les trois couleurs de la charte plutôt que or / argent / bronze,
absents de la palette : **1ᵉʳ rouge, 2ᵉ noir, 3ᵉ gris**. Le rang reste lisible par la
hauteur des marches et le numéro de médaille.

Le logo dans la barre et sur l'écran de connexion est un SVG inline qui reprend le
motif des sillons. Pour utiliser le fichier officiel, remplacez le `<svg>` de la
`<div class="glyph">` par une balise `<img src="logo.svg" alt="Agri Santerre">`
(le fichier doit alors être publié avec les pages).

Toutes les couleurs sont des variables CSS en tête de fichier (`--rouge`, `--noir`,
`--gris`…) : une retouche de palette se fait à un seul endroit.

## Limites connues

- **Notes simultanées** : si deux personnes modifient les notes de la *même* fiche en
  même temps, la dernière enregistrée écrase l'autre.
- **Le podium révèle les scores** de chaque vendeur à toute l'équipe. C'est le principe
  du classement ; pour le rendre anonyme, il faudrait modifier la fonction `podium()`.
- **Cache local** : le portefeuille d'un vendeur est mis en cache dans son navigateur
  pour l'usage hors ligne. Sur un appareil partagé, la déconnexion l'efface.
- **Réattribuer un client** se fait par réimport, pas depuis l'interface.
