# 📊 Récapitulatif des données — Réunion du Lundi

Doc de référence : pour chaque donnée affichée, d'où elle vient et quelles règles s'appliquent.
Deux grandes familles : **(A)** données lues dans le Google Sheet (lecture seule, rafraîchies) et **(B)** données saisies à la main dans l'outil (partagées en temps réel).

---

## A. Données du Google Sheet (lecture seule)

- **Source** : un seul classeur Google publié en CSV, 5 onglets (`gid` différents).
- **Rythme** : rechargées **toutes les 5 min** + au chargement de la page.
- **Sécurité** : chaque CSV est validé avant d'être appliqué (`_csvOk`). S'il est vide, c'est une page d'erreur HTML, ou l'en-tête attendu manque → **on garde les dernières bonnes valeurs** (statut « Données en cache »). **Jamais de passage par zéro.**
- ⚠️ Google met le CSV en cache : une modif faite dans le Sheet peut mettre quelques minutes de plus à apparaître (délai côté Google, pas côté outil).

| Donnée | Onglet (gid) | Mot-clé de validation |
|---|---|---|
| Résumé consultants | `1583361967` | `Consultant` |
| Entretiens (ITW) | `1643040749` | `User ID` |
| Jobs ouverts | `584601804` | `User ID` |
| Matrice activité | `438027768` | `PRESC` |
| Calls prospect | `91361805` | `Call Type` |

### A.1 — Résumé consultants (`summary`)
En-tête détecté par le mot `Consultant`. Une ligne par consultant, colonnes **fixes** :

| Col | Champ | Usage |
|---|---|---|
| 0 | Nom | nom affiché |
| 1 | User ID | clé d'identification |
| 2 | nbJob | nb jobs |
| 5 | nbProcess | nb process |
| 6 | ulScore | score UL |
| 7 | firstItw | 1ères ITW |
| 8 | meeting | meetings |
| 9 | callNA | calls NA |
| 10 | callAb | calls aboutis |
| 11 | deals | deals |
| 12 | ca | CA mensuel |
| 13 | sem | semaine |

**Règles** : ligne ignorée si User ID ou Nom vide. **Boris (UID 48693) est exclu.** Si pas d'en-tête `Consultant` → rien n'est appliqué.

### A.2 — Entretiens / ITW (`itw`)
Une ligne par entretien. Colonnes : 0 User ID, 3 client, 4 slug, 5 job, 6 candidat, 7 stage, 8 date, 9 semaine.
⚠️ dans le tableau, un ⚠️ signale un process dont le dernier entretien date de **+ de 3 semaines** (à fermer potentiellement).

### A.3 — Jobs ouverts (`jobs`)
Une ligne par job. Colonnes : 0 User ID, 3 client, 4 slug, 5 intitulé du job. Ligne ignorée si User ID vide.

### A.4 — Matrice activité → tuiles « ACTIVITÉ & RÉSULTATS » (`matrix`)
Structure : ligne 4 = numéros de **semaine**, ligne 5 = nom de la **métrique**. Les consultants sont les lignes dont la colonne 1 (UID) est numérique.
Métriques lues : `PRESC, CVSENT, 1STITW, OTHITW, PRISEREF, PROSPVIS, PROSPPHY, ACCMANA, TNCS, ULSCORE, JOBTOT, JOBHOT`.
Tuiles affichées : Prescreen, CV Sent, 1ères ITW, Autres ITW, Meetings (= PROSPVIS + PROSPPHY + ACCMANA), Calls Prospect.

**Règles clés** :
- 🗓️ **Toujours la semaine PRÉCÉDENTE** (semaine ISO du jour − 1), jamais la semaine en cours. Repli sur la plus récente disponible en début d'année.
- 🛡️ La matrice est reconstruite dans un buffer et **n'est échangée que si elle est complète** (assez de consultants) → empêche tout flicker à zéro sur un CSV partiel.

### A.5 — Calls prospect (`calls`)
Colonnes : 0 date, 2 nom, 4 type, 7 semaine.
**Règles** : on compte **uniquement** `5 - Prospect Call` (les vrais), on **exclut** `4 - Prospect Call NA`. Même **semaine précédente** que la matrice. Rapprochement par **nom** (User ID absent de cet onglet).

---

## B. Données saisies à la main (partagées en temps réel)

- **Stockage** : Firebase Firestore, collection `state` (un document par clé).
- **Rythme** : **temps réel** (instantané). Une saisie chez l'un apparaît aussitôt chez les autres, sans recharger.
- **Clés synchronisées** (préfixes) : `obj_`, `plan_`, `planitems_`, `plannext_`, `planpast_`, `fcastentries_`, `note_`, `team_`, `pot_decisions`.
- **NON synchronisés** (caches dérivés du Sheet, locaux à chaque navigateur) : `ulhist_`, `wstat_`.

| Donnée | Clé | Contenu |
|---|---|---|
| Objectifs mensuels | `obj_<uid>` | CA cible, deals cible |
| Plan d'action (semaine passée) | `planpast_<uid>` / `planitems_` | engagements de la semaine écoulée |
| Plan d'action (semaine prochaine) | `plannext_<uid>` | engagements S+1 |
| Potentiels / forecast | `fcastentries_<uid>` | liste { candidat, company, ca, pct } |
| Décisions potentiels | `pot_decisions` | go / nogo par ligne |
| Notes | `note_<uid>` | notes libres |
| Équipe | `team_…` | composition |

**Règles clés** :
- ✍️ Seules les **vraies saisies utilisateur** sont poussées au cloud. Les écritures **automatiques** (auto-remplissage / seed depuis le pipeline) restent **locales** → elles ne peuvent pas écraser les données partagées des autres.
- Écriture par clé indépendante (un doc par champ) → deux personnes qui éditent deux consultants différents ne se gênent pas.

---

## C. Sauvegardes & récupération

- 💾 **Sauvegarde automatique quotidienne** : le 1er qui ouvre l'outil dans la journée fige un **instantané complet** de l'état partagé (collection `backups`, un doc par date).
- ♻️ **Restauration** : ouvrir `…/monday-meeting/?restore=AAAA-MM-JJ` → remet l'état de la sauvegarde de ce jour (confirmation + rechargement auto).
- ⛑️ **Récupération d'urgence** : `…/monday-meeting/?push=1` pousse le `localStorage` du navigateur courant vers le cloud (utile si un poste a des données qu'un autre n'a pas). ⚠️ À utiliser sciemment : ça fait foi sur le cloud.

---

## ⚠️ Distinction importante : AllBoard vs Monday Meeting (notion de semaine)

Les deux écrans sont **indépendants** et n'ont **pas** la même règle de semaine :

| Écran | Fichier | « Semaine » affichée | ITW |
|---|---|---|---|
| **AllBoard** (Production / UL Score) | `index.html` | **semaine EN COURS** (lue du Sheet, col « UL SCORE SEMAINE ») | 1st ITW + Total ITW = **mois en cours, inclut la semaine en cours** |
| **Monday Meeting** (Activité & résultats) | `monday-meeting.html` | **semaine PRÉCÉDENTE** (ISO − 1) | ITW de la semaine précédente |

- L'AllBoard charge ses propres sources (`SHEET_CSV_URL` gid=2060399592 + onglet ITW gid=1643040749) et **ne passe pas** par `MATRIX` / `buildAllboard` / `_targetWeek`.
- Le « Total ITW » de l'AllBoard est recalculé à partir des dates d'entretien : toutes les ITW dont la date tombe dans le **mois calendaire en cours**.
- ✅ C'est voulu : l'AllBoard est un tableau de bord temps réel, le Monday Meeting passe en revue la semaine écoulée.

---

## D. Identification des consultants

- Clé technique = **User ID** partout, **sauf** l'onglet Calls (rapprochement par **nom** normalisé).
- **Boris DC (UID 48693)** est exclu de l'affichage consultants.
