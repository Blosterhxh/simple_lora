# Même si le pivotal tuning marchait pour les modèles de diffusion, serait-il vraiment utile ?

Au début du finetuning, plus on est éloigné du point d'arrivée, plus au cours du finetuning on va perdre en editability. Dans le cas du styleGAN,
cette perte est négligeable car on était proche du point d'arrivée grâce à l'inversion. C'est pour ça qu'on ne fait pas de reconstruction tout de suite et qu'on fait une inversion en 
premier lieu : si on faisait uniquement le finetuning, on ne pourrait alors plus dire qu'on a pas perdu en editability car on aurait trop modifié le modèle.

Cependant, il faut noter que contrairement au styleGAN, l'inversion a un coût pour le modèle de diffusion lorsque le token s'éloigne de la classe de départ. 
Là où l'inversion était évidente pour le styleGAN car elle offre une reconstruction sans inconvénient, cela n'est plus le cas pour le modèle de diffusion.
En effet, qui nous dit que inverser puis finetuner depuis la classe de départ, est plus efficace que finetuner depuis la classe de départ, comme l'inversion partage avec
le finetuning la perte en editability.
