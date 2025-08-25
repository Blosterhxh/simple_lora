L'espace des embeddings de CLIP est de dimension 512 et contient à la fois les embeddings des phrases et les embeddings
des images.

# Thin shell theory :

Les auteurs donnent plusieurs résultats sur la géométrie d'une distribution dans un espace à plusieurs dimensions.
Avant cela ils donnent des définitions

## Variable aléatoire isotropique : 

![ellipse01.PNG](ellipse01.PNG)

## Distribution log concave :

![ellipse02.PNG](ellipse02.PNG)

## 1er résultat :

Pour une distribution isotropique, la norme moyenne de x vaut racine de n le nombre de dimensions. Les évènements
élémentaires sont donc sur une sphère de rayon racine de n.

![ellipse1.PNG](ellipse1.PNG)


## 2e résultat :

Pour une distribution isotropique et log concave, on a l'épaisseur de la coquille de la sphère qui est majorée par la
racine du logarithme de n.

![ellipse2.PNG](ellipse2.PNG)

## 3e résultat :

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
