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
l'apprentissage de G sur wp par tr sera négligeable devant le terme d'entrainement.

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
Pour l'empêcher d'apprendre des features inutiles, on pourrait ajouter un terme de régularisation qui force \<tok1>, sur les features
qu'on ne souhaite pas apprendre, à rester identique à la version avant le finetuning.

Pour ce faire, il faut qu'on soit capable de générer des images qui prennent en compte uniquement les features indésirées de \<tok1>,
et ainsi on pourra calculer l'erreur entre ces features modifiées par le finetuning et ces features sur le modèle de base.
Pour cela, on peut partir du prompt de base "an anime illustration of \<tok1>", et ajouter des termes qui précisent 
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

Pour diminuer ce pourcentage, l'idée naturelle serait de s'éloigner de l'embedding de <tok1>.
En s'éloignant de <tok1>, deux choses se passent.
La fonction G est moins impacté par le finetuning, donc l'écart avec le modèle de base est plus faible, 
et la régularisation a moins d'effet.
D'un autre côté, le pourcentage de prise en compte des autres termes du prompt augmente.
Il faut donc trouver un compromis : s'éloigner suffisamment pour que les termes d'apparence
prennent le dessus sur ceux de l'embedding, mais pas trop pour G soit suffisament différente
de la fonction de départ et que la régularisation fasse effet.



## Interpolé entre <tok1> et <character>

Pour s'éloigner de <tok1> dans l'espace des embeddings, on peut utiliser le fait que l'on connaisse le manifold des embeddings de texte dans CLIP qui est une ellispoïde
qu'on peut approximer par une sphère, décalée de l'origine. On peut commencer par faire une
vSLERP entre <tok1> et <character>. 

La méthode vSLERP est de base pensée pour l'embedding du token de fin d'une phrase/image, qui est
le seul donc on connaît la géometrie dans l'espace latent (deux ellipsoïdes). Ceci dit, on peut vérifier que cette
structure reste globalement vraie pour les embeddings des tokens précédant le token de fin, en calculant leur norme moyenne et sa variance. En utilisant par exemple les tokens placés à la position 5 d'une phrase, on trouve que la structure d'ellipsoïde est 
préservée.

Toutefois il y a un problème dans mon code car en calculant la norme moyenne et la variance pour le token de fin (je devrai donc avoir le même résultat que l'article), j'obtiens une variance plus faible avec les embeddings de base plutôt qu'avec les embeddings centrés, ce qui n'est pas cohérent avec le décalage de l'ellipsoïde texte de l'origine que les auteurs ont montré.

Je vais donc faire une SLERP et pas une vSLERP pour la régularisation tant que ce problème n'est pas résolu.

Maintenant, il reste à choisir où interpoler exactement entre les deux embeddings. L'idée est qu'en s'éloignant,
les features comprises dans <tok1> vont s'atténuer, et il est inutile de régulariser si l'image générée ne contient
plus aucune feature de <tok1>. Mais si on est trop proche de <tok1>, le modèle ne va utiliser que les features de <tok1> et pas 
celles du reste du prompt. On doit donc trouver un compromis, avec un embedding où les features apparence du prompt vont prendre
le dessus, mais les features position, environnement de <tok1> seront également exprimées.

Pour trouver ce point, on peut générer pour un embedding une image avec le prompt de base, et une image avec le prompt
avec des termes d'apparence ajoutées. Si avec le prompt de base, on obtient une image proche de ce qu'aurait donné
<tok1>, cela veut dire que les features de <tok1> ont encore une influence à cette distance. De plus, si avec le 
prompt apparence on arrive à changer l'apparence du personnage, on a alors réussi à diminuer le pourcentage de prise
en compte de la feature apparence de <tok1>. Le meilleur point doit donc ressembler à <tok1> sur le prompt de base,
et avoir une apparence complètement différente sur le prompt apparence.

On génère les images pour les deux prompts, dans le cas d'une SLERP avec t = 0.5, et avec t = 1 ce qui est équivalent
à utiliser le token character.

Au final, on observe une reconstruction similaire pour le prompt de base, mais l'apparence est mieux modifiée 
pour t = 1. On va donc laisser tomber SLERP et juste régulariser sur le token <character>.



