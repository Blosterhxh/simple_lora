# Le finetuning apprend les concepts les moins changés en premier 

Lors d'un finetuning une fonction apprend différents concepts. Par exemple en finetunant notre fonction sur un personnage, la fonction
apprend son apparence, mais aussi des éléments non voulus comme sa position, l'environnement dans lequel il se situe. Les concepts dont
la différence est la plus faible entre l'état de base et le dataset de finetuning seront améliorés les premiers car la fonction G dans
l'espace des fonctions aura moins de distance à parcourir pour les améliorer.
Par exemple, si un personnage dans le dataset contient plusieurs
positions différentes, pour apprendre ces positions le modèle devra déjà abandonner le fait qu'il produit des positions aléatoires pour
le token associé à ce personnage, et en plus il ne pourra apprendre une position donnée que lorsqu'il passe sur l'image du dataset avec
cette position. Pour l'apparence, l'état de base de la fonction est déjà proche de l'état d'arrivée, et chaque image du dataset contribue
également à améliorer l'apparence donc elle sera améliorée beaucoup plus vite que la position. Si on veut améliorer un concept précis
par exemple l'apparence, il faut qu'on se déplacer pas plus loin que la distance permettant de modifier ce concept et pas les autres.
La distance de déplacement peu être mesurée globalement par learning rate x nb de steps (ce n'est pas exact car parfois la fonction
peu rester bloquer dans un minimum local et ne plus avancer, mais sa donne une idée). Si on veut augmenter la précision d'apprentissage
d'un concept, il faut diminuer le learning rate pour s'approcher plus précisemment du minimum global, et augmenter le nombre de steps
pour conserver la distance parcourue par la fonction.

Voici un exemple dans le cas du modèle de diffusion. Après l'inversion, on fait un finetuning avec un learning rate de 1e-4 sur 1000 steps.
On constate que les positions ont été apprises en plus de l'apparence. On s'est donc trop déplacé.
Pour y remédier, on diminue le learning rate en gardant le nombre de steps constants, ce qui diminue la distance parcourue. Désormais seule
l'apparence est apprise et pas les positions.




# Même si le pivotal tuning marchait pour les modèles de diffusion, serait-il vraiment utile ?

Au début du finetuning, plus on est éloigné du point d'arrivée, plus au cours du finetuning on va perdre en editability. Dans le cas du styleGAN,
cette perte est négligeable car on était proche du point d'arrivée grâce à l'inversion. C'est pour ça qu'on ne fait pas de reconstruction tout de suite et qu'on fait une inversion en 
premier lieu : si on faisait uniquement le finetuning, on ne pourrait alors plus dire qu'on a pas perdu en editability car on aurait trop modifié le modèle.

Cependant, il faut noter que contrairement au styleGAN, l'inversion a un coût pour le modèle de diffusion lorsque le token s'éloigne de la classe de départ. 
Là où l'inversion était évidente pour le styleGAN car elle offre une reconstruction sans inconvénient, cela n'est plus le cas pour le modèle de diffusion.
En effet, qui nous dit que inverser puis finetuner depuis la classe de départ, est plus efficace que finetuner depuis la classe de départ, comme l'inversion partage avec
le finetuning la perte en editability.
