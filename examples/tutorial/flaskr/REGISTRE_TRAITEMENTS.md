# Registre des activités de traitement — Application Flaskr (Blog avec authentification)

**Version** : 1.0
**Date** : 2026-11-02
**Référence** : CP-RGPD-30

---

## 1. Responsable du traitement

- **Nom / Entité** : Démonstration Flask (projet tutoriel)
- **Représentant** : Non applicable (code d'exemple)
- **Délégué à la protection des données (DPO)** : Non désigné (démonstration)
- **Coordonnées** : N/A

---

## 2. Finalités du traitement

| Finalité | Base juridique (Art. 6 RGPD) | Description |
|---|---|---|
| Gestion des comptes utilisateurs (inscription, connexion, déconnexion) | Intérêt légitime / Exécution d'un contrat (conditions d'utilisation du service démo) | Permettre aux utilisateurs de créer un compte, s'authentifier et publier des articles sur le blog. |
| Publication et affichage d'articles de blog | Intérêt légitime | Permettre aux auteurs authentifiés de créer, modifier, supprimer leurs articles et les rendre visibles. |
| Journalisation technique (horodatage des articles) | Intérêt légitime | Horodatage automatique (`CURRENT_TIMESTAMP`) à la création de chaque article pour l'affichage chronologique. |

---

## 3. Catégories de personnes concernées

| Catégorie | Description |
|---|---|
| Utilisateurs inscrits | Personnes ayant créé un compte via le formulaire `/auth/register`. |
| Auteurs d'articles | Sous-ensemble des utilisateurs inscrits ayant publié au moins un article. |
| Visiteurs (lecteurs) | Personnes consultant le blog sans compte (données non collectées hors logs serveur standards). |

---

## 4. Catégories de données à caractère personnel

| Catégorie | Données | Table / Source | Caractère obligatoire |
|---|---|---|---|
| Identifiant de compte | `username` (texte, unique) | Table `user.username` | Oui |
| Authentification | `password` (hash bcrypt/scrypt via `werkzeug.security.generate_password_hash`) | Table `user.password` | Oui |
| Identifiant technique | `id` (entier, clé primaire auto-incrément) | Table `user.id`, `post.author_id` | Oui (interne) |
| Contenu éditorial | `title` (texte), `body` (texte) | Table `post.title`, `post.body` | Oui (pour publication) |
| Horodatage | `created` (TIMESTAMP, défaut `CURRENT_TIMESTAMP`) | Table `post.created` | Oui (automatique) |
| Session | `user_id` stocké dans la session Flask (cookie signé) | `session["user_id"]` | Oui (pendant session) |

**Note** : Aucun traitement de données sensibles (Art. 9), de données judiciaires (Art. 10), ni de données de mineurs n'est effectué.

---

## 5. Catégories de destinataires

| Destinataire | Type | Justification |
|---|---|---|
| Application Flaskr (interne) | Destinataire interne | Accès en lecture/écriture à la base SQLite pour l'exécution des fonctionnalités. |
| Administrateur de l'instance (si déployé) | Destinataire interne | Accès direct au fichier `flaskr.sqlite` pour maintenance. |
| **Aucun tiers** | — | Le code ne prévoit aucune transmission à des tiers (pas d'API externe, pas d'envoi d'email, pas d'analytics). |

---

## 6. Transferts vers pays tiers ou organisations internationales

**Aucun transfert** n'est prévu par l'application. La base de données SQLite est stockée localement dans le dossier d'instance (`instance/flaskr.sqlite`).

---

## 7. Durées de conservation / Délais d'effacement

| Données | Durée de conservation | Mécanisme de suppression |
|---|---|---|
| Compte utilisateur (`user` table) | **Indéfinie** (pas de suppression automatique) | Aucune dans le code : pas de route `/delete-account`, pas de tâche cron, pas de politique de rétention. |
| Articles (`post` table) | **Indéfinie** (pas de suppression automatique) | Suppression possible uniquement par l'auteur via l'interface (`/delete/<id>`), mais pas de purge automatique. |
| Sessions | Durée de session navigateur (cookie de session Flask, expire à la fermeture du navigateur sauf `PERMANENT_SESSION_LIFETIME` non configuré) | `session.clear()` à la déconnexion (`/auth/logout`). |

**Recommandation** : Implémenter une politique de rétention (ex. suppression comptes inactifs > 2 ans, anonymisation articles sur demande) et une route de suppression de compte (droit à l'effacement Art. 17).

---

## 8. Mesures de sécurité techniques et organisationnelles (Art. 32 RGPD)

| Mesure | Statut | Détail / Preuve technique |
|---|---|---|
| Hachage des mots de passe | ✅ Implémenté | `generate_password_hash()` / `check_password_hash()` (werkzeug, défaut pbkdf2:sha256). |
| Protection des sessions | ✅ Implémenté | Sessions Flask signées (`SECRET_KEY`), `session.clear()` à la déconnexion. |
| Requêtes SQL paramétrées | ✅ Implémenté | Utilisation de `?` placeholders dans `db.execute()` (protection injection SQL). |
| Clé secrète (`SECRET_KEY`) | ⚠️ Faible en démo | Valeur par défaut `"dev"` dans `create_app()`, doit être surchargée en production via `config.py`. |
| Protection CSRF | ❌ Absente | Aucune protection CSRF visible (pas de `flask-wtf` / `CSRFProtect`, pas de jetons dans formulaires). |
| HTTPS / TLS | ❌ Non géré par l'app | Dépend du déploiement (reverse proxy). Aucune contrainte `SESSION_COOKIE_SECURE=True` visible. |
| Journalisation des accès / audit | ❌ Absente | Pas de logs d'événements sécurité (connexions échouées, modifications données). |
| Chiffrement base de données | ❌ Absent | Fichier SQLite en clair sur disque. |
| Minimisation des données | ✅ Respectée | Seuls username, password hash, titre, corps, timestamp collectés. |

---

## 9. Cohérence avec les preuves techniques (code source)

| Élément du registre | Preuve technique (fichier : lignes) |
|---|---|
| Table `user` (id, username, password) | `schema.sql` : L10-L14 |
| Table `post` (id, author_id, created, title, body) | `schema.sql` : L16-L21 |
| Inscription : username + password hashé | `auth.py` : L45-L55 (`generate_password_hash`) |
| Connexion : vérification hash + session `user_id` | `auth.py` : L68-L85 (`check_password_hash`, `session["user_id"]`) |
| Chargement utilisateur via session | `auth.py` : L28-L36 (`load_logged_in_user`, `SELECT * FROM user WHERE id = ?`) |
| Création article avec auteur + timestamp auto | `blog.py` (non lu, inféré via FK `author_id` + `DEFAULT CURRENT_TIMESTAMP`) |
| Suppression article par auteur | `blog.py` (route `/delete/<id>` attendue) |
| Clé secrète par défaut `"dev"` | `__init__.py` : L10 |
| Absence CSRF | Aucun import `flask_wtf.csrf` ou `CSRFProtect` dans le code lu |

---

## 10. Actions correctives recommandées (pour mise en conformité)

1. **CSRF** : Ajouter `flask-wtf` et `CSRFProtect(app)` ; inclure `{{ csrf_token() }}` dans tous les formulaires (`register.html`, `login.html`, `create.html`, `update.html`).
2. **SECRET_KEY** : Documenter l'obligation de surcharger via `instance/config.py` avec une valeur aléatoire forte.
3. **HTTPS** : Ajouter `SESSION_COOKIE_SECURE=True`, `SESSION_COOKIE_HTTPONLY=True`, `SESSION_COOKIE_SAMESITE='Lax'` en config production.
4. **Droit à l'effacement** : Implémenter route `/account/delete` supprimant l'utilisateur et ses articles (ou les anonymisant).
5. **Politique de rétention** : Définir et documenter des durées maximales ; ajouter script de purge / anonymisation.
6. **Journalisation sécurité** : Logger les échecs de connexion, créations/suppressions de comptes/articles.
7. **Chiffrement au repos** : Évaluer SQLite chiffré (SQLCipher) ou chiffrement disque pour déploiements sensibles.
8. **Registre à jour** : Réviser ce document à chaque évolution fonctionnelle.

---

*Document généré pour satisfaire l'obligation CP-RGPD-30 (Article 30 RGPD) sur la base du code source de l'application Flaskr (tutoriel Flask officiel).*