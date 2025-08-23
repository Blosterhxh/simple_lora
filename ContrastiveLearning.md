On aimerait maintenant regrouper les images qui font partie d'une même classe entre elles, et pas avoir une
classe pour une image ce qui éloigne inutilement des images similaires.
Comme on ne connaît pas à l'avance les classes, on fait un clustering sur le dataset.
Il y a alors deux mesures de similarité sur l'espace : la distance géométrique et le clustering.
Là où auparavant on avait juste à améliorer la distance géométrique entre les embeddings, ici il faut en plus
prendre en compte le clustering. On améliore ces mesures conjointement en deux étapes :
- on rapproche les vecteurs qui sont proches géométriquement et dans un même cluster
- pour un vecteur dans le cluster et loin géométriquement, ou un vecteur hors du cluster et proche géométriquement :
on éloigne géométriquement ce vecteur du vecteur cible. On pourrait faire l'inverse et le rapprocher dans les deux
cas, mais cela amènerait à renforcer le cluster de départ qui est aléatoire comme la fonction n'est pas entrainée.
Au contraire on veut déplacer au maximum les vecteurs et faire varier le cluster avec l'entrainement.

La loss permet de maximiser la probabilité que x appartienne à la même classe des éléments du cluster parmi 
les voisins proches. Cela veut dire que parmi les voisins proches, les éléments du cluster vont se rapprocher de 
x, et comme on utilise un softmax, les éléments hors du cluster vont s'en éloigner. Toutefois cette loss ne traite
pas le cas où un élément est dans le cluster mais n'est pas un voisin proche de x. Dans ce cas, il va être un 
voisin proche d'un autre embedding et se faire déplacer par sa loss.
