<img width="582" height="292" alt="image" src="https://github.com/user-attachments/assets/51ea1081-e3bc-4f28-9c06-38416acf96c5" />L'espace des embeddings de CLIP est de dimension 512 et contient à la fois les embeddings des phrases et les embeddings
des images.

# 1/ Thin shell theory :

Les auteurs donnent plusieurs résultats sur la géométrie d'une distribution dans un espace à plusieurs dimensions.
Avant cela ils donnent des définitions

## a) Variable aléatoire isotropique : 

![ellipse01.PNG](ellipse01.PNG)

## b) Distribution log concave :

![ellipse02.PNG](ellipse02.PNG)

## c) 1er résultat :

Pour une distribution isotropique, la norme moyenne de x vaut racine de n le nombre de dimensions. Les évènements
élémentaires sont donc sur une sphère de rayon racine de n.

![ellipse1.PNG](ellipse1.PNG)


## d) 2e résultat :

Pour une distribution isotropique et log concave, on a l'épaisseur de la coquille de la sphère qui est majorée par la
racine du logarithme de n.

![ellipse2.PNG](ellipse2.PNG)

## e) 3e résultat :

Si on sait seulement que les coordonées de la variable aléatoire X sont toutes de moyenne 0, et que les fluctuations
de la norme de x autour de sa moyenne sont faibles comparées à cette moyenne, on a que la norme moyenne de x
vaut environ la racine de la trace de la matrice de covariance.

![ellipse3.PNG](ellipse3.PNG)

On peut faire un exemple pour se le représenter en deux dimensions. On prend le cas où x et y sont de moyenne nulle,
de variance différente et de covariance nulle (la covariance change juste l'orientation de l'ellipse qui n'intervient pas
dans le raisonnement donc on peut prendre une covariance nulle pour simplifier). Les vecteurs ayant des fluctuations de 
la norme faible par rapport
à la moyenne de la norme, ces vecteurs sont placés sur un cercle de rayon la moyenne de la norme. Comme x et y ont
une variance différente, on doit élargir/rétrécir le cercle sur les axes x et y, ce qui ne modifie pas la norme moyenne
à condition que  la différence de variance entre x et y est faible par rapport à cette norme moyenne. Au final tous
les points sont sur une ellipse, et environ sur un cercle
de rayon la racine de la trace de la matrice de covariance.

![ellipse4.png](ellipse4.png)

Dans cet exemple, les coordonées x et y suivent une loi uniforme sur leur valeurs possibles. Par exemple pour x, il y a pour toute valeur
possible xi 2 points correspondant (xi,+yi) et (xi,-yi). Dans CLIP, on va voir que les coordonnées suivent individuellement une loi normale
centrée en 0. En deux dimensions, on ne voit pas comment une telle loi permettrait de garder une norme moyenne de racine(tr(C)) sans fluctuations, vu qu'on aurait beaucoup de vecteurs proches de 0. Sauf que dans CLIP, on a 512 dimensions et comme on l'a vu dans le 3e résultat même si on a pas ici exactement les mêmes
conditions, plus il y a de dimension plus l'épaisseur de l'ellipsoide va diminuer. Cela explique qu'on puisse dans CLIP avoir une norme
moyenne racine(tr(C)) bien que les coordonnées suivent une loi normale centrée en 0.

# 2/ Analyse de l'espace des embeddings

Les auteurs commencent ensuite à analyser l'espace des embeddings pour en déduire des propriétés.

## a) Modality gap :

Les vecteurs des images et des textes ne suivent pas la même distribution. Par exemple pour les feature 93 et 134, 
les chercheurs obtiennent ces distributions.

![ellipse5.PNG](ellipse5.PNG)

Ils sont capables avec ces deux seules features de séparer linéairement l'ensemble des vecteurs d'image de celui des
vecteurs de texte.

![ellipse6.PNG](ellipse6.PNG)

Par contre toutes les features ne sont pas facilement séparables, ils en trouvent 9 qui sortent du lot dont la 93 et 
la 134.

![ellipse7.PNG](ellipse7.PNG)
![ellipse8.PNG](ellipse8.PNG)

## b) Forme des deux distributions :

En prenant les 10 premières features de chaque distribution, ils constatent que les valeurs ont un pic au niveau de 0.

![ellipse9.PNG](ellipse9.PNG)

A l'inverse la norme vaut en moyenne une valeur non-nulle, avec une variance beaucoup plus faible que les coordonées
individuelles.

Les auteurs en concluent :

On sait qu'avec le 3e résultat, la norme moyenne vaut racine(tr(C)). Pour l'instant je n'ai pas compris à quel moment 
cette information était utile dans l'article.

![ellipse10.PNG](ellipse10.PNG)

## c) Shell ellipsoïdale :

Au sein d'une distribution, les auteurs observent que les features ont des variances différentes. Ils en concluent donc que la shell de rayon
racine de tr(C) est 
ellipsoïdale et non sphérique. Ici je ne suis pas sûr de comprendre comment ils en arrivent à cette conclusion, mais
voici une justification que j'ai trouvée. Premièrement on définit qu'est ce que c'est être sur une shell ellipsoïdale :
c'est quand l'ensemble de points à une norme de malahanodis environ constante. A première vue on peut penser que ce 
n'est pas le cas pour nous, car la distance euclidenne est environ constante a racine(tr(C)) et la transformation
de malahanodis pour recalculer une distance euclidienne va étirer/réduire certaines coordonées, donc elle ne devrait
pas pouvoir être constante elle aussi.
Cependant lorsqu'on observe le graphe des variances des features, il apparaît que les variances sont en moyenne autour
de 0.1, et que les variances plus élevées sont rares.

![ellipse11.PNG](ellipse11.PNG)

Ainsi en déformant l'espace avec malahanodis, les points
restreints à ces features vont rester sur une sphère comme toutes les coordonées auront été étirées presque de manière
égale. Les autres coordonées étant minoritaires, leur modification ne devrait pas trop impactée la norme des vecteurs
qui devrait rester proche de la sphère. Au final ce n'est pas contradictoire d'avoir une norme euclidienne et une norme
de malahanodis constante, si les variances des coordonées sont en moyenne les mêmes. 
Pour savoir si les points sont plus sur une shell ellispoïdale que sphérique, on pourrait recalculer la norme de 
malahanodis pour ces points et voir si la variance autour de la valeur moyenne est plus faible que dans le cas euclidien,
mais je ne l'ai pas vu dans l'article.

## d) Orientation des ellipsoïdes :

Les auteurs calculent la off-diagonal dominance pour chaque ligne de la matrice de covariance pour voir si
la diagonale est prépondérante par rapport aux restes des coefficients, autrement dit si les variables sont corrélées
ou non. 

![ellipse12.PNG](ellipse12.PNG)

Ils observent des valeurs significatives, signifiant que les variables sont corrélées et que donc les ellipses
sont penchées (comme en dimension 2 où une covariance implique que l'axe de l'ellipse est orientée selon cette
covariance).

![ellipse13.PNG](ellipse13.PNG)

## e) Centre des ellipsoïdes :

Les auteurs veulent évaluer la distance du centre des ellipsoïdes à l'origine. Pour cela, ils comparent l'écart-type
au vecteur moyen.

![ellipse14.PNG](ellipse14.PNG)

Ils trouvent le même ordre de grandeur. Cela signifie que les ellipsoïdes sont significativement éloignés de l'origine.

# 3/ Justification de la structure en deux ellipsoïdes

Les auteurs montrent ensuite que cette répartition en deux ellipsoides est optimale pour la loss CLIP.

## a) La transformation par un encodeur d'une distribution d'entrée en une distribution contenue dans une ellipsoïde

Je n'ai pas l'explication de pourquoi un encodeur transforme la distribution de départ en une distribution qui est 
centrée autour d'une valeur pour chaque coordonées.
En tout cas une fois qu'on a une distribution centrée pour chaque coordonée, les points de la distribution sont
dans une ellispoïde qui a pour chaque coordonée pour rayon environ la variance de la coordonnée.

## b) Le fonctionnement des deux encodeurs

Avant d'expliquer l'origine des autres propriétés des ellipsoïdes, il faut comprendre comment fonctionne les deux encodeurs.
Chaque encodeur fonctionne grâce à des transformers qui permettent de comprendre le sens des entrées. L'encodeur va synthétiser
l'information qu'il a extraite de l'entrée dans un vecteur de l'espace des embeddings.

La loss de CLIP correspond à la somme de la distance de la distribution des images de celles du texte et de la distance de la distribution
du texte à celle des images. Pour minimiser cette somme, il faut que chaque paire du dataset est une cosine similarity de 1, et que les
vecteurs de cette paire aient une cosine similarity de 0 avec les vecteurs de paires différentes. Cette optimum global, si il existe
avec les fonctions d'encodeur d'image et de texte qui ont été choisies, est difficile à trouver. Il existe cependant un optimum
local plus facile. Pour cela, il faut que des images/textes sémantiquement proches aient des embeddings similaires. De cette manière,
la loss diminue car au lieu d'avoir une cosine similarity moyenne entre tous les vecteurs, on aura une cosine similarity moyenne
pour les vecteurs correspondant à des entrées sémantiquement proches, et une cosine similarity proche de 0 pour les autres.
Cet optimum local, en plus d'être plus facilement atteignable que l'optimum global, est favorisé par la forme des fonctions des encodeurs
qui avec les transformers ont la capacité de comprendre le sens des entrées.

Jusque-là on a vu que les encodeurs sont capables d'extraire des informations de leur entrée, et qu'ils doivent utiliser ces informations pour rapprocher des vecteurs sémantiquement proches. Pour cela, il faut qu'ils encodent une information identique de manière identique,
et qu'ils extraient exhaustivement toutes les informations contenues dans leur entrée. Or cette extraction exhaustive n'est pas possible,
car l'entrée contient parfois trop d'informations. Par exemple dans une image, les
informations contenues sont infinies si on va chercher tous les détails précis. Les encodeurs apprennent donc à extraire seulement les
informations qui en moyenne sont les plus utiles pour rapprocher les bonnes paires entre elles.

## c) L'existence de deux ellipsoïdes distinctes pour l'encodeur d'image et l'encodeur de texte

On a vu que les encodeurs n'extraient pas exhaustivement toutes les informations de leur entrée mais uniquement celles qui sont en
moyenne les plus utiles pour minimiser la loss. Ainsi les informations extraites par les deux encodeurs pour 
une même paire ne sont pas toujours identiques : le texte peut avoir été très descriptif pour une image simple, l'encodeur image
voyant une image simple va extraire peu d'informations contrairement à l'encodeur texte.

![ellipse15.PNG](ellipse15.PNG)

Ici les Ci/Ct indiquent la cosine similarity moyenne de l'embedding avec les autres vecteurs de sa modalité. On voit qu'une image et un
texte d'une même paire peuvent avoir des Ci/Ct très différents, indiquant que un encodeur extrait des informations communes par rapport
à sa modalité tandis que l'autre extrait des informations plus précises et rares.

Comme les deux encodeurs codent identiquement les informations, mais que leurs ensembles d'informations extraites n'est 
pas exactement le même, on obtient dans l'espace latent des distributions proches mais avec des différences. Ces différences 
sont plus ou moins grandes selon les features, et elles sont surtout importantes sur 9 features précises. Je n'ai pas trouvé l'explication
de la concentration des différences sur ces 9 features.

## d) La concentration des points sur la shell des ellipsoïdes

La concentration des points sur la shell est observée avec le calcul de la norme moyenne et de sa variance, mais ce n'est
pas expliqué pourquoi cette concentration intervient. C'est possiblement du à la dimension élevée n = 512, comme c'est le
cas pour les distributions isotropiques log concaves.

## e) Le décalage des ellipsoïdes par rapport à l'origine 

### e.1) Compromis entre uniformity et alignment

Les auteurs réecrivent la loss avec un terme d'alignment et un terme d'uniformity.

![representation7.PNG](representation7.PNG)

Le terme d'alignment rapproche les
vecteurs d'une même paire, et le terme d'uniformity repousse les vecteurs de paires différentes. Pour diminuer la loss,
on veut augmenter l'alignment et diminuer l'uniformity.

Pour comprendre le décalage des ellipsoïdes, ils déplacent l'ellipsoïde des images par rapport à l'origine pour voir 
l'évolution de la loss et précisemment de ces deux termes.

![ellipse16.PNG](ellipse16.PNG)

On observe que l'alignment augmente en s'éloignant de l'origine, tandis que l'uniformity diminue en s'en rapprochant.
La loss optimale correspond à une ellipsoïde centrée sur le vecteur moyen qui a été trouvé par l'entrainement.

![ellipse17.PNG](ellipse17.PNG)

On peut expliquer ces évolutions différentes de l'alignment et de l'uniformity.
La différence vient du fait que pour les embeddings de l'encodeur d'image, les directions possibles augmentent en se
rapprochant de l'origine et diminuent en s'en éloignant.
Si on est proche de l'origine, on peut donc plus facilement
donner aux vecteurs de paires différentes des directions différentes ce qui diminue l'uniformity. Cependant pour les 
paires d'embedding que le modèle n'arrive pas à réunir (ex : bruit dans les entrées, encodeurs qui n'extraient pas les 
mêmes informations du côté image et texte), la cosine similarity de la paire sera plus faible et l'alignement diminue.
L'inverse se passe en s'éloignant de l'origine, et au final le meilleur centre pour l'ellipsoïde est un compromis
entre uniformity et alignment.

## e.2) Gestion des faux négatifs

Un faux négatif est une paire image texte sémantiquement proches mais qui ne sont pas une vraie paire du dataset.
Sur ces faux négatifs les informations clés extraites sont similaires donc ils doivent être proches dans l'espace
latent. 
Le problème avec le centrage de l'ellipsoïde à l'origine, c'est que un petit déplacement des vecteurs entraîne une
chute rapide de la cosine similarity. En effet, si on calcule le gradient d'une cosine similarity par rapport à un 
vecteur, on voit que ce gradient est inversement proportionnel à la norme de ce vecteur.

![ellipse18.PNG](ellipse18.PNG)

Cela veut dire que l'on peut difficilement avoir une cosine similarity élevée pour des faux-négatifs.
En éloignant l'ellipsoïde de l'origine, on diminue le gradient de la cosine similarity et ainsi on augmente la cosine
similarity des faux négatifs.

## e.3) Encodage différent des informations communes et rares :

On prend un encodeur, par exemple l'encodeur image. Si une image a des informations extraites fréquentes, elle doit 
avoir une cosine similarity moyenne avec les autres embeddings image plus élevée qu'une image avec des informations rares.
Si l'ellispoïde image est centrée à l'origine, les embeddings étant sur la shell, tout embedding à la même cosine
similarity aux autres embeddings peu importe sa position sur la shell. On ne peut donc pas différencier les images
communes des images rares. A l'inverse en éloignant l'ellipsoïde de l'origine, un embedding avec une direction centrale
aura une cosine similarity moyenne plus élevée avec les autres embeddings qu'un embedding avec une direction extrême.
Le schéma ci-dessous montre un exemple simplifié avec une sphère, mais c'est le même raisonnement pour un ellipsoïde.

![ellipse19.PNG](ellipse19.PNG)

Ainsi, on peut placer les images communes dans des embeddings dirigés vers le centre et des images rares dans des 
embeddings aux directions extrêmes de l'ellipsoïde.

