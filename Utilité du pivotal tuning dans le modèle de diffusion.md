# Le finetuning apprend les concepts les moins changés en premier 

Lors d'un finetuning une fonction apprend différents concepts. Par exemple en finetunant notre fonction sur un personnage, la fonction
apprend son apparence, mais aussi des éléments non voulus comme sa position, l'environnement dans lequel il se situe. Les concepts dont
la différence est la plus faible entre l'état de base et le dataset de finetuning seront améliorés les premiers car la fonction G dans
l'espace des fonctions aura moins de distance à parcourir pour les améliorer.
Par exemple, si un personnage dans le dataset contient plusieurs
positions différentes, pour apprendre ces positions le modèle devra déjà abandonner le fait qu'il produit des positions aléatoires pour
le token associé à ce personnage, et en plus il ne pourra apprendre une position donnée que lorsqu'il passe sur l'image du dataset avec
cette position. Pour l'apparence, l'état de base de la fonction est déjà proche de l'état d'arrivée grâce à l'inversion, et chaque image du dataset contribue
également à améliorer l'apparence donc elle sera améliorée beaucoup plus vite que la position. 

Si on veut améliorer un concept précis
comme l'apparence, il faut donc qu'on se déplace pas plus loin que la distance permettant de modifier ce concept et pas les autres.
La distance de déplacement peut être mesurée approximativement par learning rate x nb de steps (ce n'est pas exact car parfois la fonction
peut rester bloquée dans un minimum local et ne plus avancer, mais ça donne une idée). 
Si on veut augmenter la précision d'apprentissage
d'un concept, il faut diminuer le learning rate pour s'approcher plus précisemment du minimum global, et augmenter le nombre de steps
pour conserver la distance parcourue par la fonction.

Voici un exemple dans le cas du modèle de diffusion. Après l'inversion, on fait un finetuning avec un learning rate de 1e-4 sur 1000 steps.
On constate que les positions ont été apprises en plus de l'apparence. On s'est donc trop déplacé.
Pour y remédier, on diminue le learning rate en gardant le nombre de steps constants, ce qui diminue la distance parcourue. Désormais seule
l'apparence est apprise et pas les positions.




# Le pivotal tuning fonctionne pour le modèle de diffusion, mais est-il une solution optimale comme pour le styleGAN ?

Dans le cas du styleGAN, l'inversion permet de gagner en reconstruction sans inconvénient. L'apparence étant alors proche de celle recherchée, un entrainement avec un learning rate pas trop élevé va modifier uniquement cette apparence sans modifier d'autres concepts.

Maintenant pour le modèle de diffusion, l'inversion n'est pas gratuite comme pour le styleGAN : lorsque le token s'éloigne de la classe de départ, on perd en editability. Découper l'apprentissage entre une phase d'inversion et une phase de finetuning n'est alors plus évident : peut être qu'un simple finetuning depuis le token de la classe donnera une plus faible perte en editability que la perte de l'inversion ajoutée à celle du finetuning.
