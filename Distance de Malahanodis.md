# Distance de malahadonis

On souhaite trouver une distance qui permet de savoir si un point est proche d'une distribution ou pas : plus la 
distance est faible plus le point à une densité de probabilité élevé, et plus il est loin plus il a une densité de 
probabilitié faible.

Comme on peut le voir sur ce schéma, la distance euclidienne ne fonctionne pas car deux points à des distances
euclidiennes différentes de la moyenne ont la même densité de probabilité.

Solution :

Rappel : 

Dans une distribution de probabilités, la variance nous donne les bornes dans lesquelles les données se trouvent,
tandis que la covariance nous indique la direction que ces données prennent.

Pour trouver une distance de malahanobis à partir de notre distance euclidenne, il faut normaliser les distances sur 
l'axe de corrélation et l'axe perpendiculaire. 

On utilise pour cela la 
racine de la matrice de covariance (on prend la racine car on normalise avec les écarts-types et pas la variance). 
La matrice de covariance nous donne sur la diagonale les normalisations dans 
la base canonique, et les autres termes de covariance permettent de recalculer cette normalisation dans les
directions de corrélation. Par exemple en deux dimensions :

Pour comprendre le fonctionnement, on peut voir comment cette matrice transforme un vecteur. Comme elle-est
symétrique, on peut la diagonaliser avec des matrices de rotation. Lorsqu'on multiplie un vecteur par cette matrice,
il va donc être exprimé dans une nouvelle base créée par rotation de la première, puis être normalisé sur les 
axes de la nouvelle base avec les coefficients de diagonale, puis retourner en sens inverse pour revenir à la base
de départ. On normalise donc suivant les axes qu'on voulait : axe de corrélation et axe perpendiculaire.

En réalité dans le calcul, on utilise l'inverse de la racine de la  matrice de covariance car 
la matrice de covariance augmente les distances dans les directions où la variance est grande ce qui est l'inverse 
de ce que l'on veut. Au final,
ce que fait l'inverse de la racine de la  matrice de covariance c'est transformé l'ellipse de l'espace de départ en 
un cercle
dans l'espace d'arrivé, et la distance de malahanobis est la distance euclidienne calculée sur ce cercle.
