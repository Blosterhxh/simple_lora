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

Dans styleGAN, la régularisation s'applique sur des latents proches du pivot wp, pour empêcher que les visages aux
alentours soient modifiés par le finetuning. C'est important car on a qu'un seul modèle, et on ne veut pas tout
perdre en apprenant un seul personnage.

Schématiquement, cette régularisation se passe comme suit : 

wp : le pivot
wr : le latent de régularisation
loss (wp) : le terme d entrainement
loss(wr) : le terme de régularisation
modifications en wp =  loss(wp) - 0.1 loss(wr)
modifications en wr = loss(wr) -0.1 loss(wp)
(ce n'est pas du tout exact mathématiquement c'est juste pour visualiser l'influence de chaque terme sur ces deux points)

L'idée est que les modifications de G sur un latent w vont se propager aux latents proches, cette propagation diminuant
avec la distance. En ajoutant ce terme de régularisation, on s'assure que les latents proches subissent peu de 
modifications. On perd alors un peu en reconstruction au niveau de wp, mais c'est faible par rapport au gain en wr.

## Appliquer la régularisation au modèle de diffusion

Pour appliquer la régularisation au modèle de diffusion, on se place dans l'espace des embeddings après le transformer
de CLIP qui est entrainé pour avoir une relation entre la géométrie et la sémantique, contrairement à l'espace 
d'embeddings après les tokens. 
Il faut ensuite se demander si la régularisation appliquée telle quelle comme dans le styleGAN a une utilité pour nous.
En réalité, elle n'en a pas, car avec les loras, on peut facilement charger/décharger une config
donc ce n'est pas un problème si l'apprentissage déborde sur les autres embeddings, contrairement au styleGAN.

Cependant, on pourrait voir une autre utilité de la régularisation
, si on était capable de la cibler à certaines features. En effet,
durant
l'entrainement, le modèle apprend toutes les features du dataset. Il serait utile de pouvoir diminuer l'apprentissage
des features inutiles (position, environnement) en appliquant une régularisation ciblée sur notre token <tok1>. 
Pour cela on peut utiliser un avantage qu'a le modèle de diffusion
par rapport au styleGAN : il génère à partir de plusieurs tokens. 
On peut alors trouver une méthode pour cibler la régularisation. On accompagne notre token <tok1> dans le prompt d'autres
tokens pour faire en sorte que la feature apparence de <tok1> ne soit
pas utilisée. Par exemple on peut écrire : "an anime illustration of <tok1> woman with long blue hair", de manière
à ce que l'image générée construise toutes les features à partir de <tok1>, sauf l'apparence.

Voici une schématisation des features utilisées par le nouveau modèle et l'ancien en cas de régularisation, en utilisant
un tel prompt :

Modèle original :

1\*apparence_org + 1\*autre_org

Tuning :

0.1\*apparence_tuning + 0.9\*apparence_org + 1\*autre_tuning

Erreur : 

0.1\*apparence_tuning + 1\*autre_tuning + ...

L'erreur entre les deux modèles va donc principalement concerner les features autres que l'apparence, ce qui va forcer
notre modèle lors du finetuning à rester proche de l'ancien modèle sur ces features. Ainsi on ne devrait apprendre 
réellement que l'apparence.

Pour tirer parti de cette régularisation, on peut partir de la configuration d'entrainement où on avait un overfitting
qui correspond à un learning rate de 1e-4. On peut espérer qu'en appliquant cette régularisation, on n'ait plus 
d'overfitting comme seule l'apparence devrait être apprise, ce qui nous permettra de parcourir une zone plus vaste
de l'espace des fonctions et donc d'avoir potentiellement une meilleure reconstruction qu'avec un learning rate de 
1e-5.

Il reste cependant un problème à traiter. J'ai écrit : 
0.1\*apparence_tuning + 0.9\*apparence_org + 1\*autre_tuning, laissant sous entendre que le modèle prendrait en compte
10% de la feature apparence de <tok1> et 90 % de la feature apparence du reste du prompt. En réalité ces pourcentages
sont arbitraires, et peuvent être bien plus élevées en faveur de <tok1> ce qui pose problème.
Par exemple, pour la configuration avec un lr de 1e-4 qu'on a prévu d'utilisé, l'apparence est prise à 100% depuis <tok1>.

Pour diminuer ce pourcentage, il faut s'éloigner de <tok1> dans l'espace des embeddings. Pour cela, on peut faire un
vSLERP entre <tok1> et <character>. La méthode vSLERP est de base pensée pour l'embedding du token de fin d'une phrase/image, qui est
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



