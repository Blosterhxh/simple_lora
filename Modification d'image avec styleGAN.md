# Modifier une image en utilisant le styleGAN

Pour modifier une image, on peut tirer parti de la représentation des facteurs de variation en sous-espaces vectoriels dans W. On prend une image, on l'inverse dans l'espace latent W, on se déplace dans le sous-espace d'un facteur de variation, puis on applique G à notre nouveau latent et on récupère notre image modifiée. Cette méthode fonctionne, mais elle pose plusieurs problèmes.

## Le problème de l'inversion

L'inversion d'une image dans W donne un latent w, mais la reconstruction de l'image avec G(w) ne redonne pas parfaitement l'image. Pour évaluer une reconstruction, on introduit deux mesures : la distortion et la perceptual quality. La distortion correspond à la distance entre deux images mesurée avec une norme sur l'espace des images. La perceptual quality désigne à quel point une image est réaliste.
L'idée est qu'une image peut ressembler esthétiquement à une autre, mais n'avoir aucun sens réel. 

Comme l'inversion dans W n'est pas toujours satisfaite, on propose de l'améliorer en agrandissant l'espace de départ sur lequel est réalisé l'optimisation. Cela peut être fait de différentes manières : prendre k vecteurs latents dans W (un par couche du générateur) au lieu d'un, prendre un vecteur dans 
W* = R512 dans lequel est inclus W, prendre k vecteurs dans W*k. Un graphique de l'article illustre comme s'organisent ces différents espaces.

Note : Pourquoi prendre k vecteurs dans W au lieu d'un pourrait marcher alors qu'on a entrainé le générateur à n'utiliser
qu'un seul vecteur ?

Le générateur à chaque couche extrait des informations différents d'un vecteur w, pour les couches plus profondes
des informations globales, pour les couches en surface des informations plus précises. Donc pour une image donnée,
peut être qu'un vecteur w peut donner les informations exactes pour les informations globales mais ne pourra 
pas restituer parfaitement les informations précises, et inversement, d'où la possibilité que plusieurs vecteurs

## Le compromis distortion/perceptual quality + editability  

On est maintenant capable de faire des reconstructions plus précises en inversant dans des espaces plus grands, mais cela n'est pas sans coût. Premièrement, il y a une perte de la perceptual quality. Si les images ont une meilleur distortion, elles ont par contre moins de sens que celles inversées dans W, comme on peut le voir dans cet exemple entre une image inversée dans W et dans W*k.

De plus, on constate également une perte en editability. Les modifications apportées à des images inversées dans W donnent de meilleurs résultats que celles inversées dans W*k, comme on peut le voir sur cette image.

En fait, il y a un compromis entre la distortion, et la perceptual quality + editability. Plus on s'écarte de l'espace W, plus on gagne en distortion mais en perdant sur les deux autres caractéristiques. Cela s'explique car c'est sur W que le générateur a été entrainé à produire des images qui ont du sens.

Pour évaluer la distance d'un vecteur de W*k à W, on utilise deux critères. Le premier est la distance de W*k à W*, qui 
correspond à la variation entre les vecteurs de W*k : plus les vecteurs sont proches entre eux, plus on peut 
considérer qu'on est proche de W*. Le deuxième critère est la distance de W*k à Wk, qui correspond à la distance
individuelle des k vecteur de W* à W. En minimisant ces deux critères, un vecteur de W*k se rapproche de W.

Ce graphique issu de l'article reprend la représentation précédente de W*k, et introduit deux couleurs de flèche pour les deux moyens de s'approcher de W : la flèche rouge pour diminuer la variation entre vecteurs, et la flèche bleue pour le rapprochement des vecteurs individuellement à ceux de W.





