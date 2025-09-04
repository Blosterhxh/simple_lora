# Introduction

Je présente les résultats importants que j'ai pu déduire de la lecture des articles et des tests effectués avec les notebooks pour parfaire le pivotal tuning d'un modèle de diffusion. 

A chaque étape je résume ce que j'ai tiré des articles sans justification, pour voir les détails du raisonnement tout est dans les résumés individuels des articles.

Pour toutes ces étapes, on prend un nombre
de steps constant de 1000 pour ne pas
éterniser l entrainement, et on étudiera 
l'influence des autres parametres.

# Choix du learning rate pour l'inversion

Dans notre cas contrairement
au styleGAN,
l'inversion n'est pas gratuite. En effet en s'éloignant du token de départ, on perd en editability, là où dans le styleGAN avec l'inversion on reste en permanence dans W. Dans l'idéal, il faudrait qu'il y ait une première période où en s'éloignant du token de 
départ on gagne beaucoup en reconstruction et on perd peu en editability, et une seconde période où le gain en reconstruction est plus
faible et la parte en editability plus importante. Il faudrait alors inverser à la distance limite entre ces deux périodes.

Il faut donc qu'on mesure l'évolution de la reconstruction et de l'editability en fonction de l'éloignement au token, pour voir à quelle distance on doit inverser. Pour parcourir différentes distances, on utilise le fait que la distance parcouru vaut environ nb steps x learning rate, ce qu'on pourra confirmer en calculant la norme euclidienne entre le token obtenu avec l'entrainement et le token de départ. On réalise les mesures sur 3 learnings rates : 5e-3, 5e-4 et 5e-5.

![inversionconfigs.png](inversionconfigs.png)

L'intersection entre les deux périodes qu'on avait prévues se trouve à 5e-4, on va donc utiliser ce learning rate.

De plus on peut confirmer que le token s'éloigne bien avec l'augmentation du learning rate : 

![graphe2.png](graphe2.png)

Voici quelques images générées avec les trois learning rates pour visualiser les différences :

![inversion5e-3.png](inversion5e-3.png)
> LR = 5e-3

![inversion5e-4.png](inversion5e-4.png)
> LR = 5e-4

![inversion5e-5.png](inversion5e-5.png)
> LR = 5e-5


# Choix du learning rate pour le finetuning

Dans le styleGAN ils disent qu'ils 
appliquent un "léger" finetuning qui
leur permet de gagner en reconstruction
sans perdre en editability. Il faut donc définir qu'est-ce qu'un finetuning léger
avant de pouvoir trouver le bon learning
rate. Pour cela on va expliquer le fonctionnement général du finetuning.

Lors d'un finetuning une fonction apprend différents concepts. Par exemple en finetunant notre fonction sur un personnage, la fonction apprend son apparence, mais aussi des éléments non voulus comme sa position, l'environnement dans lequel il se situe. Les concepts dont la différence est la plus faible entre l'état de base et le dataset de finetuning seront améliorés les premiers car la fonction G dans l'espace des fonctions aura moins de distance à parcourir pour les améliorer. Par exemple, si un personnage dans le dataset contient plusieurs positions différentes, pour apprendre ces positions le modèle devra déjà abandonner le fait qu'il produit des positions aléatoires pour le token associé à ce personnage, et en plus il ne pourra apprendre une position donnée que lorsqu'il passe sur l'image du dataset avec cette position. Pour l'apparence, l'état de base de la fonction est déjà proche de l'état d'arrivée grâce à l'inversion, et chaque image du dataset contribue également à améliorer l'apparence donc elle sera améliorée beaucoup plus vite que la position.

Si on veut améliorer un concept précis comme l'apparence, il faut donc qu'on se déplace pas plus loin que la distance permettant de modifier ce concept et pas les autres. C'est cela que les auteurs du pivotal tuning veulent dire quand ils parlent de finetuning léger. 

Pour trouver la bonne distance de finetuning, on essaie différents learning rate pour trouver la distance limite où
on commence a apprendre les positions, et on s'arrête juste avant. On réalise les mesures sur 2 learnings rates : 1e-4 et 1e-5.

![finetuning1e-4.png](finetuning1e-4.png)
> LR = 1e-4

![finetuning1e-5.png](finetuning1e-5.png)
> LR = 1e-5

On voit que à 1e-5 seule l'apparence est apprise, et qu'ensuite d'autres features comme la position et l'environnement se rajoute à l'apprentissage. On
va donc choisir un learning rate de 1e-5.


# 3/ Régularisation

## A) Utilité de la régularisation pour le modèle de diffusion

### a.1) La régularisation dans styleGAN

Lorsque qu'on finetune G sur un latent wp, le finetuning va déborder sur les latents aux alentours,ce débordement
diminuant avec la distance. Le problème, c'est qu'on a qu'un seul modèle, donc on ne peut pas se permettre
de perdre toute la capacité de génération de visages de G juste pour apprendre une seule personne.
Il faut donc éviter ce débordement.

Pour cela, on ajoute un terme de régularisation, qui force G à coller au modèle de base sur les latents proches de wp.
Si on prend un latent où on régularise wr, et qu'on note tr le terme de régularisation et te le terme d'entrainement,
tout va se passer comme suit :
grad(G(wp)) = 0.1\*grad(tr) + 0.9\*grad(te)
grad(G(wr)) = 0.1\*grad(te) + 0.9\*grad(tr)
(les coefficients 0.1/0.9 ne sont pas exacts c'est juste pour montrer que selon le gradient un terme où l'autre va plus être pris en compte).
Ainsi les modifications de G sur wr par te vont être négligeables devant la régularisation, et le freinage de 
l'apprentissage de G sur wp par tr sera négligeable devant le terme d'entrainement. On peut donc
préserver les visages situés autour du pivot sans trop freiner l'apprentissage.

### a.2) Régularisation sur styleGAN = régularisation sur modèle de diffusion ?

Pour appliquer la régularisation au modèle de diffusion, on se place dans l'espace des embeddings après le transformer
de CLIP qui est entrainé pour avoir une relation entre la géométrie et la sémantique, contrairement à l'espace 
d'embeddings après les tokens. 
Il faut ensuite se demander si la régularisation appliquée telle quelle comme dans le styleGAN a une utilité pour nous.
En réalité, elle n'en a pas, car avec les loras, on peut facilement charger/décharger une config
donc ce n'est pas un problème si l'apprentissage déborde sur les autres embeddings, contrairement au styleGAN.

### a.3) Sélectionner les features apprises grâce à la régularisation

Cependant, on peut trouver une autre utilité à la régularisation.
Durant le finetuning, l'embedding \<tok1> va apprendre toutes les features du dataset : apparence, position, environnement ...
(quand je dis "l'embedding tok1 va apprendre", c'est un raccourci pour dire que G va changer ses valeurs sur tok1).
Pour l'empêcher d'apprendre des features inutiles, on pourrait ajouter un terme de régularisation qui force \<tok1>, sur les features
qu'on ne souhaite pas apprendre, à rester identique à la version avant le finetuning.

Ainsi, on aurait plus à limiter la distance 
parcourue par G avec un petit learning rate pour éviter l'apprentissage de feature parasites.
On peut augmenter le learning rate pour que G parcourt une plus grande zone de l'espace des fonctions
et trouver une meilleure reconstruction.

La limite de l'overfitting où les positions et l'environnement étaient appris se trouvaient à lr = 1e-4.
Dans la suite, on va utiliser ce learning rate et voir si on arrive à annuler l'apprentissage des positions et de l'environnement.

## B) Interpoler entre deux embeddings

Pour trouver le terme de régularisation, on va avoir besoin d'interpoler entre \<tok1> et character.
Voyons comme cela est possible.

On sait que le manifold des embeddings de texte dans CLIP est une ellispoïde
qu'on peut approximer par une sphère comme la majorité des coordonées ont la même variance.
Cette sphère est décalée de l'origine. 
Cependant, ceci n'est vrai que pour les embeddings de token de fin d'une phrase/image qui sont ceux sur lesquelles
la loss de CLIP porte. Pour les autres embeddings on ne sait rien. Or ce sont ces autres embeddings qui sont passés
sous forme de matrice au modèle de diffusion.

Comme character/tok1 sont en milieu de phrase à l'indice 5, on peut essayer de voir si les embeddings à l'indice 5
d'une phrase suivent la même répartition dans l'espace que les embeddings de fin. Pour cela, j'ai pris
le même dataset que celui utilisé par les chercheurs pour déterminer le manifold des embeddings de fin (MS-COCO 2014),
et j'ai calculé la norme moyenne et la variance de cette norme. Au final, j'ai obtenu le même résultat
que sur les embeddings de fin : norme de 24 et variance 1. On va donc pouvoir faire une vSLERP pour nos embeddings à la position 5 comme 
les chercheurs font sur les embeddings de fin.

Toutefois il y a un problème dans mon code car en calculant la norme moyenne et la variance pour le token de fin (je devrai donc avoir le même résultat que l'article), j'obtiens une variance plus faible avec les embeddings de base plutôt qu'avec les embeddings centrés, ce qui n'est pas cohérent avec le décalage de l'ellipsoïde texte de l'origine que les auteurs ont montré. J'ai exactement 27 de norme et 0.1 de variance. Je vais donc faire une SLERP et pas une vSLERP tant que ce problème n'est pas résolu.

## C) Le prompt apparence

Avant de pouvoir trouver le terme de régularisation, on va démontrer une propriété.

### c.1) Modélisation du problème

We begin by modeling our situation mathematically.

We model a prompt/image using two variables, $x$ : appearance, and $y$ : other features, which summarize the information contained in the prompt/image.
In what follows, we consider that the other features are solely the environment, $y$: environment, which does not change the reasoning and allows for better visualization.

The function $G$ is the Unet that transforms a prompt into an image:

$$
G(x_t, y_t) = G_1(x_t),G_2(y_t)
$$

where $G_1$ transforms the text appearance into image appearance and  $G_2$ transforms the text environment into image environment.

Let's take the training prompt: “an anime illustration of \<tok1>”.

Initially, the function $G$, which we will annotate as $G_a$, is:  

$$
G_a(\text{“an anime illustration of \<tok1>”}) = G_a(x_t = \langle tok1 \rangle,\, y_t = \langle tok1 \rangle) 
= G_{a1}(\langle tok1 \rangle),G_{a2}(\langle tok1 \rangle) 
= G_{a1}(\langle tok1 \rangle),\text{random}
$$

This is because the appearance and environment information are included in  $\langle tok1 \rangle$.  
In terms of appearance, it resembles our target character thanks to inversion.  
In terms of environment, it is generated randomly because  $\langle tok1 \rangle$ does not contain any information about the environment.

At the end of training:  

$$
G_b(x_t = \langle tok1 \rangle,\, y_t = \langle tok1 \rangle) 
= G_{b1}(\langle tok1 \rangle),G_{b2}(\langle tok1 \rangle)
$$

We would like:  

$$
G_{b2}(\langle tok1 \rangle) = \text{random}
$$

but this is not the case because the token  $\langle tok1 \rangle$  has been associated with both the appearance **and** the environment of the dataset.


### c.2) Existence du prompt apparence

We would like to demonstrate that there exists a prompt $(x_t, y_t)$, such that:  

$$
G_b(x_t, y_t) = G_{a1}(\text{char1}) + G_{b2}(\langle tok1 \rangle)
$$

where $G_{a1}(\text{char1})$ is a character appearance known by $G_{a}$, with $G_{a1}(\text{char1})  \neq  G_{b1}(\langle tok1 \rangle)$
so it is not affected by the finetuning.

Let's take the prompt, which we will call **appearance prompt**: $\text{“an anime illustration of } \langle tok1 \rangle \text{ woman with long blue hair”}$

We take:  

$$
\text{char1} = \text{“woman with long blue hair”}
$$  

We have :   

$$
G_b(\text{“an anime illustration of } \langle tok1 \rangle \text{ woman with long blue hair”}) = 
\big( p \cdot G_{b1}(\langle tok1 \rangle) + (1-p) G_{b1}(\text{char1}), G_{b2}(\langle tok1 \rangle) \big)
$$  

which can also be written as:  

$$
G_b(\text{“an anime illustration of } \langle tok1 \rangle \text{ woman with long blue hair”}) = 
\big( p \cdot G_{b1}(\langle tok1 \rangle) + (1-p) G_{a1}(\text{char1}),G_{b2}(\langle tok1 \rangle) \big)
$$  

since $G$ does not change its values on $char1$ with training.

The model constructs the appearance with a percentage taken from  $\langle tok1 \rangle\$ and a percentage taken from   $char1$

If we could get $p = 0$, the prompt would satisfy the property.  By testing this prompt on the model trained at $10^{-4}$,  we see that, on the contrary, we have $p = 1$.

![apparence1.png](apparence1.png)

To reduce this percentage, we can try to increase editability by interpolating between $\langle tok1 \rangle$ and character.
In fact, fine-tuning causes a loss in $\langle tok1 \rangle$ editability, and this loss decreases with distance. However, the effect
of fine-tuning decreases with distance, so we must ensure that even with distance, we still have:

$$
G_{b2}(interpolation) = G_{b2}(\langle tok1 \rangle\)
$$

We will therefore interpolate between tok1 and character, measure the influence of fine-tuning and editability,
and choose an optimal point. To measure the influence of fine-tuning, we calculate the cosine similarity
of the generated images with those in the dataset. To measure editability, we generate images using the simple prompt and the appearance prompt from the interpolated point
and calculate the cosine
similarity between the two. The decrease in similarity in this case will be due to the inclusion
of the appearance terms in the appearance prompt.
Since we plan to regularize fine-tuning at 1e-4, we perform these measurements on the model fine-tuned
at 1e-4.

### c.3) Résultats de l'interpolation

We have these evolutions of the influence of fine-tuning and editability with interpolation.

![interpolation3.PNG](interpolation3.png)

![interpolation2.PNG](interpolation2.png)

The influence decreases linearly while editability increases logarithmically. It would therefore be beneficial to take the interpolation at t = 0.5,
which gives us the best compromise between editability and the influence of fine-tuning.
However, when analyzing the images generated by the interpolations, we realize that the evolution of editability does not accurately represent the extent to which
appearance terms take precedence over $\langle tok1 \rangle$. In fact, appearance terms seem to be taken into account much more at t = 1,
which is not highlighted by the curve.

![interpolation4.PNG](interpolation4.png)

One explanation is that by moving away from $\langle tok1 \rangle$, the generator avoids overfitting and generates more random images. 

![interpolation5.PNG](interpolation5.png)

Thus, editability will decrease significantly between t= 0 and 
t = 0.5 because the cosine similarity will be decreased by the randomness of instance, even if the appearance is only slightly modified by the appearance prompt. To verify this, we change the measure of editability. We calculate the cosine similarity
between images generated with the same prompt, and we compare it with the cosine similarity of images generated with the simple prompt and the appearance prompt.
By calculating the difference between these two cosine similarities, we should be able to
quantify only the evolution of the consideration of appearance in the generation, without being confused by the increase in randomness.

![interpolation1.PNG](interpolation1.png)

Ultimately, the evolution of editability is still not representative of the consideration of appearance terms. I therefore decided to follow my observation
and regularize at t = 1, where we see that appearance is indeed modified and that other features such as environment and positions remain influenced by
fine-tuning, even though I am unable to
find a formula to substantiate this observation.

## D) Le terme de régularisation

### d.1) Trouver le terme de régularisation

The two previous points allowed us to find a prompt $(x_t,y_t)$ where 

$G_{b1}(x_t) = G_{a1}(\text{char1})$ and $G_{b2}(y_t) = G_{b2}(\langle tok1 \rangle)$

with $\langle G_{a1}(\text{char1}) \mid G_{b1}(\langle tok1 \rangle) \rangle = 0$.

We denote $G_e$ as the function $G$ trained from $G_a$ to $G_b$.  

On the prompt appearance:

- $G_{e1}(x_t) = G_{e1}(\text{char1})$ because the editability of $G_e$ is greater than that of $G_b$,  then $G_{e1}(\text{char1}) = G_{a1}(\text{char1})$ because
$G_{a1}(\text{char1})$ is not affected by the finetuning.

- $G_{e2}(y_t) = G_{e2}(\langle tok1 \rangle)$, because only $\langle tok1 \rangle$ contains environmental information.


We will modify the loss by adding a second term relating to the appearance prompt:

$$
\text{Loss} = \| G_e(\text{“an anime illustration of }\langle tok1 \rangle\text[{"}) - \text{dataset} \|
+
 \| G_e(\text{“an anime illustration of character woman with long blue hair”}) - G_a(\text{“an anime illustration of character woman with long blue hair”}) \|
$$

Let's break down the second term:  

$$
\| G_e(\text{“an anime illustration of character woman with long blue hair”}) - G_a(\text{“an anime illustration of character woman with long blue hair”}) \|
$$

Expanding:  

$$
= \| G_{a1}(\textttt{char1}),G_{b2}(\langle tok1 \rangle) - (G_{a1}(\textttt{char1}),\text{random}) \|
$$

$$
= \| 0,G_{b2}(\langle tok1 \rangle) - \text{random} \|
$$

Thus, the first term of the loss pushes G to resemble the dataset on $\langle tok1 \rangle$, and the second term forces $\langle tok1 \rangle$ not to store environmental information.

That said, there are still two problems. 
First, by learning the dataset environment, the first term decreases, and by keeping the original environment,
the second term decreases. So we don't know how the model will evolve to decrease the loss because the two possibilities are equivalent.
We will therefore weight the second term by a coefficient, such as *2, so that keeping the initial environment
decreases the loss more than learning the dataset environment.

The second problem is that in the term $\| G_{b2}(\langle tok1 \rangle) - \text{random} \|$, we do not know if learning the dataset environment will actually increase the term.
In fact, the base model generates a random environment, so comparing two generations of random environments potentially gives as much error 
as comparing a fixed environment (the one learned from the dataset) with random environments.

For now, we will set aside problem 2 by setting an environment in the regularization prompt: "an anime illustration of character woman with long blue
hair in a garden,“ and we will see if the term $\| 0,G_{b2}(\langle tok1 \rangle) - G_{a2}(garden) \|$ actually allows us to learn the environment ”a garden" rather than the one from
the dataset.

### d.2) Résultats de la régularisation






