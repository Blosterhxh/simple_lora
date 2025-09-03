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

On commence par modéliser notre situation mathématiquement.

On modélise un prompt/une image par deux variables, $x$ : apparence,  $y$ : autres features,  qui résument les informations contenues par le prompt/l'image.  
Dans la suite on considère que les autres features sont uniquement l'environnement,  $y$ : environnement,  ce qui ne change rien au raisonnement et permet de mieux visualiser.

La fonction $G$ est l'Unet qui transforme un prompt en image :

$$
G(x_t, y_t) = G_1(x_t) + G_2(y_t)
$$

où $G_1$ transforme l'apparence texte en apparence d'image et  $G_2$ transforme l'environnement texte en environnement d'image.

Prenons le prompt d'entraînement : "an anime illustration of \<tok1>".

Au départ, la fonction $G$, qu'on annotera $G_a$, vaut :  

$$
G_a(\text{prompt}) = G_a(x_t = \langle tok1 \rangle,\, y_t = \langle tok1 \rangle) 
= G_{a1}(\langle tok1 \rangle) + G_{a2}(\langle tok1 \rangle) 
= G_{a1}(\langle tok1 \rangle) + \text{random}
$$

En effet, les informations d'apparence et d'environnement sont incluses dans  $\langle tok1 \rangle$.  
Pour l'apparence, elle ressemble à notre personnage cible grâce à l'inversion.  
Pour l'environnement, il est généré de manière aléatoire car  $\langle tok1 \rangle$ ne contient pas d'information sur l'environnement.

À la fin de l'entraînement :  

$$
G_b(x_t = \langle tok1 \rangle,\, y_t = \langle tok1 \rangle) 
= G_{b1}(\langle tok1 \rangle) + G_{b2}(\langle tok1 \rangle)
$$

On aimerait que :  

$$
G_{b2}(\langle tok1 \rangle) = \text{random}
$$

mais ce n'est pas le cas car le token  $\langle tok1 \rangle$  a été associé à l'apparence **et** à l'environnement du dataset.


### c.2) Existence du prompt apparence

On aimerait démontrer qu'il existe un prompt $(x_t, y_t)$, tel que :  

$$
G_b(x_t, y_t) = G_{a1}(\texttt{char1}) + G_{b2}(\langle tok1 \rangle)
$$

où $G_{a1}(\texttt{char1})$ est une apparence de personnage telle que :  

$$
\langle G_{a1}(\texttt{char1}) \mid G_{b1}(\langle tok1 \rangle) \rangle = 0
$$  

Autrement dit, les deux apparences n'ont rien en commun.


Prenons le prompt, qu'on appellera **prompt apparence** :  "an anime illustration of <tok1> woman with long blue hair"


On prend :  

$$
\texttt{char1} = \text{"woman with long blue hair"}
$$  

On a déjà :  

$$
\langle G_{a1}(\texttt{char1}) \mid G_{b1}(\langle tok1 \rangle) \rangle = 0
$$  

De plus :  

$$
G_b(x_t, y_t) = 
\big( p \cdot G_{b1}(\langle tok1 \rangle) + (1-p) G_{b1}(\texttt{char1}), G_{b2}(\langle tok1 \rangle) \big)
$$  

ce qui peut aussi s’écrire :  

$$
G_b(x_t, y_t) = 
\big( p \cdot G_{b1}(\langle tok1 \rangle) + (1-p) G_{a1}(\texttt{char1}),\; G_{b2}(\langle tok1 \rangle) \big)
$$  

comme $G$ ne change pas ses valeurs sur $char1$ avec l'entrainement.

Le modèle construit l'apparence avec un pourcentage pris de  $\langle tok1 \rangle\$ et un pourcentage pris de   $char1$

Si on arrivait à avoir $p = 0$, le prompt satisferait la propriété.  En testant ce prompt sur le modèle entraîné à $10^{-4}$,  on voit qu'au contraire on a $p = 1$ .

![apparence1.png](apparence1.png)

Pour diminuer ce pourcentage, on peut essayer d'augmenter l'editability, en interpolant entre $\langle tok1 \rangle\$ et character.
En effet le finetuning fait perdre en editability $\langle tok1 \rangle\$, et cette perte décroît avec l'éloignement. Toutefois l'effet
du finetuning décroit avec l'éloignement, il faut s'assurer que même en s'éloignant on a toujours :

$$
G_{b2}(embed) = G_{b2}(\langle tok1 \rangle\)
$$

On va donc interpoler entre tok1 et character, mesurer l'influence du finetuning et l'editability,
et choisir un point optimal. Pour mesurer l'influence du finetuning, on calcule la cosine similarity
des images générées avec celles du dataset. Pour mesurer l'editability, on génère à partir du point
interpolé, le prompt simple et le prompt apparence, et on calcule la cosine
similarity entre les deux. La baisse de similarité dans ce cas sera dû à la prise en compte
des termes d'apparence du prompt apparence.
Comme on compte régulariser le finetuning à 1e-4, on réalise ces mesures sur le modèle finetuné
à 1e-4.

### c.3) Résultats de l'interpolation

On obtient ces évolutions de l'influence du finetuning et de l'editability avec l'interpolation.

![interpolation3.PNG](interpolation3.png)

![interpolation2.PNG](interpolation2.png)

L'influence décroit linéairement tandis que que l'editability augmente logarithmiquement. On aurait donc intérêt à prendre l'interpolation à t = 0.5,
qui nous donne le meilleur compromis entre editability et influence du finetuning.
Toutefois, en analysant les images générées par les interpolations, on se rend compte que que l'évolution de l'editability ne représente pas bien à quel
point les termes d'apparence prennent le dessus sur tok1. En effet, les termes d'apparence semblent être beaucoup plus pris en compte à t = 1,
ce qui n'est pas mis en valeur par la courbe.

![interpolation4.PNG](interpolation4.png)

Une explication est que en s'éloignant de tok1, le générateur quitte l'overfitting et génère des images plus aléatoires. 

![interpolation5.PNG](interpolation5.png)

Ainsi, l'editability va beaucoup baisser entre t= 0 et 
t = 0.5, même si l'apparence est peu modifiée par le prompt apparence. Pour vérifier ça, on change la mesure de l'editability. On calcule la cosine similarity
entre images générées avec le même prompt, et on fait la différence avec la cosine similarity d'images générées avec le prompt simple et le prompt apparence.
En faisant la différence de ces deux cosine similarity, on devrait
pouvoir quantifier uniquement l'évolution de la prise en compte de l'apparence dans la génération, sans être brouillé par l'augmentation de l'aléatoire.

![interpolation1.PNG](interpolation1.png)

Au final, l'évolution de l'editability n'est toujours pas représentative de la prise en compte des termes d'apparence. J'ai donc décidé de suivre mon observation
et de régulariser à t = 1, où l'on voit que l'apparence est bien modifié et que les autres features comme l'environnement et les positions restent influencés par le
finetuning, même si je n'arrive
pas à trouver une formule permettant de concrétiser cette observation.

## D) Le terme de régularisation

### d.1) Trouver le terme de régularisation

Les deux points précédents nous ont permis de trouver un prompt $(x_t,y_t)$ où 

$G_{b1}(x_t) = G_{a1}(\texttt{char1})$ et $G_{b2}(y_t) = G_{b2}(\langle tok1 \rangle)$

avec $\langle G_{a1}(\texttt{char1}) \mid G_{b1}(\langle tok1 \rangle) \rangle = 0$.

On note $G_e$ la fonction $G$ entraînée de $G_a$ à $G_b$.  

Sur le prompt apparence :  

- $G_{e1}(x_t) = G_{e1}(\texttt{char1})$ car l’editability de $G_e$ est plus grande que celle de $G_b$,  puis $G_{e1}(\texttt{char1}) = G_{a1}(\texttt{char1})$.

- $G_{e2}(y_t) = G_{e2}(\langle tok1 \rangle)$, car seul $\langle tok1 \rangle$ contient des informations d’environnement.  


On va modifier la loss en ajoutant un deuxième terme portant sur le prompt apparence :  

$$
\text{Loss} = \| G_e(\text{"an anime illustration of tok1"}) - \text{dataset} \|
+
 \| G_e(\text{"an anime illustration of character woman with long blue hair"}) - G_a(\text{"an anime illustration of character woman with long blue hair"}) \|
$$

On détaille le deuxième terme :  

$$
\| G_e(\text{"an anime illustration of character woman with long blue hair"}) - G_a(\text{"an anime illustration of character woman with long blue hair"}) \|
$$

En développant :  

$$
= \| G_{a1}(\texttt{char1}) + G_{b2}(\langle tok1 \rangle) - (G_{a1}(\texttt{char1}) + \text{random}) \|
$$

$$
= \| G_{b2}(\langle tok1 \rangle) - \text{random} \|
$$


Ainsi, le premier terme de la loss pousse G à ressembler au dataset sur $\langle tok1 \rangle$, et le deuxième terme oblige $\langle tok1 \rangle$ à ne pas stocker d'informations d'environnement.

Ceci dit il y a encore deux problèmes. 
Premièrement, en apprenant l'environnement du dataset le premier terme diminue, et en gardant l'environnement original
le second terme diminue. Ainsi on ne sait pas comment va évoluer le modèle pour faire diminuer la loss car les deux possibiltiés sont équivalentes.
On va donc pondérer le deuxième terme par un coefficient, comme *2, pour que garder l'environnement initial
diminue plus la loss qu'apprendre l'environnement du dataset.

Le second problème, est que dans le terme $\| G_{b2}(\langle tok1 \rangle) - \text{random} \|$, on ne sait pas si apprendre l'environnement du dataset va réellement faire augmenter le terme.
En effet, le modèle de base génère un environnement aléatoire, donc comparer deux générations d'environnement aléatoire donne potentiellement autant d'erreur 
que comparer un environnement fixe (celui appris du dataset) avec des environnements aléatoires.

Pour l'instant, on va mettre de côté le problème 2 en se fixant un environnement dans le prompt de régularisation : "an anime illustration of character woman with long blue
hair in a garden", et on va voir si le terme $\| G_{b2}(\langle tok1 \rangle) - G_{a2}(garden) \|$ nous permet effectivement d'apprendre l'environnement "a garden" plutôt que celui 
du dataset.

### d.2) Résultats de la régularisation






