# A/ Modifying an image using a diffusion model

Article: https://arxiv.org/pdf/2208.01618

In a diffusion model, the image is generated from a prompt, whereas with a GAN, we start from a latent w. To modify an image in the diffusion model, we modify the words
in the prompt, whereas in the GAN, we move in specific directions in W.
This comparison allows us to understand how to modify an image in a diffusion model by drawing inspiration from the method used for GANs. 
We create one or more tokens (we choose how many tokens we want to represent our image) and optimize their embeddings to obtain the best reconstruction.
Then we place these tokens in different prompts to modify the image.

To initialize the embeddings of these tokens, we can choose as the starting value the embedding of the class to which our concept belongs (for example, if it is an animal,
we can take the embedding of the token lion, tiger, etc.). We choose a prompt such as “a photo of s\*1 s\*2 s\*3 etc.” to generate the images during training.

In the article, the authors explored this inversion method from different angles to see its capabilities and limitations.

# B/ Capabilities:

## a) Semantic understanding

Even though the model is trained to represent the image of the object as a whole and not its interaction with the environment, it semantically understands
its composition and can make it interact with other objects. For example, if we teach it about a specific bowl, it will be able to contain
objects inside it, just like any other bowl.

![capacite1.PNG](capacite1.PNG)


## b) Abstract ideas

The model is not limited to learning about concrete objects, but can also grasp abstract ideas such as
a drawing style with this type of prompt: “A painting in the style of s\*”

![capacite2.PNG](capacite2.PNG)

# C/ Limitations:

## a) Interaction between new objects

Since objects are learned using a set of images in which they are at the center of the scene, the model has difficulty making them
interact with each other (it can make them interact with concepts it already knows, as we have seen, but it is the interaction between new concepts that
poses a problem). It is possible that training them to interact
with other new objects rather than being at the center of the scene could resolve this issue.

# D/ Evaluation of the method:

The inversion method can be varied by changing certain hyperparameters: learning rate, loss formula, number of tokens, etc.
Changing these hyperparameters does not alter the capabilities/limitations we saw earlier, which are specific to the method.
However, within the scope of these capabilities/limitations, they can improve reconstruction and editability if chosen wisely.
Methods are therefore needed to evaluate these two criteria in order to select the best hyperparameters.

## a) Reconstruction

We want to evaluate whether the generated concept resembles the concept in the dataset.
We generate 64 images, encode them and the images in the dataset in the CLIP embedding space, calculate
the cosine similarity of all generated image/dataset image pairs, and take the average.

## b) Editability

We want to evaluate how editable the learned concept is. To do this, we create prompts with variations
in environment, style, and interaction with other objects. For each prompt, we generate 64 images, calculate the
average vector in the CLIP embedding space, and calculate its cosine similarity with the prompt used
for generation, from which we have removed the new token “a photo of S\* on the moon” -> “a photo of on the moon.”


# E/ Choice of hyper-parameters

The authors of the article have analyzed the impact of different hyper-parameters on reconstruction and editability using the above methods,
which gives an idea of what to change according to our objective.

## a) Basic implementation

The implementation was done on a model which preceded stable diffusion, so the parameters may
not be exactly the same in our case, but they still give an idea of the starting point:

- For the embedding of the new word, we take as starting vector the vector of the class to which our object is attached
(e.g. cat, man, tree ...)
- 5 images in dataset
- 0.005 LR
- 4 batch-size
- 5000 steps


## b) Déplacement sur la courbe editability/distorsion

Les auteurs ont observé l'existence d'une courbe editability/reconstruction, et plus le/les vecteurs pris dans l'espace des embeddings
s'éloignent de l'embedding initial de la classe (ex : lion, tigre ...), plus on gagne en reconstruction tout en perdant en editability. 

![tradeoff.PNG](tradeoff.PNG)

Sur ce graphique, une augmentation en image similarity indique une meilleure reconstruction, et une augmentation en text similarity indique une meilleure editability (conformément
aux deux méthodes d'évaluation qu'on a vu précédemment). Il y a donc une courbe en haut a droite sur laquelle les différentes configurations sont positionnées, chaque configuration privilégiant plus ou moins la reconstruction/editability par rapport aux autres.

Ce compromis est le même que l'on observait
avec les styleGAN  : lorsque le/les latents s'éloignaient de l'espace W où le générateur avait été entrainé, on gagnait en reconstruction et perdait en editability.

Au final, la variation des hyperparamètres va éloigner plus ou moins les nouveaux tokens de l'embedding initial, il faut donc les choisir selon là où l'on veut être sur la courbe : soit on veut privilégier la reconstruction, soit l'editability.

### b.1) Gagner en editability

Pour gagner en editability il faut se rapprocher de l'embedding de départ. 

#### b.1.1) Diminuer le learning rate

Avec un learning rate plus faible on s'éloigne moins de l'embedding de départ avec un même nombre de steps. C'est le point "Low LR" sur le graphique.

#### b.1.2) Régularisation

On peut introduire dans la loss une norme L2 entre le nouvel embedding et l'embedding de la classe de l'objet, de manière à privilégier avec l'optimisation le changement de
direction du nouvel embedding plutôt que son éloignement de l'embedding de départ. C'est le point "W/Regularization" sur le graphique.

### b.2) Gagner en reconstruction

#### b.2.1) Augmenter le learning rate

Augmenter le learning rate augmente la reconstruction pour les mêmes raisons que le diminuer augmente l'editability. C'est le point "High LR" sur le graphique.

#### b.2.2) Augmenter le nombre de tokens appris

On peut choisir d'apprendre plusieurs embeddings au lieu d'un, ou même d'apprendre un embedding
commun pour toutes les images et un différent de manière à ce que le modèle concentre les informations communes
dans l'embedding partagé et que les emebeddings spécifiques récupères les informations particulières à chaque image
qui ne nous sont pas utiles. Ces méthodes se rapprochent de celles du styleGAN lorsqu'on choisissait d'inverser dans W\*k au lieu d'W\* pour avoir une meilleure reconstruction,
et de la même manière elles éloignent plus les nouveaux tokens de l'embedding initial en multipliant les vecteurs.

Toutefois ces méthodes par rapport au token unique apporte un très léger gain en
reconstruction voir aucun, et une perte beaucoup plus importante en flexibilité. On peut le voir sur le graphe avec les configurations "3-words", "2-words" et "Per-image token"
comparées à "Ours".

Au final, le réglage du learning rate est la méthode la plus simple pour se déplacer sur la courbe.

## c) S'affranchir de la courbe editability/reconstruction : le pivotal tuning

Dans le styleGAN pour s'affranchir de la limite imposée par la courbe, on découpe l'inversion en deux étapes.
On inverse l'image dans W pour avoir une bonne editability, et ensuite on finetune le générateur afin de modifier
les valeurs prises localement près de w. Ainsi on obtient une bonne reconstruction en profitant de l'editability
des points de W. Pour le modèle de diffusion, on peut appliquer cette même méthode en finetunant l'unet (on ne finetune pas aussi le text encoder car on ne veut pas
modifier la sémantique dans l'espace des embeddings mais seulement modifier l'apparence de certains embeddings ce qui est géré par l'unet).
Toutefois, les auteurs de l'article ont observé  qu'appliquer cette méthode dans le cas du modèle de diffusion n'est pas aussi efficace et fait perdre l'editability qui aura du être préservée par le finetuning. Ils proposent des idées pour modifier ce finetuning, mais qu'ils n'ont pas encore exploré.

![pivotaltuningdiffusion.PNG](pivotaltuningdiffusion.PNG)

Ces images correspondent à la génération d'un prompt étant censé placée la sculpture dans un tableau, avec à chaque fois un guidance scale s (paramètre qui influe sur a quel point
l'unet va prendre en compte le prompt pour générer l'image) plus élevé : 1,2 et 5. Au final le modèle est incapable de placer la statue dans un tableau donc on a perdu en editability.

## d) Augmenter le nombre d'images dans le dataset

Le nombre d'image affecte peu la reconstruction, un nombre optimal de 5 est trouvé pour l'editability.

![datasetsize.PNG](datasetsize.PNG)
