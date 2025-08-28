Pour toutes ces etapes, on prend un nombre
de steps constant de 1000 pour ne pas
eterniser l enteainement, et on etudiera 
l'influence des autres parametres.

# Choix du learning rate

Pour l'inversion on a vu qu'augmenter le
learning rate. Dans notre cas contrairement
contrairement au pivotal tuning,
l'inversion n'est pas gratuite car on va
perdre en editability.

L'idée est de voir comment la reconstruction
et l'editability évoluent avec l'
éloignement au token de départ. Dans l'
idéal on devrait avoir une partie où on
gagne beaucoup en reconstruction et on
perd peu en editability, suivi d'une partie
ou le gain en reconstruction est plus
faible et la perte en editability plus
importante.

Pout tester différents éloignement,
on teste plusieurs learning rate a nb de 
steps constant et on a environ éloignement
= learning rate x nb de steps.

# Choix du learning rate pour le finetuning

Dans le styleGAN ils disent qu'ils 
appliquent un "léger" finetuning qui
leur permet de gagner en reconstruction
sans perdre en editability. Il faut donc définir qu'est-ce qu'un finetuning léger
avant de pouvoir trouver le bon learning
rate. Pour cela on va expliquer le fonctionnement general du finetuning.

Lors d'un finetuning une fonction apprend différents concepts. Par exemple en finetunant notre fonction sur un personnage, la fonction apprend son apparence, mais aussi des éléments non voulus comme sa position, l'environnement dans lequel il se situe. Les concepts dont la différence est la plus faible entre l'état de base et le dataset de finetuning seront améliorés les premiers car la fonction G dans l'espace des fonctions aura moins de distance à parcourir pour les améliorer. Par exemple, si un personnage dans le dataset contient plusieurs positions différentes, pour apprendre ces positions le modèle devra déjà abandonner le fait qu'il produit des positions aléatoires pour le token associé à ce personnage, et en plus il ne pourra apprendre une position donnée que lorsqu'il passe sur l'image du dataset avec cette position. Pour l'apparence, l'état de base de la fonction est déjà proche de l'état d'arrivée grâce à l'inversion, et chaque image du dataset contribue également à améliorer l'apparence donc elle sera améliorée beaucoup plus vite que la position.

Si on veut améliorer un concept précis comme l'apparence, il faut donc qu'on se déplace pas plus loin que la distance permettant de modifier ce concept et pas les autres. C'est cela que les auteurs du pivotal tuning veulent dire quand ils parlent de finetuning léger. 

Pour trouver la bonne distance de finetuning, on essaie différents learning rate pour trouver la distance limite où
on commence a apprendre les positions, et on s'arrête juste avant.


# Regularisation

Les auteurs du pivotal tuning on rajouté un terme de regularisation a la loss
pour s'assurer que le générateur ne modifie
ses valeurs que localement autour du pivot.

Cette regularisation a un objectid double : faire en sorte que le modèle garde ses capacités de base, et permette un apprentissage plus precis de l'apparence. En effet on a dit qu'on devait se limiter a une certaine distance d'apprentissage pour ne pas toucher aux positions. Sauf qu'avec ce terme de regularisation, on empeche le generateur de se deplacer dans des directions où il va modifier autre chose que l'apparence. On peut donc augmenter le learning rate pour parcourir une zone plus vaste de l'espace des fonctions et ameliorer l'apparence sans craindre une répercution sur les autres features qui pourraient etre apprises.






