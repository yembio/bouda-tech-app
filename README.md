# BOUDA-TECH — Application web (Lot 1 + 2 + 3 : fondations, pages front, admin & e-mails)

Ce projet est le vrai site (front + back + base de données), qui reprend
l'identité visuelle et les écrans déjà validés dans les maquettes. C'est un
vrai projet **Next.js** (React + API + base de données dans un seul projet,
plus simple à héberger qu'une architecture séparée).

## Ce qui est fonctionnel maintenant

- Base de données complète (Prisma / PostgreSQL) : utilisateurs, produits,
  commandes, rendez-vous, formations, paiements, factures, e-mails envoyés,
  messages de contact
- Inscription / connexion client (`/inscription`, `/connexion`)
- Catalogue branché en direct sur la base (`/catalogue`)
- Panier persistant (`/panier`), tunnel d'achat complet (`/checkout`) avec
  paiement Mobile Money **sandbox**, confirmation avec facture réelle
- Prise de rendez-vous (`/rdv`) et confirmation
- Formations : liste, fiche détaillée, inscription avec gestion des places
- Espace client (`/compte`) : commandes, RDV, formations du client connecté
- **Pages À propos, Contact (formulaire fonctionnel) et Blog** (`/a-propos`,
  `/contact`, `/blog`)
- **Navigation commune** à toutes les pages publiques (`src/components/Navbar.tsx`)
- **E-mails automatiques** (mode sandbox par défaut, journalisés en base) :
  confirmation de commande, confirmation de RDV, confirmation d'inscription
  formation, rappel de RDV, rappel de formation, demande d'avis en fin de
  formation, notification de message de contact
- **Interface d'administration** (`/admin`, réservée au rôle `ADMIN`) :
  tableau de bord, gestion des produits, des commandes (changement de
  statut), des formations, et journal des e-mails envoyés

## Compte administrateur de test

Le script de seed crée automatiquement un compte admin :

```
Email : admin@bouda-tech.bf
Mot de passe : admin1234
```

⚠️ À changer avant toute mise en production (voir la table `User` dans la
base, ou crée un nouveau compte admin via Prisma Studio : `npx prisma studio`).

## Ce qu'il reste à faire (lots suivants)

- Vraie intégration Orange Money / Moov Money / Wave, une fois le compte
  marchand obtenu (voir `src/lib/mobileMoney.ts`)
- Vrai envoi SMTP/SMS, une fois les identifiants obtenus (voir
  `src/lib/mail.ts` et `src/lib/sms.ts`)
- Table `Article` en base pour un vrai blog administrable (actuellement le
  contenu de `/blog` est statique)
- Upload de vraies images produits (actuellement les cartes utilisent des
  pictogrammes)

## Démarrer en local

Prérequis : Node.js 20+, et une base PostgreSQL (locale ou gratuite sur
[Neon](https://neon.tech) ou [Supabase](https://supabase.com) le temps des tests).

```bash
cp .env.example .env
# renseigner DATABASE_URL dans .env

npm install
npx prisma migrate dev --name init
npm run seed
npm run dev
```

Le site est alors disponible sur http://localhost:3000, avec le catalogue et
un compte admin déjà prêts grâce au seed.

## Comment fonctionne le paiement en mode sandbox

1. Le client choisit un opérateur et initie le paiement → `/api/payments/initiate`
2. Un enregistrement `Payment` est créé avec le statut `PENDING`
3. Le front simule la confirmation en appelant `/api/payments/webhook` après
   un court délai — en production, c'est l'opérateur qui appelle cette URL
4. Dès que le paiement passe à `SUCCESS`, la commande est confirmée, la
   facture générée automatiquement, et l'e-mail de confirmation envoyé

## Comment fonctionnent les e-mails automatiques

En mode sandbox (par défaut), rien n'est réellement envoyé : chaque e-mail
est affiché dans les logs du serveur **et** journalisé dans la table
`EmailLog`, consultable depuis `/admin/emails`.

Pour passer à l'envoi réel :
1. Choisis un fournisseur SMTP (Brevo, Zoho, ou tout autre — Brevo est bien
   implanté en Afrique de l'Ouest et propose un plan gratuit)
2. Renseigne `SMTP_HOST`, `SMTP_USER`, `SMTP_PASS` dans `.env`
3. Installe `nodemailer` (`npm install nodemailer`) et complète la fonction
   `sendViaSmtp()` dans `src/lib/mail.ts`
4. Passe `EMAIL_MODE=live`

## Planifier les rappels automatiques (RDV, formations, avis)

La route `/api/cron/daily-tasks` doit être appelée une fois par jour pour
déclencher les rappels de RDV, rappels de formation et demandes d'avis.
Elle est protégée par un secret :

```bash
curl -X POST https://ton-domaine/api/cron/daily-tasks \
     -H "Authorization: Bearer $CRON_SECRET"
```

Sur Render, cela se configure via **New > Cron Job**, avec cette commande et
un horaire quotidien (par exemple 7h du matin). GitHub Actions avec un
`schedule` fonctionne aussi très bien pour ce genre de tâche simple.

## Passer aux vrais paiements Mobile Money

Une fois que tu as un compte marchand actif chez Orange Money, Moov Money ou
Wave Burkina Faso :

1. Renseigne les clés API fournies par l'opérateur dans `.env`
   (`ORANGE_MONEY_MERCHANT_KEY`, etc.)
2. Passe `PAYMENT_MODE=live`
3. Complète les fonctions correspondantes dans `src/lib/mobileMoney.ts` avec
   la documentation technique que l'opérateur te fournira à l'activation du
   compte (le squelette et les commentaires expliquant le flux général sont
   déjà en place)

Aucune autre partie du code n'a besoin de changer : la logique métier est
indépendante de l'opérateur utilisé.

## Déployer sur Render (recommandé pour démarrer)

Render permet d'héberger le site **et** la base de données PostgreSQL au même
endroit, sans configuration serveur à gérer — bien adapté pour une PME.

1. Créer un compte sur [render.com](https://render.com)
2. **New > PostgreSQL** — créer une base, copier l'URL de connexion fournie
3. **New > Web Service** — connecter ce projet (dépôt Git), Render détecte
   Next.js automatiquement
4. Dans les variables d'environnement du service, coller le contenu de ton
   `.env` (avec l'URL de la base créée à l'étape 2)
5. (Optionnel) **New > Cron Job** pour les rappels automatiques (voir
   ci-dessus)
6. Render construit et déploie automatiquement ; le site est accessible sur
   une URL `*.onrender.com`, à laquelle tu pourras ensuite rattacher ton
   propre nom de domaine

## Structure du projet

```
prisma/schema.prisma        schéma de la base de données
prisma/seed.ts               données de test (produits, formation, admin)
src/lib/prisma.ts            connexion base de données
src/lib/auth.ts               authentification (JWT)
src/lib/requireAdmin.ts       contrôle d'accès admin
src/lib/mobileMoney.ts        couche de paiement (sandbox + réel)
src/lib/mail.ts                couche e-mail + tous les gabarits
src/lib/sms.ts                 couche SMS (sandbox + Africa's Talking)
src/components/Navbar.tsx      navigation commune aux pages publiques
src/app/api/                   routes back-end (public + /api/admin protégé)
src/app/admin/                 interface d'administration
src/app/(pages publiques)/     catalogue, panier, checkout, rdv, formations,
                                compte, à propos, contact, blog
```
