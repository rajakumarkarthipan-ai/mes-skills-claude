---
name: securite-creation-application
description: Sécurité pour création d'application. Check-list de 28 contrôles de sécurité (authentification, secrets, base de données, HTTPS, XSS, CSRF, CORS, paiements, dépendances, historique Git, skills/plugins/MCP…) à appliquer à toute application web, mobile ou API créée ou modifiée avec l'IA. Utiliser dès qu'on crée, modifie, relit, déploie ou met en ligne une application, un site avec back-end, une API ou un formulaire, et dès que l'utilisateur parle de sécurité, de mise en production, de lancement ou d'audit. Vérifie chaque point dans le code, corrige, puis rend un rapport.
license: MIT
metadata:
  version: "1.7.0"
  source: "Reel Instagram @simplif.ia du 13/11/2026, recoupé avec OWASP Top 10, OWASP ASVS, NVIDIA SkillSpector, Gitleaks et Semgrep"
---

# Sécurité pour création d'application

Objectif : qu'aucune application créée avec l'IA ne parte en ligne « grande ouverte ».
L'utilisateur est débutant : explique simplement, en français, sans jargon inutile, et fais le travail à sa place autant que possible.

## Quand l'appliquer

- **Création** d'une application : intègre ces protections dès le départ (c'est moins cher que de corriger après).
- **Modification** : vérifie que le changement ne casse aucun des 28 points qu'il touche.
- **Avant une mise en ligne** ou sur demande d'**audit** : passe les 28 points en revue complète et rends le rapport.

Si un point ne concerne pas le projet (ex. : pas de paiement, pas d'envoi de fichiers), marque-le « Non applicable » avec une phrase d'explication. Ne le saute pas en silence.

## Règles de travail

1. **Vérifier, pas supposer.** Pour chaque point, ouvre le code, la configuration ou lance un outil. Cite le fichier et la ligne. Un point n'est « OK » que si tu as vu la preuve.
2. **Corriger les failles simples directement**, puis montrer ce qui a changé.
3. **Demander avant** toute action risquée ou irréversible : réécrire l'historique Git, révoquer une clé, changer la base de production, supprimer des données.
4. **Ne jamais afficher un secret** (clé, mot de passe, jeton) en clair dans la conversation ou le rapport : masque-le (`sk-...abcd`).
5. **Tester** les protections quand c'est possible (requête sans connexion sur une route admin, formulaire avec `<script>`, trop de requêtes d'affilée…).

## Premier passage automatique : Semgrep

Au début de chaque audit (et après une grosse modification), lancer **Semgrep Community Edition** (https://semgrep.dev, gratuit, licence LGPL 2.1, analyse le code en local) pour repérer automatiquement les failles courantes. Il accélère la check-list mais ne la remplace pas.

- Installation : `pipx install semgrep` (ou `pip install semgrep`). Sur Windows, Semgrep est en version bêta : si l'installation ou l'analyse échoue, le dire et continuer la check-list à la main.
- Analyse : `semgrep scan --config p/default --config p/owasp-top-ten --metrics off` dans le dossier du projet.
  - `--metrics off` : aucune statistique d'utilisation envoyée. Le code n'est jamais envoyé ; seules les règles sont téléchargées depuis le registre Semgrep.
  - **Pas besoin de compte** : ne pas lancer `semgrep login` ni la plateforme en ligne (payante au-delà de l'offre gratuite) sans accord de l'utilisateur.
- Pour chaque résultat : vérifier dans le code si c'est une vraie faille (il y a des faux positifs), le rattacher au contrôle concerné ci-dessous (ex. injection SQL → 11, XSS → 13, secret en dur → 6), corriger, et indiquer « détecté par Semgrep » dans le rapport.
- Un résultat Semgrep « propre » ne prouve pas que le contrôle est OK : la logique métier (droits d'accès, prix, paiements) se vérifie à la main.

## Les 28 contrôles

### A. Accès et comptes
1. **Vraie authentification** : pas de connexion « maison » fragile ; préférer une solution éprouvée (Auth.js, Supabase Auth, Clerk, Firebase Auth, Django/Laravel auth…). Vérifier chaque route protégée côté serveur.
2. **Mots de passe hachés** avec bcrypt, scrypt ou Argon2 (jamais en clair, jamais MD5/SHA1 seul).
3. **Routes admin verrouillées** : contrôle du rôle côté serveur sur chaque route et chaque API admin, pas seulement un bouton caché dans l'interface.
4. **Cookies de session sécurisés** : `HttpOnly`, `Secure`, `SameSite=Lax` ou `Strict`, durée limitée.
5. **Liens de réinitialisation du mot de passe** : à usage unique, expiration courte (≤ 1 h), jeton aléatoire long.

### B. Secrets et clés
6. **Clés API cachées** : jamais dans le code côté navigateur ni dans le dépôt ; uniquement côté serveur via variables d'environnement.
7. **Variables d'environnement vérifiées** (Gitleaks `dir`, voir contrôle 8, aide à repérer les clés écrites en dur) : `.env` présent dans `.gitignore`, un `.env.example` sans valeurs réelles, aucune variable secrète préfixée pour le navigateur (`NEXT_PUBLIC_`, `VITE_`, `REACT_APP_`…).
8. **Historique Git scanné** à la recherche de secrets ayant fuité, avec **Gitleaks** (https://github.com/gitleaks/gitleaks, open source, gratuit, fonctionne en local, rien n'est envoyé) :
    - Historique complet : `gitleaks git -v --redact` (à lancer dans le dossier du projet).
    - Fichiers actuels, y compris ceux qui ne sont pas dans Git : `gitleaks dir -v --redact`.
    - Toujours utiliser `--redact` pour que les secrets n'apparaissent pas en clair.
    - Installation sur Windows : télécharger le fichier `gitleaks_..._windows_x64.zip` depuis la page officielle des versions (https://github.com/gitleaks/gitleaks/releases) ; ne jamais utiliser un site ou une copie non officielle.
    - Chaque résultat : vérifier si c'est un vrai secret (faux positifs possibles). Un vrai secret trouvé → contrôle 9.
    - Proposer (avec accord) une protection pour l'avenir : un hook `pre-commit` Gitleaks qui bloque tout commit contenant un secret.
    - Si Gitleaks ne peut pas être installé, le dire et faire une recherche manuelle ciblée (`git log -p` avec des motifs comme `api_key`, `secret`, `password`, `sk-`, `BEGIN PRIVATE KEY`).
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
26. **Skills, plugins et MCP du projet vérifiés** : si le projet audité contient des outils d'IA (`.claude/`, `skills/`, `.mcp.json`, hooks, `CLAUDE.md`, `AGENTS.md`), les vérifier en suivant la méthode de la skill **verifier-extension** (scan NVIDIA SkillSpector + relecture des alertes + verdict). Reporter son verdict dans le rapport. L'installation de nouvelles extensions est gérée par verifier-extension, pas par cette skill.

### G. Clôture
27. **Audit de sécurité complet** : repasser sur les 26 points précédents après les corrections.
28. **Tester les protections et rendre le rapport** ci-dessous.
    - **Étape bonus, optionnelle et souvent payante : test d'attaque avec Strix** (https://github.com/usestrix/strix, open source, Apache 2.0). Des agents IA attaquent réellement l'application et prouvent chaque faille. Complémentaire de la check-list (qui vérifie le code) : Strix tente de casser l'application en fonctionnement.
    - **Poser la question une seule fois par audit, ne jamais lancer sans accord explicite.** Expliquer d'abord en une phrase chacun :
      - coût : il faut une **clé API d'un fournisseur d'IA**, payante à l'usage (l'abonnement Claude ne la fournit pas). Option gratuite possible avec un **modèle local** (ex. Ollama, voir https://docs.strix.ai/llm-providers/local), mais il faut un PC puissant et les résultats sont moins bons ;
      - il faut **Docker Desktop** en marche ;
      - le code testé est envoyé au fournisseur d'IA choisi (sauf modèle local) ;
      - **uniquement sur ses propres applications** : attaquer un site sans autorisation écrite est un délit (art. 323-1 du Code pénal).
    - **Si l'utilisateur refuse, ne veut pas payer, ou si Docker / la clé manquent : ne pas faire l'étape, ne pas insister.** Dans le rapport, l'indiquer « ➖ Non fait (étape bonus optionnelle) ». L'audit reste complet et valide sans elle ; ne pas la compter comme un échec.
    - Si l'utilisateur accepte : tester en priorité une **version locale ou de test**, pas la production (sinon vérifier qu'une sauvegarde récente existe). Installation : `pipx install strix-agent` (éviter `curl … | bash`), puis `strix --target <dossier du projet ou URL de test>`. Résultats dans `strix_runs/`. Ajouter les failles confirmées au rapport avec la mention « confirmée par Strix ».
    - Avant d'ajouter les skills de Strix à Claude (`npx skills add usestrix/strix`), utiliser la skill verifier-extension.

## Modèle de rapport

Rends toujours le rapport sous cette forme, en français simple :

```markdown
# Rapport de sécurité — <nom du projet> — <date>

## En bref
- Contrôles OK : X / 28
- Analyse automatique Semgrep : faite / non faite (raison)
- Failles corrigées : X
- À faire par toi : X  (actions que je ne peux pas faire seul, ex. régénérer une clé)
- Non applicables : X
- Étape bonus Strix : faite / non faite (optionnelle)

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
- Gitleaks : https://github.com/gitleaks/gitleaks
- Semgrep Community Edition : https://semgrep.dev/docs/getting-started/quickstart — métriques : https://semgrep.dev/docs/metrics
- Strix (étape bonus) : https://github.com/usestrix/strix — documentation : https://docs.strix.ai
- Guide NVIDIA « Scan agent skills before installation » : https://docs.nvidia.com/skills/scanning-agent-skills
