# Le styleGAN et l'espace latent W

Pré-requis : savoir ce qu'est un GAN

Article original : https://arxiv.org/pdf/1812.04948

## Le styleGAN : un GAN où le générateur est modifié.

Au lieu de mettre en entrée du générateur le vecteur z tiré de l'espace Z (par exemple R512),
on met un vecteur constant appris avec l'entrainement, et le vecteur z est injecté après chaque convolution
dans le générateur. Cela permet au générateur d'extraire à chaque étape les informations dont il a besoin
au lieu de devoir se souvenir de tout depuis le début. Entre chaque convolution on ajoute également un vecteur 
de bruit, qui permet de donner une base au générateur pour construire de l'aléatoire à chaque étape (par exemple
pour construire des cheveux). Sans cela il devrait lui-même trouver un moyen de construire de l'aléatoire avec 
ses poids et les autres capacités de reconstruction en seraient donc diminuées.

![styleGAN.png](styleGAN.png)

## L'espace latent W

Dans cette partie on prend comme exemple pour illustrer un styleGAN entrainé sur des visages.

Dans l'idéal, on aimerait que l'espace Z soit disentangled (désemmêlé) : cela signifie que pour chaque
facteur de variation de l'image (ex : rotation du visage), on peut trouver un sous-espace vectoriel dans Z, dont les directions
contrôle la modification de ce facteur de variation (ex : une direction pour la rotation autour de chaque axe).
L'intuition pour comprendre que cette espace disentangled peut exister, est que des facteurs de variations différents (
cheveux, vieillissement) modifient des attributs communs entre eux (couleur des cheveux est partagé entre vieillissement
et cheveux), donc pour créer un sous-espace il faut trouver les directions qui modifient les attributs conformément à
notre facteur de variation.

Analysons Z pour voir si l'espace est disentangled. Z peut-être représenté comme une hypersphère de rayon racine de 512 de R512, dans laquelle
la majorité des vecteurs vont être tirés d'après la loi normale. Avec l'entrainement, sur cette sphère, le générateur va prendre
des valeurs qui sont proches de celles du dataset. Supposons maintenant qu'un facteur de variation à une zone de valeur absente du dataset,
par exemple supposons que sur le dataset ils prennent des valeurs de [0,1] puis de [2,+oo] .
Si on peut représenter ce facteur de variation avec un sous-espace dans Z, on peut prendre une direction de cette espace sur laquelle
déplacer G va faire varier le facteur de variation de manière continue. On peut donc trouver une portion de droite, dans l'hypersphère, où G va prendre des valeurs pour le facteur de variation qui sont impossibles [1,2]. Ainsi les facteurs de variation ne peuvent
pas être représentés par des sous-espaces dans Z, l'espace est entangled. 

On résout ce problème en créant un espace latent après Z : f(Z) = W.
L'espace latent résout le problème car il n'a pas la contrainte spaciale que les vecteurs dans l'hypersphère donne des images proches
du dataset. Il peut donc choisir librement les valeurs que G prend sur chaque zone de l'espace. Avec l'optimisation on peut alors approcher
la représentation des facteurs de variation par des sous-espaces.

L'article utilise ce schéma pour illustrer le problème :

![styleganW.png](styleganW.png)

## Modifier une image avec W

Maintenant que l'espace est disentangled, on peut se dire que pour modifier une image il suffit de l'inverser dans R**512, et de 
se déplacer dans les sous-espaces des facteurs de variation. 
Le problème est que si G a appris à se déplacer sur un sous-espace pour faire varier un facteur de variation, il ne l'a appris
que sur l'intersection entre ce sous-espace et W, donc en sortant de W les résultats vont être moins précis. 
