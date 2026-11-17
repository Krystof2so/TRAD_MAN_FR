# TRAD_FR : pages de manuel traduites en français

Ce dépôt rassemble des traductions françaises de pages de manuel (`man`) de logiciels libres, principalement pour Debian.

L'évolution de dépot dépendra de mes besoins en traduction et des éventuelles contributions. Il n'y a donc pas de ligne directrice.

## Organisation des traductions

```txt
.
├── application1
│   └── fichier_man.1.gz
│   └── fichier_man.5.gz
├── application2
│   ├── fichier_man.1.gz
│   └── fichier-config.5.gz
└── etc...
```

Un dossier par logiciel/application, contenant les pages traduites et **compressées** (`.gz`), prêtes à être installées. Le chiffre avant `.gz` est la section du manuel (`1` = commandes, `5` = formats de fichiers, etc.).

## Installer les traductions

### Une page

Adapter les exemples qui suivent avec les noms des fichiers traduits disponibles.

```zsh
sudo install -Dm644 labwc/labwc-config.5.gz /usr/share/man/fr/man5/labwc-config.5.gz
```

- `-D` crée les répertoires manquants ;
- `-m644` donne les droits `rw-r--r--` ;
- la section (`man5`) doit correspondre au chiffre du nom de fichier.

### Toutes les pages du dépôt

```zsh
for f in */*.gz; do
  nom=$(basename "$f")
  section=${nom%.gz}; section=${section##*.}
  sudo install -Dm644 "$f" "/usr/share/man/fr/man${section}/${nom}"
done
```

### Alternative : liens symboliques

Utile pour que les mises à jour du dépôt (`git pull`) soient prises en compte directement :

```zsh
sudo ln -sf "$PWD/labwc/labwc-config.5.gz" /usr/share/man/fr/man5/labwc-config.5.gz
```

### Vérifier

```zsh
man labwc-config
```

La page s'affiche en français si la locale du système est française (`echo $LANG` → `fr_FR.UTF-8`). Pour forcer le test :

```zsh
LANG=fr_FR.UTF-8 man labwc-config
```

Si besoin, mettre à jour l'index : `sudo mandb`.

## Ajouter sa propre traduction

1. **Copier** la page d'origine dans un répertoire de travail :
   ```zsh
   mkdir -p traductions && cp /usr/share/man/man5/fichier.5.gz traductions/
   ```
   (adapter le chemin et la section ; `man -w fichier` donne l'emplacement exact).
2. **Décompresser** :
   ```zsh
   gzip -d traductions/fichier.5.gz
   ```
   (supprime le `.gz`, ne reste que `fichier.5`).
3. **Traduire** le fichier, en **gardant son nom tel quel**. Il s'agit du format `roff` : ne modifier que le texte, pas les macros (`.TH`, `.SH`, `.B`, etc.). Penser à lancer `man ./traductions/fichier.5` pour contrôler le rendu.
4. **Compresser** :
   ```zsh
   gzip -9nk logiciel/fichier.5
   ```
   (`-9` compression maximale, `-n` sans nom ni date dans l'archive, `-k` conserve le fichier non compressé).
5. **Installer** pour tester (voir plus haut).

## Contribuer

Les contributions sont les bienvenues : nouvelles traductions, corrections, mises à jour de pages existantes.

### Avant de commencer

- Vérifier dans les *issues* et les *pull requests* ouvertes que la page n'est pas déjà en cours de traduction.
- Pour une nouvelle page, ouvrir au préalable une *issue* « Traduction de `xxx.N` » afin d'éviter les doublons.
- Vérifier la licence de la page d'origine (en-tête du fichier, `/usr/share/doc/<paquet>/copyright`) : elle doit permettre la redistribution des traductions.

### Procédure

1. **Forker** le dépôt (bouton *Fork* sur GitHub), puis cloner **son propre fork** :
   ```zsh
   git clone git@github.com:<votre-compte>/TRAD_FR.git
   cd TRAD_FR
   git remote add upstream git@github.com:Krystof2so/TRAD_FR.git
   ```
2. **Créer une branche** à partir de la version à jour du dépôt principal :
   ```zsh
   git fetch upstream
   git switch -c trad-logiciel upstream/main
   ```
3. **Traduire** la page en suivant la section « Ajouter sa propre traduction ».
4. **Contrôler** avant de proposer :
   ```zsh
   gzip -t logiciel/fichier.5.gz        # archive valide
   man ./logiciel/fichier.5.gz          # rendu correct
   ```
5. **Commiter** uniquement les fichiers concernés (éviter `git add .`) :
   ```zsh
   git add logiciel/
   git commit -m "Ajout de la traduction de fichier.5 (logiciel x.y)"
   ```
6. **Pousser vers son fork**, jamais vers le dépôt principal :
   ```zsh
   git push -u origin trad-logiciel
   ```
7. **Ouvrir une *pull request*** vers la branche `main` du dépôt principal, en décrivant la page traduite et la version d'origine.

### Règles/Conventions

- Un répertoire par logiciel et nom de fichier identique à l'original (`fichier.SECTION.gz`).
- Conserver les mentions de copyright et de licence d'origine.
- Une *pull request* par logiciel ou par page, pour faciliter la relecture.
- Ne pas modifier les macros `roff` : seul le texte est traduit.
- Aucun fichier binaire autre que les `.gz` attendus.

### Relecture

Toute contribution est relue avant d'être fusionnée. Des retours ou des demandes de modification peuvent être formulés dans la *pull request*.

## Limites

Ces traductions sont personnelles et peuvent être en retard sur la version de la page d'origine. Une page installée dans `/usr/share/man/fr/` n'est pas gérée par un paquet : si un paquet Debian fournit lui-même une version française, c'est celle-ci qui peut la remplacer lors d'une mise à jour.

## Licence

Chaque traduction est une œuvre dérivée de la page de manuel d'origine et reste placée sous la licence de celle-ci. Les mentions de copyright et de licence d'origine sont conservées dans les fichiers traduits.

Les licences applicables sont indiquées dans le tableau ci-dessous (à vérifier d'après l'en-tête de la page d'origine ou `/usr/share/doc/<paquet>/copyright`).
