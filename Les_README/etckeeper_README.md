etckeeper est un ensemble d'outils permettant de stocker `/etc` dans un dépôt git,
mercurial, bazaar ou darcs. Cela vous permet d'utiliser git pour examiner ou
annuler les modifications apportées à `/etc`. Ou même de pousser le dépôt
ailleurs pour des sauvegardes ou pour récupérer sélectivement des modifications de configuration.

Il s'intègre aux gestionnaires de paquets comme apt pour valider automatiquement les modifications
apportées à /etc lors des mises à niveau de paquets. Il suit les métadonnées de fichiers que git
ne prend normalement pas en charge, mais qui sont importantes pour /etc, telles que
les permissions de `/etc/shadow`.

Il est assez modulaire et configurable, tout en restant simple à utiliser si vous
comprenez les bases du travail avec un système de contrôle de version.


## Avertissements de sécurité

Tout d'abord, un avertissement important : en plaçant /etc sous contrôle de version, vous
créez une copie de fichiers comme /etc/shadow qui doivent rester secrets. Dès que
vous avez une copie d'un fichier secret, il devient plus probable que le contenu du fichier
ne reste pas secret. etckeeper fait attention aux permissions des fichiers,
et s'assurera que les dépôts qu'il configure n'autorisent personne d'autre que root
à lire leur contenu. Cependant, vous devez *aussi* faire attention lors du clonage
ou de la copie de ces dépôts, à ne pas permettre à d'autres personnes de voir les données.

Comme git fusionne tous les fichiers dans des packs sous le répertoire .git, l'ensemble
du contenu du répertoire .git doit rester secret. (Idem pour mercurial
et .hg ainsi que bazaar et .bzr)

De plus, comme les systèmes de contrôle de version ne suivent pas le mode des fichiers
comme le fichier shadow, il effectuera une extraction en lecture pour tous, avant qu'etckeeper
ne corrige les permissions. Le tutoriel contient quelques exemples de moyens sûrs d'éviter
ces problèmes lors du clonage d'un dépôt /etc.

Notez aussi que `etckeeper init` exécute du code stocké dans le dépôt.
Donc ne l'utilisez pas sur des dépôts provenant de sources non fiables.


## Ce que fait etckeeper

etckeeper dispose d'un support spécial pour gérer les modifications de /etc causées par
l'installation et la mise à niveau de paquets. Avant qu'apt n'installe des paquets,
`etckeeper pre-install` vérifiera que /etc ne contient aucun changement non validé.
Après qu'apt a installé des paquets, `etckeeper post-install` ajoutera tout nouveau
fichier intéressant au dépôt, et validera les modifications.

Vous pouvez aussi exécuter `etckeeper commit` à la main pour valider les modifications.

Il existe également une tâche cron, qui utilisera etckeeper pour valider automatiquement
toute modification de /etc chaque jour.


## Limitations des VCS

Les systèmes de contrôle de version sont conçus comme un moyen de gérer du code source, pas comme
un moyen de gérer des répertoires arbitraires comme /etc. Cela signifie qu'il y a quelques
limitations qu'etckeeper doit contourner. Celles-ci incluent le stockage des métadonnées de fichiers,
les répertoires vides et les fichiers spéciaux.

La plupart des VCS, y compris git, mercurial et bazaar, n'ont qu'un suivi limité des
métadonnées de fichiers, étant capables de suivre le bit exécutable, mais pas les autres
permissions ou informations de propriétaire. (darcs ne suit même pas les bits exécutables.) Donc
les métadonnées de fichiers sont stockées séparément. Entre autres tâches, `etckeeper init`
configure un hook `pre-commit` qui stocke les métadonnées sur les propriétaires et
permissions des fichiers dans un fichier `/etc/.etckeeper`. Ces métadonnées sont stockées dans
le contrôle de version avec tout le reste, et peuvent être appliquées si le dépôt
devait être extrait à nouveau.

git et mercurial ne peuvent pas suivre les répertoires vides, mais ils peuvent être
significatifs parfois dans /etc. Donc le hook `pre-commit` stocke aussi
les informations qui peuvent être utilisées pour recréer les répertoires vides dans le
fichier `/etc/.etckeeper`.

La plupart des VCS ne prennent pas en charge plusieurs fichiers spéciaux que vous n'aurez _probablement_ pas
dans /etc, tels que les sockets unix, les tubes nommés, les fichiers à liens physiques (mais les liens symboliques
sont acceptables), et les fichiers de périphérique. Le hook `pre-commit` avertira si votre /etc
contient de tels fichiers spéciaux.

Darcs ne prend pas en charge les liens symboliques, donc ils sont aussi stockés dans
`/etc/.etckeeper`.


## Tutoriel

Un guide rapide de l'utilisation d'etckeeper.

Notez que le VCS par défaut est git, et ce tutoriel suppose que vous l'utilisez.
L'utilisation d'autres VCS devrait être globalement similaire.

D'abord, installez etckeeper. Quelque chose comme :

	apt-get install etckeeper

La commande `etckeeper init` initialise un dépôt /etc/.git/.
Si vous avez installé etckeeper depuis un paquet, cela a probablement été effectué
automatiquement lors de l'installation du paquet. Sinon, votre première étape est de
l'exécuter à la main :

	etckeeper init

La commande `etckeeper init` fait attention à ne jamais écraser les fichiers
ou répertoires existants dans /etc. Elle créera un `.gitignore` s'il n'en existe pas
déjà (ou mettra à jour le contenu à l'intérieur d'un bloc de commentaires "managed by etckeeper"),
configurera les hooks pre-commit s'ils n'existent pas déjà, etc. Elle
ne valide *pas* de fichiers, mais fait un `git add` de tous les fichiers intéressants pour
une validation initiale ultérieure.

Maintenant vous voudrez peut-être exécuter `git status` pour vérifier qu'il inclut tous
les bons fichiers, et aucun des mauvais. Et vous pouvez modifier le
`.gitignore` et ainsi de suite. Une fois que vous êtes prêt, il est temps de valider :

	cd /etc
	git status
	git commit -m "initial checkin"
	git gc # pack git repo to save a lot of space

Après cette première validation, vous pouvez utiliser les commandes git habituelles pour gérer
les modifications ultérieures :

	passwd someuser
	git status
	git commit -a -m "changed a password"

Répétez l'opération. Vous pourriez constater que certains fichiers sont modifiés par
des démons et ne devraient pas être suivis par git. Ceux-ci peuvent être retirés de git :

	git rm --cached printcap # modified by CUPS
	echo printcap >> .gitignore
	git commit -a -m "don't track printcap"

etckeeper s'intègre à apt (et systèmes similaires) afin que les modifications des fichiers intéressants
de /etc causées par l'installation ou la mise à niveau de paquets soient automatiquement
validées. Ici "intéressant" signifie les fichiers qui ne sont pas ignorés par
`.gitignore`.

Vous pouvez utiliser toutes les commandes git que vous voulez, mais gardez à l'esprit que, si vous
extrayez une branche différente ou une ancienne version, git opère directement
sur le /etc de votre système. Si vous décidez d'extraire une branche ou un tag,
assurez-vous d'exécuter à nouveau "etckeeper init", pour récupérer les modifications de métadonnées :

	git checkout april_first_joke_etc
	etckeeper init

Il est souvent préférable de cloner /etc ailleurs et de faire des choses potentiellement dangereuses
dans un répertoire de préparation. Vous pouvez cloner le dépôt en utilisant git clone,
mais faites attention à ce que le répertoire dans lequel il est cloné commence en mode 700, pour
empêcher quiconque d'autre de voir des fichiers comme `shadow`, avant qu'`etckeeper init`
ne corrige leurs permissions :

	mkdir /my/workdir
	cd /my/workdir
	chmod 700 .
	git clone /etc
	cd etc
	etckeeper init -d .
	chmod 755 ..

Une autre raison courante de cloner le dépôt est de faire une sauvegarde vers un
serveur. Lors de l'utilisation de `git push` pour créer un nouveau clone distant, assurez-vous que le
nouveau clone distant est en mode 700 ! (Et, évidemment, ne poussez que sur un transport
sécurisé comme ssh, et uniquement vers un serveur de confiance.)

	ssh server 'mkdir /etc-clone; cd /etc-clone; chmod 700 .; git init --bare'
	git remote add backup ssh://server/etc-clone
	git push backup --all

Si vous avez plusieurs machines utilisant etckeeper, vous pouvez commencer avec un
dépôt etckeeper sur une machine, puis ajouter le dépôt etckeeper d'une autre machine
comme dépôt distant git. Vous pouvez alors faire un diff par rapport à lui, examiner son
historique, fusionner avec lui, et ainsi de suite. Il ne serait probablement cependant pas sage
de faire "git checkout" de la branche de l'autre machine ! (Et si vous le faites, assurez-vous
d'exécuter "etckeeper init" pour mettre à jour les permissions des fichiers.)

	root@darkstar:/etc>git remote add dodo ssh://dodo/etc
	root@darkstar:/etc>git fetch dodo
	root@darkstar:/etc>git diff dodo/master group |head
	diff --git a/group b/group
	index 0242b84..b5e4384 100644
	--- a/group
	+++ b/group
	@@ -5,21 +5,21 @@ sys:x:3:
	adm:x:4:joey
	tty:x:5:
	disk:x:6:
	-lp:x:7:cupsys
	+lp:x:7:

Incidemment, cela signifie aussi que j'ai une sauvegarde du /etc de dodo sur darkstar.
Donc si darkstar est compromis, ces données pourraient être utilisées pour attaquer dodo
aussi. D'un autre côté, si le disque de dodo meurt, je peux le restaurer à partir de cette
sauvegarde pratique.

Bien sûr, il est aussi possible de tirer des modifications d'un serveur vers des machines
clientes, pour déployer des modifications de /etc. Une fois /etc sous contrôle de version,
le ciel est la limite..


## Configuration

Le fichier de configuration principal est `/etc/etckeeper/etckeeper.conf`

etckeeper exécute les fichiers exécutables dans `/etc/etckeeper/$command.d/`. (Il
ignore les mêmes que run-parts(1) ignorerait.) Vous pouvez modifier ces
fichiers, ou ajouter vos propres fichiers personnalisés. Chaque fichier individuel est court, simple,
et n'effectue qu'une seule action.

Par exemple, voici comment le configurer pour exécuter `git gc` après chaque exécution d'apt,
ce qui économisera beaucoup d'espace disque :

	cd /etc/etckeeper/post-install.d
	(echo '#!/bin/sh' ; echo 'exec git gc') > 99git-gc
	chmod +x 99git-gc
	git add .
	git commit -m "run git gc after each apt run"

Voici comment désactiver les validations automatiques après chaque exécution d'apt, tout en
laissant le git add des nouveaux fichiers :

	rm /etc/etckeeper/commit.d/50vcs-commit

## Intégration de sudo

etckeeper remarquera s'il est exécuté via sudo, et effectuera une validation
avec l'auteur défini à l'utilisateur qui a fait sudo vers root. C'est utile lorsqu'un
système a plusieurs administrateurs ; tant qu'ils utilisent sudo pendant leur
administration, et exécutent `sudo etckeeper commit` pour valider leurs modifications,
`git blame` peut montrer qui était responsable de chaque changement.

## Changer de VCS

Par défaut, etckeeper utilise git. Ce choix a été soigneusement fait ;
git est le VCS le mieux pris en charge par etckeeper et le VCS que les utilisateurs sont le plus
susceptibles de connaître.

[ Il est possible que votre distribution ait choisi de modifier etckeeper afin
  que son VCS par défaut ne soit pas git -- si c'est le cas, veuillez vous en plaindre auprès d'eux,
  car ils vous rendent les choses inutilement difficiles, et causent
  une divergence inutile des installations d'etckeeper.
  Vous ne devriez utiliser etckeeper avec un VCS autre que git que si vous êtes
  amoureux de l'autre VCS. ]

Si vous souhaitez utiliser un autre VCS, et qu'`etckeeper init` a déjà
été exécuté pour configurer un dépôt git, vous avez une décision à prendre : L'historique
enregistré dans ce dépôt est-il quelque chose que vous devez préserver, ou pouvez-vous
vous permettre de le jeter et de placer le /etc actuel dans le nouveau
VCS ?

Dans ce dernier cas, il vous suffit de suivre trois étapes :

	etckeeper uninit # deletes /etc/.git!
	vim /etc/etckeeper/etckeeper.conf
	etckeeper init

Dans le premier cas, vous devrez convertir le dépôt git vers l'autre
VCS en utilisant les outils disponibles pour cela. Ensuite vous pouvez
exécuter `etckeeper uninit`, déplacer les fichiers que votre nouveau VCS utilisera en place,
modifier `etckeeper.conf` pour changer le paramètre VCS, et enfin
`etckeeper init`. Cette procédure n'est clairement réservée qu'aux courageux.


## Inspiration

Deux articles de blog ont fourni l'inspiration pour les techniques utilisées par etckeeper :

* <http://www.jukie.net/~bart/blog/20070312134706>

* <http://bryan-murdock.blogspot.com/2007/07/put-etc-under-revision-control-with-git.html>

isisetup avait certains des mêmes objectifs qu'etckeeper, cependant, contrairement à lui,
etckeeper ne vise pas à être une porcelaine git avec son propre ensemble de commandes
pour manipuler le dépôt /etc. Au lieu de cela, etckeeper fournit une simple
procédure de configuration et des hooks pour configurer un dépôt /etc, puis s'efface ;
vous gérez le dépôt en utilisant les commandes VCS habituelles.


## Licence

etckeeper est sous licence version 2 ou supérieure de la GNU GPL.


## Site web

<https://etckeeper.branchable.com/>


## Auteur

Joey Hess <id@joeyh.name>