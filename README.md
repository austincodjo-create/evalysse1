# EvalysSE — kit de démarrage de l'application réelle (V1)

Ce dossier contient la base de l'application EvalysSE connectée à Supabase. Il suit le cahier des charges V1.

| Élément | Où | État |
| --- | --- | --- |
| Base de données : 19 tables, sécurité par ligne, règles de gestion | `supabase/migrations/` | Prêt, testé |
| Banque d'indicateurs de départ (26 indicateurs, 8 secteurs) | `supabase/seed.sql` | Prêt |
| Tests des règles (isolement, rôles, validation, verrouillage, plan) | `supabase/tests/test_regles.sql` | Tous passent |
| Paiement FedaPay : création du lien, webhook signé, activation | `supabase/functions/creer-paiement`, `webhook-fedapay` | Prêt, à tester en mode sandbox |
| Rappels quotidiens et cycle de l'abonnement | `supabase/functions/rappels` | E-mail prêt ; SMS et WhatsApp à brancher |
| Application web : connexion, inscription, portefeuille, saisie hors ligne et validation, abonnement | `src/` | Prêt, compile |
| Écrans cadre logique, indicateurs, PTA, tableau de bord, rapports, paramètres | à porter depuis le prototype | Étape suivante |

## 1. Les comptes à créer (par vous)

Comptez environ une heure. Gardez les clés dans un gestionnaire de mots de passe et ne les envoyez jamais par WhatsApp ou par e-mail.

1. **GitHub** (github.com) : pour garder le code. Créez un dépôt privé `evalysse`.
2. **Supabase** (supabase.com) : base de données, connexion des utilisateurs, stockage des pièces jointes.
   - Créez un projet nommé `evalysse`. Région conseillée : la plus proche de vos utilisateurs parmi celles proposées (Europe de l'Ouest pour l'Afrique de l'Ouest).
   - Notez dans *Project Settings → API* : l'URL du projet, la clé `anon` (publique) et la clé `service_role` (secrète).
3. **Vercel** (vercel.com) ou **Netlify** : pour mettre l'application en ligne. Connectez-le à votre compte GitHub.
4. **FedaPay** (fedapay.com) : compte marchand pour Mobile Money et carte. Commencez en mode *sandbox* (test). L'activation du mode *live* demande les documents de l'entreprise.
   - Notez la clé secrète `sk_sandbox_…`.
   - Plus tard, dans *Workbench → Webhooks*, vous créerez un webhook et noterez son secret `wh_…`.
5. **Resend** (resend.com) : envoi des e-mails de rappel. Il faut valider votre nom de domaine.
6. **Nom de domaine** : par exemple `evalysse.com` ou `evalysse.africa`, chez le registraire de votre choix.

## 2. Installer la base de données

Sur un ordinateur avec Node.js 20 ou plus :

```bash
npm install -g supabase
supabase login
supabase init                      # une seule fois, dans ce dossier
supabase link --project-ref VOTRE_REF_PROJET
supabase db push                   # applique les 3 migrations
psql "VOTRE_CHAINE_DE_CONNEXION" -f supabase/seed.sql   # banque d'indicateurs
```

Dans Supabase, *Authentication → URL Configuration* : mettez l'adresse de votre site (par exemple `https://app.evalysse.com`) comme *Site URL*.

Dans *Storage*, créez un compartiment privé `justificatifs` pour les pièces jointes. Chaque organisation y aura son propre dossier.

## 3. Installer les fonctions serveur (paiement et rappels)

```bash
supabase secrets set FEDAPAY_SECRET_KEY=sk_sandbox_xxx FEDAPAY_ENV=sandbox \
  FEDAPAY_WEBHOOK_SECRET=wh_xxx APP_URL=https://app.evalysse.com \
  RESEND_API_KEY=re_xxx MAIL_FROM="EvalysSE <rappels@evalysse.com>"
supabase functions deploy creer-paiement
supabase functions deploy webhook-fedapay --no-verify-jwt
supabase functions deploy rappels
```

- Dans FedaPay (*Workbench → Webhooks*), déclarez l'URL `https://VOTRE_REF_PROJET.supabase.co/functions/v1/webhook-fedapay`. Cochez au moins les événements `transaction.approved`, `transaction.declined` et `transaction.canceled`.
- Dans Supabase (*Integrations → Cron*), planifiez un appel quotidien de la fonction `rappels` à 6 h UTC, soit 7 h à Cotonou.

L'abonnement n'est jamais activé par la page de retour du navigateur. Seul le webhook l'active, après avoir vérifié la signature et relu la transaction chez FedaPay (règle RA5). Le montant est toujours lu dans la base, jamais envoyé par le navigateur.

## 4. Lancer et mettre en ligne l'application

```bash
cp .env.example .env.local         # puis remplir l'URL et la clé anon
npm install
npm run dev                        # http://localhost:5173
npm test                           # tests des calculs (taux, feux, exécution)
```

Pour la mise en ligne, importez le dépôt GitHub dans Vercel. Ajoutez les variables `VITE_SUPABASE_URL` et `VITE_SUPABASE_ANON_KEY`, puis déployez. Branchez ensuite votre nom de domaine.

## 5. Tester avant les pilotes

1. Créez un compte, puis une organisation. L'espace reste en lecture seule tant que l'essai n'est pas payé.
2. Payez l'essai en sandbox avec les numéros de test fournis par FedaPay. Vérifiez que le plan passe à « Actif ».
3. Créez un projet, invitez un agent (*Authentication → Users → Invite*), puis ajoutez son rôle dans la table `memberships`. L'écran Paramètres de la prochaine étape le fera depuis l'application.
4. Faites une saisie en mode avion sur un téléphone, puis rétablissez le réseau. Vérifiez qu'elle arrive une seule fois.
5. Validez la saisie, renvoyez-en une autre avec un motif, puis vérifiez le tableau de bord.

Pour rejouer les tests de règles sur une base de test :

```bash
psql "CHAINE_BASE_DE_TEST" -v ON_ERROR_STOP=1 -f supabase/tests/test_regles.sql
```

## 6. Ce qui vient ensuite

- Porter les écrans du prototype : cadre logique, banque d'indicateurs, PTA avec chronogramme, tableau de bord, rapports avec export Word, PDF et Excel, et paramètres (invitations, rôles, seuils).
- Rendre l'application installable sur téléphone (PWA) pour un vrai mode hors ligne au démarrage.
- Brancher l'assistant IA de rédaction des synthèses, avec relecture obligatoire (RG16) et sans noms de bénéficiaires.
- Brancher SMS et WhatsApp dans `supabase/functions/rappels`, avec le fournisseur retenu pendant le pilote.
- Faire relire la sécurité par un développeur expérimenté avant d'accueillir des données réelles de bénéficiaires.
