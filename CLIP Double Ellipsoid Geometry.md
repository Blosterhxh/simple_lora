L'espace des embeddings de CLIP est de dimension 512 et contient à la fois les embeddings des phrases et les embeddings
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
