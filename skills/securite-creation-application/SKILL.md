---
name: securite-creation-application
description: Sécurité pour création d'application. Check-list de 28 contrôles de sécurité (authentification, secrets, base de données, HTTPS, XSS, CSRF, CORS, paiements, dépendances, historique Git, skills/plugins/MCP…) à appliquer à toute application web, mobile ou API créée ou modifiée avec l'IA. Utiliser dès qu'on crée, modifie, relit, déploie ou met en ligne une application, un site avec back-end, une API ou un formulaire, avant d'installer une skill, un plugin ou un serveur MCP, et dès que l'utilisateur parle de sécurité, de mise en production, de lancement ou d'audit. Vérifie chaque point dans le code, corrige, puis rend un rapport.
license: MIT
metadata:
  version: "1.1.0"
  source: "Reel Instagram @simplif.ia du 13/11/2026, recoupé avec OWASP Top 10, OWASP ASVS et NVIDIA SkillSpector"
---

# Sécurité pour création d'application

Objectif : qu'aucune application créée avec l'IA ne parte en ligne « grande ouverte ».
L'utilisateur est débutant : explique simplement, en français, sans jargon inutile, et fais le travail à sa place autant que possible.

## Quand l'appliquer

- **Création** d'une application : intègre ces protections dès le départ (c'est moins cher que de corriger après).
- **Modification** : vérifie que le changement ne casse aucun des 28 points qu'il touche.
- **Avant d'installer une skill, un plugin ou un serveur MCP** (dans Claude, Claude Code ou le projet) : applique le contrôle 26 avant l'installation.
- **Avant une mise en ligne** ou sur demande d'**audit** : passe les 28 points en revue complète et rends le rapport.

Si un point ne concerne pas le projet (ex. : pas de paiement, pas d'envoi de fichiers), marque-le « Non applicable » avec une phrase d'explication. Ne le saute pas en silence.

## Règles de travail

1. **Vérifier, pas supposer.** Pour chaque point, ouvre le code, la configuration ou lance un outil. Cite le fichier et la ligne. Un point n'est « OK » que si tu as vu la preuve.
2. **Corriger les failles simples directement**, puis montrer ce qui a changé.
3. **Demander avant** toute action risquée ou irréversible : réécrire l'historique Git, révoquer une clé, changer la base de production, supprimer des données.
4. **Ne jamais afficher un secret** (clé, mot de passe, jeton) en clair dans la conversation ou le rapport : masque-le (`sk-...abcd`).
5. **Tester** les protections quand c'est possible (requête sans connexion sur une route admin, formulaire avec `<script>`, trop de requêtes d'affilée…).

## Les 28 contrôles

### A. Accès et comptes
1. **Vraie authentification** : pas de connexion « maison » fragile ; préférer une solution éprouvée (Auth.js, Supabase Auth, Clerk, Firebase Auth, Django/Laravel auth…). Vérifier chaque route protégée côté serveur.
2. **Mots de passe hachés** avec bcrypt, scrypt ou Argon2 (jamais en clair, jamais MD5/SHA1 seul).
3. **Routes admin verrouillées** : contrôle du rôle côté serveur sur chaque route et chaque API admin, pas seulement un bouton caché dans l'interface.
4. **Cookies de session sécurisés** : `HttpOnly`, `Secure`, `SameSite=Lax` ou `Strict`, durée limitée.
5. **Liens de réinitialisation du mot de passe** : à usage unique, expiration courte (≤ 1 h), jeton aléatoire long.

### B. Secrets et clés
6. **Clés API cachées** : jamais dans le code côté navigateur ni dans le dépôt ; uniquement côté serveur via variables d'environnement.
7. **Variables d'environnement vérifiées** : `.env` présent dans `.gitignore`, un `.env.example` sans valeurs réelles, aucune variable secrète préfixée pour le navigateur (`NEXT_PUBLIC_`, `VITE_`, `REACT_APP_`…).
8. **Historique Git scanné** à la recherche de secrets ayant fuité (outils : `gitleaks`, `trufflehog`, ou `git log -p` ciblé).
9. **Clés compromises révoquées et remplacées** : une clé qui a été dans Git doit être considérée comme volée, même si elle a été supprimée depuis. Guider l'utilisateur pas à pas pour la régénérer chez le fournisseur.

### C. Données et entrées utilisateur
10. **Base de données sécurisée** : accès restreint, mot de passe fort, pas exposée publiquement ; règles d'accès par utilisateur (ex. Row Level Security sur Supabase, règles Firestore).
11. **Requêtes paramétrées** ou ORM : jamais de SQL construit en collant du texte saisi par l'utilisateur.
12. **Validation des formulaires côté serveur** (zod, joi, pydantic…), même si une validation existe déjà côté navigateur.
13. **Protection XSS** : échapper l'affichage, éviter `innerHTML` / `dangerouslySetInnerHTML` / `v-html` avec du contenu utilisateur, assainir le HTML si nécessaire (DOMPurify).
14. **Protection CSRF** sur les actions qui modifient des données (jeton CSRF ou cookies `SameSite` + vérification de l'origine).
15. **Fichiers envoyés limités** : liste blanche de types, taille maximale, renommage, stockage hors du dossier public.
16. **Prix et paiements vérifiés côté serveur** : ne jamais faire confiance au prix envoyé par le navigateur ; recalculer côté serveur et valider les webhooks de paiement (signature Stripe, etc.).

### D. Réseau et configuration
17. **HTTPS imposé** partout, avec redirection du HTTP et en-tête HSTS.
18. **Limite de requêtes** (rate limiting) sur la connexion, l'inscription, la réinitialisation du mot de passe et les API coûteuses.
19. **CORS restreint** aux seuls domaines de l'application (pas de `*` avec des identifiants).
20. **En-têtes de sécurité** : `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `frame-ancestors` / `X-Frame-Options`.
21. **Mode debug désactivé en production** : pas de traces d'erreur détaillées affichées aux visiteurs.
22. **Fichiers exposés recherchés** : `.env`, `.git/`, sauvegardes (`.bak`, `.sql`, `.zip`), fichiers de configuration accessibles publiquement.

### E. Surveillance et maintenance
23. **Journalisation et surveillance** des activités suspectes : échecs de connexion répétés, accès refusés, actions admin. Ne jamais journaliser de mots de passe ou de secrets.
24. **Dépendances vulnérables mises à jour** (`npm audit`, `pip-audit`, `composer audit`, Dependabot…).
25. **Paquets inutilisés supprimés** (moins de code = moins de failles).

### F. Outils d'IA (skills, plugins, serveurs MCP)
26. **Skills, plugins et MCP scannés** : un outil d'IA peut exécuter des commandes, lire des fichiers et envoyer des données. Avant de l'installer, et lors de l'audit d'un projet qui en contient (`.claude/`, `skills/`, `.mcp.json`, hooks, `CLAUDE.md`, `AGENTS.md`) :
    - Scanner avec **NVIDIA SkillSpector** (open source, fonctionne en local, rien n'est envoyé) :
      `uv tool install git+https://github.com/NVIDIA/skillspector.git` puis `skillspector scan <dossier, fichier SKILL.md, zip ou URL GitHub> --no-llm`.
    - Alternative sans installation, **uniquement pour un dépôt GitHub public** : https://skill-spector.com (site tiers non officiel ; les scans d'URL publiques peuvent devenir des rapports publics ; ne jamais y envoyer de code privé ni de secrets).
    - Relire aussi soi-même : commandes `curl … | bash`, téléchargements, lecture de clés ou de `.env`, envois vers des adresses inconnues, consignes qui demandent d'ignorer les règles.
    - Score élevé = **revue manuelle avant installation**, pas une preuve de malveillance ; score faible = pas une garantie (l'outil produit des faux positifs et ne voit pas tout).

### G. Clôture
27. **Audit de sécurité complet** : repasser sur les 26 points précédents après les corrections.
28. **Tester les protections et rendre le rapport** ci-dessous.

## Modèle de rapport

Rends toujours le rapport sous cette forme, en français simple :

```markdown
# Rapport de sécurité — <nom du projet> — <date>

## En bref
- Contrôles OK : X / 28
- Failles corrigées : X
- À faire par toi : X  (actions que je ne peux pas faire seul, ex. régénérer une clé)
- Non applicables : X

## Détail
| # | Contrôle | Statut | Gravité | Preuve (fichier:ligne) | Ce que j'ai fait / ce qu'il reste à faire |
|---|----------|--------|---------|------------------------|-------------------------------------------|
| 1 | Vraie authentification | ✅ OK / 🔧 Corrigé / ⚠️ À faire / ➖ N/A | Critique / Haute / Moyenne / Faible | ... | ... |

## Tes prochaines actions (dans l'ordre)
1. ...
```

Gravité : **Critique** = exploitable tout de suite (secret exposé, route admin ouverte, injection SQL) ; **Haute** = fuite de données probable ; **Moyenne** = protection manquante mais exploitation plus difficile ; **Faible** = bonne pratique.

## Références
- OWASP Top 10 : https://owasp.org/www-project-top-ten/
- OWASP ASVS : https://owasp.org/www-project-application-security-verification-standard/
- OWASP Cheat Sheets : https://cheatsheetseries.owasp.org/
- NVIDIA SkillSpector : https://github.com/NVIDIA/SkillSpector
- Guide NVIDIA « Scan agent skills before installation » : https://docs.nvidia.com/skills/scanning-agent-skills
