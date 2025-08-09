#Le styleGAN et l'espace latent W

Pré-requis : savoir ce qu'est un GAN

## Le styleGAN : un GAN où le générateur est modifié.

Au lieu de mettre en entrée du générateur le vecteur z tiré de l'espace Z (par exemple R**512),
on met un vecteur constant appris avec l'entrainement, et le vecteur z est injecté après chaque convolution
dans le générateur. Cela permet au générateur d'extraire à chaque étape les informations dont il a besoin
au lieu de devoir se souvenir de tout depuis le début. Entre chaque convolution on ajoute également un vecteur 
de bruit, qui permet de donner une base au générateur pour construire de l'aléatoire à chaque étape (par exemple
pour construire des cheveux). Sans cela il devrait lui-même trouver un moyen de construire de l'aléatoire avec 
ses poids et les autres capacités de reconstruction en seraient donc diminuées.

## L'espace latent W

Dans cette partie on prend comme exemple pour illustrer un styleGAN entrainé sur des visages humains.

Dans l'idéal, on aimerait que l'espace Z soit disentangled (désemmêlé) : cela signifie que pour chaque
facteur de variation de l'image (ex : rotation du visage), on peut trouver un sous-espace vectoriel dans Z, dont les directions
contrôle la modification de ce facteur de variation (ex : une direction pour la rotation autour de chaque axe).
L'intuition pour comprendre que cette espace disentangled peut exister, est que des facteurs de variations différents (
cheveux, vieillissement) modifient des attributs communs entre eux (couleur des cheveux est partagé entre vieillissement
et cheveux), donc pour créer un sous-espace il faut trouver les directions qui modifient les attributs conformément à
notre facteur de variation.

Le problème ici est que Z est sensé suivre la distribution du dataset sur lequel
le GAN est entrainé. Cela veut dire que si certaines valeurs d'un facteur de variation sont absentes du dataset, elle ne seront
pas dans Z. Or pour représenter ce facteur de variation par un sous-espace vectoriel il faut que toutes les valeurs qu'il
prend soient possibles. Pour illustrer ce problème, voici un schéma tiré de l'article. On prend Z = R**2, et deux facteurs
de variations (dans leur cas masculunité et longueur de cheveux). Si je veux modifier la longeur des cheveux, je dois trouver un
sous-espace qui modifie uniquement ce facteur donc de dimension 1 (un sous espace de dimension 2 modifierait aussi la 
masculunité) c'est-à-dire une droite. En observant Z, on voit qu'on ne peut pas tracer de droites pour faire évoluer ce facteur.


L'espace de départ Z est entangled (emmêlé) : les attributs d'une image (couleur, partie physique, position etc.) sont mélangés 
dans les différentes dimensions, et se déplacer
dans une direction ne contrôle pas un attribut spécifique (par exemple on ne trouve pas de direction pour quantifier
la rotation du visage). Pour le comprendre, on peut raisonner par l'absurde en supposant qu'il est disentangled.
En étant disentangled, on pourrait se déplacer dans une direction pour modifier un attribut, et donc cela 
signifie que toutes les combinaisons d'attributs sont possibles. Or le dataset est limité et l'espace Z suit
la distribution du dataset donc une combinaison absente de la distribution du dataset est absente de Z, donc 
Z ne peut pas être disentangled. C'est ce qui est illustré sur la figure,où l'on voit
que l'absence d'une combinaison d'attributs dans le dataset fait varier les attributs de façon non linéaire
dans l'espace Z.

L'espace latent W :

On résout ce problème en créant un espace latent après Z : f(Z) = W.
L'espace latent résout le problème car il peut occuper une partie de l'espace dans lequel il est inclus et pas sa
totalité, ce qui n'est pas le cas de Z car le tirage selon la loi normale implique que l'on peut prendre des vecteurs
de tout l'espace Z. Ainsi si W est inclus dans W*, et que une combinaison d'attributs est impossible, et bien
cette combinaison sera dans W* mais exclu de W ce qui permet quand même à W de faire varier les attributs de
façon linéaire, seulement quand une certaine combinaison n'existera pas et bien elle ne sera plus dans W et 
on n'a pas besoin de déformer les features comme on le fait sur Z.

Comment modifier une image quelconque en utilisant les propriétés du styleGAN  :

L'espace latent du styleGAN est disentangled : des dimensions différentes codent des attributs différents.
Cela permet en se déplaçant dans des directions précises de modifier un attribut particulier et donc de modifier
une image à son souhait. Toutefois toutes les images ne sont pas dans W, certaines sont dans W* donc on ne peut pas
modifier n'importe quelle image. 
