# Modifier une image dans un modèle de diffusion


Dans un modèle de diffusion, on génère l'image à partir d'un prompt là où avec un GAN on part d'un latent w. Pour modifier une image dans le modèle de diffusion, on modifie les mots
du prompt là où dans le GAN on se déplace selon des directions spécifiques dans W.
Cette comparaison nous permet de comprendre comment modifier une image dans un modèle de diffusion en s'inspirant de la méthode pour les GANs. 
On créé un ou plusieurs tokens (on choisit combien de tokens on veut pour représenter notre image) et on optimise leurs embeddings pour obtenir la meilleure reconstruction.
Ensuite on place ces tokens dans différents prompts pour modifier l'image.

Pour initialiser les embeddings de ces tokens, on peut choisir comme valeur de départ l'embedding de la classe à laquelle appartient notre concept (par exemple si c'est un animal
on peut prendre l'embedding du token lion, tigre ...). On choisit un prompt comme "a photo of <tok1>
<tok2> <tok3> ... etc." pour générer les images lors de l'entrainement.

Dans l'article, les auteurs ont exploré cette méthode d'inversion sous différents angles pour voir ses capacités et ses limites dans le cas du modèle de diffusion.


# Capacités :

## Compréhension sémantique

Même si le modèle est entrainé à représenter l'image de l'objet en entier et pas son interaction avec l'environnement, il comprend sémantiquement
sa constitution et peut le faire interagir avec d'autres objets. Par exemple, un type de bol appris peut contenir
des objets à l'intérieur.

## Idées abstraites

Le modèle ne se limite pas à l'apprentissage d'objets concrets mais peu aussi saisir des idées abstraites comme
un style de dessin avec ce type de prompt : “A painting in the style of <tok1>”

# Limites :

## Interaction entre nouveaux objets

Les objets étant appris avec un set d'image où ils sont au coeur de la scène, le modèle a du mal a les faire
interagir entre eux (il peut les faire interagir avec des concepts qu'il connaît déjà comme on l'a vu, c'est l'interaction entre nouveaux concepts qui
pose problème). Il est possible que réaliser un entrainement où ils sont en interaction
avec d'autres nouveaux objets et non plus au coeur de la scène débloque cette situation.

# Evaluation du modèle :

## Reconstruction

On veut évaluer si le concept généré ressemble au concept du dataset.
On génère 64 images, on les encode ainsi que les images du dataset dans l'espace d'embeddings de CLIP, on calcule
la cosine-similarity de toutes les pairs image générée/image du dataset et on en fait la moyenne

b) Flexibilité

On veut évaluer à quel point le concept généré est modifiable. Pour cela on créé des prompts avec des variations
de décor, de style, et d'interaction avec d'autres objets. Pour chaque prompt, on génère 64 images, on fait une
moyenne du vecteur dans l'espace des embeddings de CLIP, et on calcule sa cosine-similarity avec le prompt utilisé
pour la génération à qui on a enlevé le nouveau mot : "a photo a S* on the moon" -> "a photo of on the moon".

Choix des hyper-paramètres :

a) Implémentation de base

L'implémentation a été faite sur un ancien modèle précédent Stable Diffusion donc les paramètres ne sont peut
être pas exactement les mêmes dans ce cas, mais ils donnent quand même une idée du point de départ :

- Pour l'embedding du nouveau mot, on prend comme vecteur de départ le vecteur de la classe à laquelle est rattachée
notre objet (ex : chat, homme, arbre ...)
- 5 images dans le dataset
- 0.005 LR
- 4 batch-size
- 5000 steps

b) Déplacement sur la courbe flexibilité/distorsion

En entrainant le modèle avec différents hyperparamètres ou avec des variations structurelles, on remarque
une l'apparition d'une courbe de flexibilité/distorsion. Quand le nouvel embedding est plus proche de l'espace
des embeddings de départ, il y a une meilleur flexibiltié, et quand il en est éloigné, il a une meilleure 
reconstruction mais est moins modifiable.

b1) Gagner en flexibilité

Pour gagner en flexibilité il faut se rapprocher de l'espace des embeddings de départ. 

b.1.1) Diminuer le learning rate

Avec un learning rate plus faible on s'éloigne moins de l'embedding de départ avec un même nombre de steps.

b.1.2) Régularisation

On peut introduire dans la loss la norme L2 entre le nouvel embedding et l'embedding de la classe de l'objet.

b2) Gagner en reconstruction

b.2.1) Augmenter le learning rate

Augmenter le learning rate pour les mêmes raisons que le diminuer augmente la flexibilité.

b.2.2) Augmenter le nombre de tokens appris

On peut choisir d'apprendre plusieurs embeddings au lieu d'un, ou même d'apprendre un embedding
commun pour toutes les images et un différent de manière à ce que le modèle concentre les informations communes
dans l'embedding partagé et que les emebeddings spécifiques récupères les informations particulières à chaque image
qui ne nous sont pas utiles. Toutefois ces méthodes par rapport au token unique apporte un très léger gain en
reconstruction et une perte beaucoup plus importante en flexibilité comme on peut le voir sur le graphe.

Au final, le réglage du learning rate est la méthode la plus simple pour choisir quoi prioriser entre reconstruction
et flexibilité.

c) S'affranchir de la courbe flexibilité/reconstruction : le pivotal tuning

Dans le styleGAN pour s'affranchir de la limite imposée par la courbe, on découpe l'inversion en deux étapes.
On inverse l'image dans W pour avoir une bonne flexibilité, et ensuite on finetune le générateur afin de modifier
localement les valeurs prises sur W. Ainsi on obtient une bonne reconstruction en profitant de la flexibilité
des points de W. Pour le modèle de diffusion, cette méthode est applicable, mais on perd quand même la flexibilité
avec le finetuning là où elle était préservée avec le styleGAN ce qui rend la méthode peu intéressante. On finetune 
dans notre cas seulement le unet, pas le text encoder, car notre objectif est un changement léger de l'apparence en
conservant la sémantique de l'espace des embeddings de texte (de la même manière qu'on ne va pas entrainer le 
mapping network dans styleGAN).

d) Augmenter le nombre d'images dans le dataset

Le nombre d'image affecte peu la reconstruction, un total de 5 est optimal pour la flexibilité.



Avantage :

a) Guidage avec un prompt

On peut guider avec un prompt précis le modèle pour obtenir notre image. Mais cela pose plusieurs problèmes.
Premièrement, certains structures complexes sont difficiles à détailler avec des mots. Ensuite, il a été observé
que le modèle lorsqu'il a en entré de longs prompts se concentrent sur certains mots et ignorent les autres, ce qui
peut poser problème s'ils se concentrent sur des détails au lieu du coeur du prompt.

b) Guidage avec une image

On peut aussi guider un modèle avec une image en entrée comme avec DALL-E 2. Le problème ici est qu'avec une 
structure singulière l'encodeur CLIP aura du mal à trouver un bon embedding et donc on aura un résultat peu
ressemblant à l'image de départ. 





