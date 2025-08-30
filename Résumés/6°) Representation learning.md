Articles : https://arxiv.org/pdf/2005.13149 , https://arxiv.org/pdf/2411.14517

# Supervised representation learning:

We want to associate images with a class, given that the images in the dataset are labeled and that we 
know all the possible classes. We model this problem using a probability distribution g(x). For
x an image, g(x) gives the probabilities that x belongs to each of the classes. The correct distribution of g(x)
is 1 for the class to which x belongs and 0 for the others. To get the model gtheta closer to g, we use entropy for loss, which gives a distance between g and gtheta. 

To construct g(x) from an image x, we create C embeddings zi, one for each class, and perform softmax
between these embeddings to find a probability distribution. For example, the probability that x belongs
to class i is: 

![representation1.PNG](representation1.PNG)

# Representation learning non supervisé : instance discrimination

Les images ne sont pas labellisées donc on ne peut pas entrainer le modèle a assigner les images à leur classe.
On choisit de donner une classe par image du dataset, et l'objectif sera que le modèle donne pour chaque image sa propre classe. 
g(x) a donc autant d'évènements élémentaires qu'il y a d'images dans le dataset. 
Je n'ai pour l'instant pas regardé l'utilité de ce genre de modèle, qui n'est pas aussi évident que le précédent.

Pour construire g(x) à partir d'une image x, on construit un seul embedding zi pour chaque image du dataset 
(au lieu de C
précédemment), et on fait le softmax d'un embedding sur tous les autres pour retrouver la distribution de probabilité.
La loss qui est l'entropie correspond au softmax de la norme de z (z=g(x)) sur le produit scalaire de z avec les autres
embeddings zi. Le numérateur ne peut pas être amélioré, mais le dénominateur peut être diminué en éloignant z des 
autres zi, ce qui permet de repousser les embeddings dans l'espace latent.

![representation2.png](representation2.png)

# Unsupervised representation learning: instance discrimination

This time the images are not labeled, so we cannot train the model to assign images to their class.
We choose to assign a class to each image in the dataset, and the goal will be for the model to assign each image to its own class. 
g(x) therefore has as many elementary events as there are images in the dataset. 
I have not yet looked into the usefulness of this type of model, which is not as obvious as the previous one.

To construct g(x) from an image x, we construct a single embedding zi for each image in the dataset 
(instead of C) and
the entropy (which is -log p where p is the probability to belong to its own class) is : 

![representation2.png](representation2.png)

The numerator cannot be improved, but the denominator can be reduced by moving z away from the 
other zi, which allows the embeddings to be pushed back into the latent space.

L'entrainement doit être simplifié car en l'état, il suppose que l'on recalcule les embeddings de toutes les images
à chaque étape, et également calcule la somme des exponentielles de tous les embeddings. Les simplifications sont
décrites dans l'article et pour l'instant je n'ai pas lu les justifications. En résumé ils stockent les embeddings
dans une banque M pour ne pas recalculer gtheta(xi) à chaque étape, et réduisent le dénominateur du softmax de N
à K<<N tirer uniformément dans N, certainement parce que beaucoup de termes ont une contribution proche de zero.

![representation3.PNG](representation3.PNG)

# Representation learning non-supervisé : local aggregation

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

La fonction g(x) est la même que précédemment, mais la loss est différente.
La loss cherche à maximiser la probabilité que x appartienne à la même classe que les éléments de son cluster parmi 
les voisins proches (là où avant elle permettait juste de maximiser la probabilité que x appartienne à sa propre
classe). Cela veut dire que parmi les voisins proches, les éléments du cluster vont se rapprocher de 
x, et comme on utilise un softmax, les éléments hors du cluster vont s'en éloigner. Toutefois cette loss ne traite
pas le cas où un élément est dans le cluster mais n'est pas un voisin proche de x. Dans ce cas, il va être un 
voisin proche d'un autre embedding et se faire déplacer par sa loss.

![representation4.PNG](representation4.PNG)
![representation5.PNG](representation5.PNG)


# Loss NT-Xent :

C'est la loss utilisée sur CLIP, qui est bimodale contrairement aux loss précédentes car on a les embeddings d'image
et de texte. Elle fonctionne sur le principe de l'instance discrimination : il y a une classe pour tous les textes et
images. Pour une paire texte-image, on veut maximiser la probabilité que l'image appartienne à la classe de son
texte, et que le texte appartienne à la classe de son image. On se retrouve donc avec deux loss au lieu d'une
quand on avait qu'une seule modalité. 

La loss complète maximise ces deux probabilités pour l'ensemble des pairs d'un batch. Maximiser différentes
probabilités en même temps permet de ne pas éloigner les vecteurs les uns des autres au hasard comme cela se
passerait si on maximisait la proba d'une seule paire. Là chaque vecteur s'éloigne les uns des autres en prenant
en compte le fait qu'il ne doit pas s'approcher d'une autre paire.

![representation6.png](representation6.png)

Cette loss n'est pas écrite comme dans l'article mais elle vaut exactement la même chose, elle montre cependant plus
explicitement la maximisation des deux probabilités pour chaque modalité. Pour s'en convaincre il suffit de faire
le changement de varibale j=k sur la deuxème espérance de cette formule.

![representation7.PNG](representation7.PNG)
