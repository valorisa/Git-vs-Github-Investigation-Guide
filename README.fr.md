# Guide d'enquete Git vs GitHub

Un guide pedagogique pour debutants qui explique pourquoi un depot GitHub peut afficher une
date de mise a jour recente meme si `git log` ne montre aucun commit recent.

![Markdown Lint](https://img.shields.io/badge/markdownlint-passing-brightgreen)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

English version available here: [README.md](README.md)

## Table des matieres

- [Pourquoi ce projet existe](#pourquoi-ce-projet-existe)
- [Ce que vous allez apprendre](#ce-que-vous-allez-apprendre)
- [Vocabulaire de base](#vocabulaire-de-base)
- [Quand Git et GitHub ne racontent pas la meme histoire](#quand-git-et-github-ne-racontent-pas-la-meme-histoire)
- [Exemple concret](#exemple-concret)
- [Methode d'enquete : comment arriver a cette conclusion](#methode-denquete--comment-arriver-a-cette-conclusion)
- [Commandes utiles](#commandes-utiles)
- [Lecture des resultats](#lecture-des-resultats)
- [Erreurs frequentes](#erreurs-frequentes)
- [Cas particuliers](#cas-particuliers)
- [Parcours recommande pour debutant](#parcours-recommande-pour-debutant)
- [Structure du depot](#structure-du-depot)
- [Ressources complementaires](#ressources-complementaires)
- [Licence](#licence)
- [Contribution](#contribution)
- [Aide et contact](#aide-et-contact)

## Pourquoi ce projet existe

Il arrive qu'une page de depot GitHub affiche "Updated 3 hours ago", alors que la commande
`git log` sur votre clone local montre que le dernier commit date de plusieurs jours. Cela
peut sembler contradictoire pour un debutant, comme si Git et GitHub se contredisaient.

En realite, ils ne se contredisent pas. Ils rapportent simplement deux choses differentes :

- **Git** rapporte les commits qui existent sur la branche que vous regardez.
- **GitHub** rapporte l'activite du depot, ce qui inclut des evenements qui ne sont pas des
  commits sur votre branche actuelle, comme une fusion, une pull request, ou la suppression
  d'une branche.

Ce guide detaille la methode d'enquete, etape par etape, pour qu'un debutant complet puisse
la reproduire sur ses propres depots.

## Ce que vous allez apprendre

A la fin de ce guide, vous saurez :

- Lire un resultat de `git log` avec confiance.
- Reconnaitre un commit de fusion (merge).
- Comprendre ce qu'est un `DeleteEvent` sur GitHub.
- Utiliser l'outil en ligne de commande `gh` pour lister les pull requests et les evenements
  d'un depot.
- Interpreter correctement pourquoi l'activite GitHub et l'historique Git local peuvent
  diverger.

## Vocabulaire de base

Ces mots reviennent tout au long du guide. Si vous les connaissez deja, vous pouvez passer
directement a la suite.

- **Commit** : une sauvegarde de l'etat du projet a un instant precis.
- **Branche** : une ligne de travail independante, creee a partir de l'historique du projet,
  qui permet de modifier le code sans affecter la ligne de travail principale.
- **Fusion (merge)** : l'action de combiner les changements d'une branche dans une autre.
- **Pull request (PR)** : une proposition de fusionner des changements d'une branche vers
  une autre, revue avant d'etre acceptee.
- **DeleteEvent** : un evenement GitHub enregistre lorsqu'une branche ou un tag est supprime.

> **A retenir :** une branche est un espace de travail, un commit est une etape sauvegardee,
> et une pull request est une proposition de fusionner ce travail dans la ligne principale.

## Quand Git et GitHub ne racontent pas la meme histoire

`git log` montre uniquement les commits qui font partie de l'historique de la branche
actuellement extraite (checked out). Il ne montre pas :

- les commits realises sur des branches que vous n'avez pas recuperees (fetch),
- les branches supprimees apres avoir ete fusionnees,
- les evenements au niveau du depot, comme les commentaires d'issues, les releases, ou les
  changements de parametres.

De son cote, la liste des depots GitHub affiche une date generale de "derniere activite".
Cette date peut etre mise a jour par une fusion, une suppression de branche, ou d'autres
evenements du depot, meme si aucun nouveau commit n'a ete ajoute a la branche que vous
consultez localement.

> **Attention :** une date "Updated" sur GitHub n'est pas la preuve qu'un nouveau commit a
> ete pousse. Elle signifie seulement que quelque chose s'est produit dans le depot.

## Exemple concret

Voici la situation qui a motive ce guide, sur le depot `ShellFromBrowser` :

- Le commit le plus recent visible dans `git log` datait de 43 heures.
- La liste des depots GitHub affichait "Updated 3 hours ago", ce qui ne correspondait pas.
- Une requete sur l'API Events de GitHub a confirme la cause exacte de cet ecart.

Le tableau ci-dessous montre ce que l'API Events de GitHub a renvoye pour l'evenement qui a
mis a jour la date d'activite du depot :

| Champ | Valeur |
| :--- | :--- |
| Type d'evenement | `DeleteEvent` |
| Date exacte | `2026-07-11T06:18:38Z` |
| Acteur | `valorisa` |
| Ressource supprimee | Branche `valorisa-patch-2` |
| Fichier modifie | Aucun, il s'agit d'une operation Git meta |

Aucun commit et aucune modification de fichier n'expliquent l'etiquette "Updated 3 hours
ago". La suppression d'une branche suffit a elle seule pour rafraichir la date d'activite
d'un depot sur GitHub.

### Pourquoi la branche a ete supprimee

La branche `valorisa-patch-2` etait la branche source de la pull request #6, qui mettait a
jour Go vers la version 1.26.3. Cette pull request a ete fusionnee le 9 juillet. Supprimer
une branche juste apres sa fusion est une pratique de nettoyage courante, coherente avec
l'habitude de ce projet de retirer les branches `feat/*` et `fix/*` une fois leur travail
integre.

### Recoupement avec d'autres signaux

Trois signaux supplementaires ont confirme le diagnostic :

- **Dernier commit** : `2092cbe`, du 9 juillet, correspond au commit de fusion de la pull
  request #6. Aucune activite de commit ne s'est produite dans les 43 heures suivantes.
- **Derniere release** : `v0.4.0`, du 18 mai. Aucune activite de release recente non plus.
- **Pull request #6** : fermee et fusionnee le 9 juillet. Sa branche source,
  `valorisa-patch-2`, a ete supprimee le 11 juillet a 06:18 UTC, ce qui correspond
  exactement a la fenetre "3 hours ago".

L'ensemble confirme que le clone local etait parfaitement a jour, et qu'aucune action
n'etait requise du cote du developpeur.

## Methode d'enquete : comment arriver a cette conclusion

Le diagnostic ci-dessus ne repose pas sur un seul indice. Il combine trois types de
raisonnement, que vous pouvez reutiliser sur vos propres depots.

1. **Reperer une incoherence temporelle.** Un ecart de quelques minutes entre `git log` et
   l'etiquette "Updated" de GitHub est normal, du a la mise en cache ou a la latence
   reseau. Un ecart de plusieurs heures, comme les 43 heures observees ici, ne l'est pas.
   Un ecart aussi important signifie que l'etiquette ne peut pas s'expliquer par un commit,
   ce qui ecarte l'historique des commits comme source et oriente vers les evenements au
   niveau du depot.
2. **Utiliser ses propres habitudes de travail comme hypothese.** Si vous savez que vous
   supprimez habituellement vos branches de fonctionnalite apres les avoir fusionnees, et
   qu'une pull request a ete recemment fusionnee, la suppression de branche devient
   l'explication la plus probable pour une activite sans commit visible.
3. **Savoir ce qui met reellement a jour l'etiquette "Updated".** GitHub rafraichit la date
   d'activite d'un depot pour tout evenement sur ses refs, pas seulement les commits. Les
   `DeleteEvent`, `ReleaseEvent`, et meme certaines activites sur les issues ou les
   discussions peuvent tous la declencher. C'est pourquoi `git fetch` et `git log` seuls ne
   peuvent pas repondre a la question : ils ne voient que l'historique des commits, jamais
   ces evenements de metadonnees. L'API Events de GitHub est l'outil capable de les voir.

## Commandes utiles

Les commandes ci-dessous peuvent etre executees depuis Termux, ou tout autre terminal ou
`git` et l'outil GitHub CLI (`gh`) sont installes.

```text
# Afficher le graphe complet des commits sur toutes les branches locales
git log --oneline --decorate --graph --all

# Afficher uniquement les commits realises apres une date et une heure donnees
git log --since="2026-07-11 00:00" --oneline --all

# Lister les pull requests fusionnees vers la branche main
gh pr list --state merged --base main

# Lister les evenements bruts du depot, y compris les suppressions de branches
gh api repos/OWNER/REPO/events
```

Remplacez `OWNER/REPO` par le proprietaire et le nom reel du depot, par exemple
`valorisa/ShellFromBrowser`.

## Lecture des resultats

- Si `git log --all` montre le commit attendu, l'historique de la branche est complet en
  local.
- Si `gh pr list --state merged` montre une fusion recente que vous ne voyez pas en local,
  executez `git fetch` pour mettre a jour votre vue locale des branches distantes.
- Si `gh api repos/OWNER/REPO/events` montre un `DeleteEvent` proche de l'heure ou GitHub
  indique que le depot a ete mis a jour, cela explique l'ecart observe.

## Erreurs frequentes

- Supposer que `git log` (sans `--all`) montre toutes les branches. Par defaut, il ne montre
  que la branche courante.
- Supposer que la mention "Updated" sur GitHub signifie toujours qu'un nouveau commit a ete
  pousse.
- Oublier d'executer `git fetch` avant de comparer l'historique local et distant.

## Cas particuliers

- **Fusions avec squash** : la pull request est fermee comme fusionnee, mais les commits
  individuels de la branche de fonctionnalite peuvent ne pas apparaitre dans `main`, seul un
  commit unique regroupe (squash) apparait.
- **Force push** : l'historique peut etre reecrit, ce qui peut faire disparaitre d'anciens
  commits de `git log`, meme s'ils restent visibles un certain temps dans l'historique des
  evenements GitHub.
- **Branches protegees** : certains parametres de depot peuvent restreindre qui peut pousser
  directement, ce qui modifie la maniere dont les mises a jour arrivent habituellement sur
  `main`.

## Parcours recommande pour debutant

1. Lire la section [Vocabulaire de base](#vocabulaire-de-base) jusqu'a ce que les termes
   soient familiers.
2. Reproduire les commandes de la section [Commandes utiles](#commandes-utiles) sur l'un de
   vos propres depots.
3. Comparer vos resultats a l'aide de la liste de verification de la section
   [Lecture des resultats](#lecture-des-resultats).
4. Relire la section [Erreurs frequentes](#erreurs-frequentes) pour eviter les
   malentendus les plus courants.
5. Conserver ce guide comme reference la prochaine fois que l'activite d'un depot semble
   incoherente.

## Structure du depot

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

## Ressources complementaires

- La documentation officielle de GitHub sur la redaction d'un bon README.
- La documentation officielle de GitHub sur l'activite et les evenements d'un depot.
- La documentation officielle de l'outil en ligne de commande `gh`.

## Licence

Ce projet est distribue sous licence MIT. Voir le fichier [LICENSE](LICENSE) pour le detail.

## Contribution

Les suggestions et corrections sont les bienvenues. Merci d'ouvrir une issue decrivant le
changement propose avant de soumettre une pull request.

## Aide et contact

Si une partie de ce guide n'est pas claire, merci d'ouvrir une issue sur ce depot en
precisant quelle section pose probleme. Cela permet d'ameliorer le guide pour le prochain
debutant.
