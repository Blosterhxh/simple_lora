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

Il faut donc qu'on mesure l'évolution de la reconstruction et de l'editability en fonction de l'éloignement au token, pour voir à quelle distance on doit inverser. Pour parcourir différentes distances, on utilise le fait que la distance parcouru vaut environ nb steps x learning rate, ce qu'on pourra confirmer en calculant la norme euclidienne entre le token obtenu avec l'entrainement et le token de départ. 

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
on commence a apprendre les positions, et on s'arrête juste avant.


# Régularisation

## La régularisation dans styleGAN

Lorsque qu'on finetune G sur un latent wp, le finetuning va déborder sur les latents aux alentours,ce débordement
diminuant avec la distance. Le problème, c'est qu'on a qu'un seul modèle, donc on ne peut pas se permettre
de perdre toute la capacité de génération de visages de G juste pour apprendre une seule personne.
Il faut donc éviter ce débordement.

Pour cela, on ajoute un terme de régularisation, qui force G à coller au modèle de base sur les latents proches de wp.
Si on prend un latent où on régularise wr, et qu'on note tr le terme de régularisation et te le terme d'entrainement,
tout va se passer comme suit :
grad(G(wp)) = 0.1\*grad(tr) + 0.9\*grad(te)
grad(G(wr)) = 0.1\*grad(te) + 0.9\*grad(tr)
Ainsi les modifications de G sur wr par te vont être négligeables devant la régularisation, et le freinage de 
l'apprentissage de G sur wp par tr sera négligeable devant le terme d'entrainement. On peut donc
préserver les visages situés autour du pivot sans trop freiner l'apprentissage.

## Appliquer la régularisation au modèle de diffusion

Pour appliquer la régularisation au modèle de diffusion, on se place dans l'espace des embeddings après le transformer
de CLIP qui est entrainé pour avoir une relation entre la géométrie et la sémantique, contrairement à l'espace 
d'embeddings après les tokens. 
Il faut ensuite se demander si la régularisation appliquée telle quelle comme dans le styleGAN a une utilité pour nous.
En réalité, elle n'en a pas, car avec les loras, on peut facilement charger/décharger une config
donc ce n'est pas un problème si l'apprentissage déborde sur les autres embeddings, contrairement au styleGAN.

## Sélectionner les features apprises grâce à la régularisation

Cependant, on peut trouver une autre utilité à la régularisation.
Durant le finetuning, l'embedding \<tok1> va apprendre toutes les features du dataset : apparence, position, environnement ...
(quand je dis "l'embedding tok1 va apprendre", c'est un raccourci pour dire que G va changer ses valeurs sur tok1).
Pour l'empêcher d'apprendre des features inutiles, on pourrait ajouter un terme de régularisation qui force \<tok1>, sur les features
qu'on ne souhaite pas apprendre, à rester identique à la version avant le finetuning.

Pour ce faire, il faudrait trouver un prompt sur lequel G ne génère que les features inutiles de tok1, et ainsi
le terme de régularisation dG calculera l'erreur entre ces features sur le nouveau et l'ancien G.
On peut partir du prompt simple "an anime illustration of \<tok1>", et ajouter des termes qui précisent 
l'apparence "an anime illustration of <tok1> woman with blue long hair", de manière à ce que toutes les features de \<tok1>
sauf l'apparence soient exprimées. Pour la suite, on appelera "an anime illustration of <tok1> woman with blue long hair"
le prompt apparence.

## Augmenter la zone de l'espace des fonctions parcourue par G

La régularisation nous permet donc en théorie d'apprendre une seule feature. Ainsi, on a plus à limiter la distance 
parcourue par G avec un petit learning rate pour éviter l'apprentissage de feature parasites.
On peut donc essayer d'augmenter le learning rate pour que G parcourt une plus grande zone de l'espace des fonctions
et trouver une meilleure reconstruction.

Pour commencer, on peut essayer d'utiliser un learning rate de 1e-4, qui précédemment était la limite de 
l'overfitting.

## La compétition entre <tok1> et les termes d'apparence

J'ai précédemment dit que avec le prompt apparence, la feature apparence de <tok1> ne serait plus exprimée.
Malheureusement, ce n'est pas aussi simple. Le modèle construit l'apparence avec un pourcentage pris
des termes d'apparence, et un pourcentage pris de <tok1>.
Ainsi sur le modèle finetuné à 1e-4, on constate que 100% de l'apparence est prise depuis <tok1>.

Il faut diminuer ce pourcentage pour que les termes d'apparence prennent le dessus sur <tok1>.
Pour cela, on peut essayer d'augmenter l'editability, en interpolant entre tok1 et character.
En effet le finetuning fait perdre en editability tok1, et cette perte décroît avec l'éloignement.
Toutefois, si on s'éloigne trop, on gagne certes en editability mais l'effet du finetuning 
s'estompe donc la régularisation aura moins d'impact. Il faut donc trouver un compromis entre
les deux.

On va interpoler entre tok1 et character, mesurer l'influence du finetuning et l'editability,
et choisir un point optimal. Pour mesurer l'influence du finetuning, on calcule la cosine similarity
des images générées avec celles du dataset. Pour mesurer l'editability, on génère à partir du point
interpolé, le prompt simple et le prompt apparence, et on calcule la cosine
similarity entre les deux. 
Comme on compte régulariser le finetuning à 1e-4, on réalise ces mesures sur le modèle finetuné
à 1e-4.

## Interpoler entre character et tok1

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
que sur les embeddings de fin, on va donc pouvoir faire une vSLERP pour nos embeddings à la position 5 comme 
les chercheurs font sur les embeddings de fin.

Toutefois il y a un problème dans mon code car en calculant la norme moyenne et la variance pour le token de fin (je devrai donc avoir le même résultat que l'article), j'obtiens une variance plus faible avec les embeddings de base plutôt qu'avec les embeddings centrés, ce qui n'est pas cohérent avec le décalage de l'ellipsoïde texte de l'origine que les auteurs ont montré. Je vais donc faire une SLERP et pas une vSLERP tant que ce problème n'est pas résolu.

## Résultats

On obtient ces évolutions de l'influence du finetuning et de l'editability avec l'interpolation.

![interpolation3.PNG](interpolation3.png)

![interpolation2.PNG](interpolation2.png)

L'influence décroit linéairement tandis que que l'editability augmente logarithmiquement. On aurait donc intérêt à prendre l'interpolation à t = 0.5.
Toutefois, en analysant les images générées par les interpolations, on se rend compte que que l'évolution de l'editability ne représente pas bien à quel
point les termes d'apparence prennent le dessus sur tok1. En effet, les termes d'apparence semblent être beaucoup plus pris en compte à t = 1,
ce qui n'ait pas mis en valeur par la courbe.

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




