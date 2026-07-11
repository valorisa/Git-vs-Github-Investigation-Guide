# Guide d'enquête Git vs GitHub

Un guide pédagogique pour débutants qui explique pourquoi un dépôt GitHub peut afficher une
date de mise à jour récente même si `git log` ne montre aucun commit récent.

![Markdown Lint](https://img.shields.io/badge/markdownlint-passing-brightgreen)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

English version available here: [README.md](README.md)

## Table des matières

- [Pourquoi ce projet existe](#pourquoi-ce-projet-existe)
- [Ce que vous allez apprendre](#ce-que-vous-allez-apprendre)
- [Vocabulaire de base](#vocabulaire-de-base)
- [Quand Git et GitHub ne racontent pas la même histoire](#quand-git-et-github-ne-racontent-pas-la-m%C3%AAme-histoire)
- [Exemple concret](#exemple-concret)
- [Méthode d'enquête : comment arriver à cette conclusion](#m%C3%A9thode-denqu%C3%AAte--comment-arriver-%C3%A0-cette-conclusion)
- [Commandes utiles](#commandes-utiles)
- [Lecture des résultats](#lecture-des-resultats)
- [Erreurs fréquentes](#erreurs-frequentes)
- [Cas particuliers](#cas-particuliers)
- [Parcours recommandé pour débutant](#parcours-recommande-pour-debutant)
- [Structure du dépôt](#structure-du-depot)
- [Ressources complémentaires](#ressources-complementaires)
- [Licence](#licence)
- [Contribution](#contribution)
- [Aide et contact](#aide-et-contact)

## Pourquoi ce projet existe

Il arrive qu'une page de dépôt GitHub affiche "Updated 3 hours ago", alors que la commande
`git log` sur votre clone local montre que le dernier commit date de plusieurs jours. Cela
peut sembler contradictoire pour un débutant, comme si Git et GitHub se contredisaient.

En réalité, ils ne se contredisent pas. Ils rapportent simplement deux choses différentes :

- **Git** rapporte les commits qui existent sur la branche que vous regardez.
- **GitHub** rapporte l'activité du dépôt, ce qui inclut des événements qui ne sont pas des
  commits sur votre branche actuelle, comme une fusion, une pull request, ou la suppression
  d'une branche.

Ce guide détaille la méthode d'enquête, étape par étape, pour qu'un débutant complet puisse
la reproduire sur ses propres dépôts.

## Ce que vous allez apprendre

À la fin de ce guide, vous saurez :

- Lire un résultat de `git log` avec confiance.
- Reconnaître un commit de fusion (merge).
- Comprendre ce qu'est un `DeleteEvent` sur GitHub.
- Utiliser l'outil en ligne de commande `gh` pour lister les pull requests et les événements
  d'un dépôt.
- Interpréter correctement pourquoi l'activité GitHub et l'historique Git local peuvent
  diverger.

## Vocabulaire de base

Ces mots reviennent tout au long du guide. Si vous les connaissez déjà, vous pouvez passer
directement à la suite.

- **Commit** : une sauvegarde de l'état du projet à un instant précis.
- **Branche** : une ligne de travail indépendante, créée à partir de l'historique du projet,
  qui permet de modifier le code sans affecter la ligne de travail principale.
- **Fusion (merge)** : l'action de combiner les changements d'une branche dans une autre.
- **Pull request (PR)** : une proposition de fusionner des changements d'une branche vers
  une autre, revue avant d'être acceptée.
- **DeleteEvent** : un événement GitHub enregistré lorsqu'une branche ou un tag est supprimé.

> **À retenir :** une branche est un espace de travail, un commit est une étape sauvegardée,
> et une pull request est une proposition de fusionner ce travail dans la ligne principale.

## Quand Git et GitHub ne racontent pas la même histoire

`git log` montre uniquement les commits qui font partie de l'historique de la branche
actuellement extraite (checked out). Il ne montre pas :

- les commits réalisés sur des branches que vous n'avez pas récupérées (fetch),
- les branches supprimées après avoir été fusionnées,
- les événements au niveau du dépôt, comme les commentaires d'issues, les releases, ou les
  changements de paramètres.

De son côté, la liste des dépôts GitHub affiche une date générale de "dernière activité".
Cette date peut être mise à jour par une fusion, une suppression de branche, ou d'autres
événements du dépôt, même si aucun nouveau commit n'a été ajouté à la branche que vous
consultez localement.

> **Attention :** une date "Updated" sur GitHub n'est pas la preuve qu'un nouveau commit a
> été poussé. Elle signifie seulement que quelque chose s'est produit dans le dépôt.

## Exemple concret

Voici la situation qui a motivé ce guide, sur le dépôt `ShellFromBrowser` :

- Le commit le plus récent visible dans `git log` datait de 43 heures.
- La liste des dépôts GitHub affichait "Updated 3 hours ago", ce qui ne correspondait pas.
- Une requête sur l'API Events de GitHub a confirmé la cause exacte de cet écart.

Le tableau ci-dessous montre ce que l'API Events de GitHub a renvoyé pour l'événement qui a
mis à jour la date d'activité du dépôt :

| Champ | Valeur |
| :--- | :--- |
| Type d'événement | `DeleteEvent` |
| Date exacte | `2026-07-11T06:18:38Z` |
| Acteur | `valorisa` |
| Ressource supprimée | Branche `valorisa-patch-2` |
| Fichier modifié | Aucun, il s'agit d'une opération Git méta |

Aucun commit et aucune modification de fichier n'expliquent l'étiquette "Updated 3 hours
ago". La suppression d'une branche suffit à elle seule pour rafraîchir la date d'activité
d'un dépôt sur GitHub.

### Pourquoi la branche a été supprimée

La branche `valorisa-patch-2` était la branche source de la pull request #6, qui mettait à
jour Go vers la version 1.26.3. Cette pull request a été fusionnée le 9 juillet. Supprimer
une branche juste après sa fusion est une pratique de nettoyage courante, cohérente avec
l'habitude de ce projet de retirer les branches `feat/*` et `fix/*` une fois leur travail
intégré.

### Recoupement avec d'autres signaux

Trois signaux supplémentaires ont confirmé le diagnostic :

- **Dernier commit** : `2092cbe`, du 9 juillet, correspond au commit de fusion de la pull
  request #6. Aucune activité de commit ne s'est produite dans les 43 heures suivantes.
- **Dernière release** : `v0.4.0`, du 18 mai. Aucune activité de release récente non plus.
- **Pull request #6** : fermée et fusionnée le 9 juillet. Sa branche source,
  `valorisa-patch-2`, a été supprimée le 11 juillet à 06:18 UTC, ce qui correspond
  exactement à la fenêtre "3 hours ago".

L'ensemble confirme que le clone local était parfaitement à jour, et qu'aucune action
n'était requise du côté du développeur.

## Méthode d'enquête : comment arriver à cette conclusion

Le diagnostic ci-dessus ne repose pas sur un seul indice. Il combine trois types de
raisonnement, que vous pouvez réutiliser sur vos propres dépôts.

1. **Repérer une incohérence temporelle.** Un écart de quelques minutes entre `git log` et
   l'étiquette "Updated" de GitHub est normal, dû à la mise en cache ou à la latence
   réseau. Un écart de plusieurs heures, comme les 43 heures observées ici, ne l'est pas.
   Un écart aussi important signifie que l'étiquette ne peut pas s'expliquer par un commit,
   ce qui écarte l'historique des commits comme source et oriente vers les événements au
   niveau du dépôt.
2. **Utiliser ses propres habitudes de travail comme hypothèse.** Si vous savez que vous
   supprimez habituellement vos branches de fonctionnalité après les avoir fusionnées, et
   qu'une pull request a été récemment fusionnée, la suppression de branche devient
   l'explication la plus probable pour une activité sans commit visible.
3. **Savoir ce qui met réellement à jour l'étiquette "Updated".** GitHub rafraîchit la date
   d'activité d'un dépôt pour tout événement sur ses refs, pas seulement les commits. Les
   `DeleteEvent`, `ReleaseEvent`, et même certaines activités sur les issues ou les
   discussions peuvent tous la déclencher. C'est pourquoi `git fetch` et `git log` seuls ne
   peuvent pas répondre à la question : ils ne voient que l'historique des commits, jamais
   ces événements de métadonnées. L'API Events de GitHub est l'outil capable de les voir.

## Commandes utiles

Les commandes ci-dessous peuvent être exécutées depuis Termux, ou tout autre terminal où
`git` et l'outil GitHub CLI (`gh`) sont installés.

```text
# Afficher le graphe complet des commits sur toutes les branches locales
git log --oneline --decorate --graph --all

# Afficher uniquement les commits réalisés après une date et une heure données
git log --since="2026-07-11 00:00" --oneline --all

# Lister les pull requests fusionnées vers la branche main
gh pr list --state merged --base main

# Lister les événements bruts du dépôt, y compris les suppressions de branches
gh api repos/OWNER/REPO/events
```

Remplacez `OWNER/REPO` par le propriétaire et le nom réel du dépôt, par exemple
`valorisa/ShellFromBrowser`.

## Lecture des résultats

- Si `git log --all` montre le commit attendu, l'historique de la branche est complet en
  local.
- Si `gh pr list --state merged` montre une fusion récente que vous ne voyez pas en local,
  exécutez `git fetch` pour mettre à jour votre vue locale des branches distantes.
- Si `gh api repos/OWNER/REPO/events` montre un `DeleteEvent` proche de l'heure où GitHub
  indique que le dépôt a été mis à jour, cela explique l'écart observé.

## Erreurs fréquentes

- Supposer que `git log` (sans `--all`) montre toutes les branches. Par défaut, il ne montre
  que la branche courante.
- Supposer que la mention "Updated" sur GitHub signifie toujours qu'un nouveau commit a été
  poussé.
- Oublier d'exécuter `git fetch` avant de comparer l'historique local et distant.

## Cas particuliers

- **Fusions avec squash** : la pull request est fermée comme fusionnée, mais les commits
  individuels de la branche de fonctionnalité peuvent ne pas apparaître dans `main`, seul un
  commit unique regroupé (squash) apparaît.
- **Force push** : l'historique peut être réécrit, ce qui peut faire disparaître d'anciens
  commits de `git log`, même s'ils restent visibles un certain temps dans l'historique des
  événements GitHub.
- **Branches protégées** : certains paramètres de dépôt peuvent restreindre qui peut pousser
  directement, ce qui modifie la manière dont les mises à jour arrivent habituellement sur
  `main`.

## Parcours recommandé pour débutant

1. Lire la section [Vocabulaire de base](#vocabulaire-de-base) jusqu'à ce que les termes
   soient familiers.
2. Reproduire les commandes de la section [Commandes utiles](#commandes-utiles) sur l'un de
   vos propres dépôts.
3. Comparer vos résultats à l'aide de la liste de vérification de la section
   [Lecture des résultats](#lecture-des-resultats).
4. Relire la section [Erreurs fréquentes](#erreurs-frequentes) pour éviter les
   malentendus les plus courants.
5. Conserver ce guide comme référence la prochaine fois que l'activité d'un dépôt semble
   incohérente.

## Structure du dépôt

```text
repo/
|-- README.md
|-- README.fr.md
|-- LICENSE
|-- .gitignore
`-- .github/
    `-- workflows/
        `-- markdownlint.yml
```

## Ressources complémentaires

- La documentation officielle de GitHub sur la rédaction d'un bon README.
- La documentation officielle de GitHub sur l'activité et les événements d'un dépôt.
- La documentation officielle de l'outil en ligne de commande `gh`.

## Licence

Ce projet est distribué sous licence MIT. Voir le fichier [LICENSE](LICENSE) pour le détail.

## Contribution

Les suggestions et corrections sont les bienvenues. Merci d'ouvrir une issue décrivant le
changement proposé avant de soumettre une pull request.

## Aide et contact

Si une partie de ce guide n'est pas claire, merci d'ouvrir une issue sur ce dépôt en
précisant quelle section pose problème. Cela permet d'améliorer le guide pour le prochain débutant.

