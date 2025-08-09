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

Analysons Z pour voir si l'espace est disentangled. Z peut-être représenté comme une hypersphère de rayon racine de 512 de R**512, dans laquelle
la majorité des vecteurs vont être tirés d'après la loi normale. Avec l'entrainement, sur cette sphère, le générateur va prendre
des valeurs qui sont proches de celles du dataset. Supposons maintenant qu'un facteur de variation à une zone de valeur absente du dataset.
Si on peut représenter ce facteur de variation avec un sous-espace dans Z, on peut prendre une direction de cette espace sur laquelle
déplacer G va faire varier le facteur de variation de manière continue. On peut donc trouver une portion de droite, dans le cercle de
rayon 1, où G va prendre des valeurs pour le facteur de variation qui sont impossibles. Ainsi les facteurs de variation ne peuvent
pas être représentés par des sous-espaces dans Z, l'espace est entangled.

de Z ne générera cette zone de valeur avec G, donc on ne pourra pas représenter ce facteur de variation avec un sous-espace car 
aucune direction ne pasn bbbbbbbbbbbbbbbbbb
le GAN est entrainé.
Cela veut dire que si certaines valeurs d'un facteur de variation sont absentes du dataset, elle ne seront
pas dans Z. Or pour représenter ce facteur de variation par un sous-espace vectoriel il faut que toutes les valeurs qu'il
prend soient possibles. Pour illustrer ce problème, voici un schéma tiré de l'article. On prend Z = R**2, et deux facteurs
de variations (dans leur cas masculinité et longueur de cheveux). Si je veux modifier la longueur des cheveux, je dois trouver un
sous-espace qui modifie ce facteur donc dans ce cas une droite. En observant Z, on voit que la direction pour faire évoluer ce 
facteur de variation de manière continue change, on ne pas donc pas trouver de telle droite.

On résout ce problème en créant un espace latent après Z : f(Z) = W.
L'espace latent résout le problème car il est libre de positionner les latents où il veut dans l'espace R**512, là où
dans le cas de Z ces latents sont concentrés uniformément dans une sphère de rayon racine de d à cause de la loi normale.
Ainsi si on veut représenter un facteur de variation par un sous-espace dans W, mais que certaines valeurs ne sont pas prises
par ce facteur dans le dataset, et bien on peut quand même placer les latents en suivant ce sous-espace, en laissant
des parties vides là où les valeurs ne sont pas dans le dataset.

il peut occuper une partie de l'espace dans lequel il est inclus et pas sa
totalité, ce qui n'est pas le cas de Z car le tirage selon la loi normale implique que l'on peut prendre des vecteurs
de tout l'espace Z. Ainsi dans W on a des sous-espaces pour les facteurs de variation, et si une valeur d'un facteur de variation
est impossible et bien elle ne sera pas dans W : on aura un "trou" dans l'espace vectoriel où G ne prendra pas valeurs.

Comment modifier une image quelconque en utilisant les propriétés du styleGAN  :

L'espace latent du styleGAN est disentangled : des dimensions différentes codent des attributs différents.
Cela permet en se déplaçant dans des directions précises de modifier un attribut particulier et donc de modifier
une image à son souhait. Toutefois toutes les images ne sont pas dans W, certaines sont dans W* donc on ne peut pas
modifier n'importe quelle image. 
