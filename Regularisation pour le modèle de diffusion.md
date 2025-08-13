# L'absence de regularisation dans l'article

Sur le code du github de textual inversion :
https://github.com/rinongal/textual_inversion/blob/424192de1518a358f1b648e0e781fdbe3f40c210/configs/stable-diffusion/v1-finetune_unfrozen.yaml on voit ce fichier de configuration utilisé
pour le pivotal tuning qui indique que la loss est calculée par le model LatentDiffusion, et cette loss dont on peut retrouver le chemin avec training_step -> shared_step -> forward
-> p_losses dans ce fichier https://github.com/rinongal/textual_inversion/blob/424192de1518a358f1b648e0e781fdbe3f40c210/ldm/models/diffusion/ddpm.py#L351 ne prend en compte que la différence
entre l'image générée et l'image cible, il n'y a pas de terme de régularisation. C'est la même chose dans le code original de cloneofsimo : https://github.com/cloneofsimo/lora. Il y a probablement une raison non expliquée dans l'article/dans le repo de cloneofsimo qui explique ce choix, mais on peut quand même
essayer d'ajouter une régularisation, d'autant plus que dans l'article ils constatent une perte d'editability avec le pivotal tuning qui est peut être due à un finetuning trop large.

# Pourquoi le finetuning dans notre modèle de diffusion pourrait ne pas être local

Même si on finetune l'unet uniquement sur notre nouveau token, il peut très bien modifié les poids pour 
d'autres tokens pour améliorer notre cas spécifique. Par exemple si notre personnage a les yeux d'une certaine
forme : il peut modifier les yeux de tous les personnages qu'il va générer pour qu'ils aient cette forme 
particulière, et comme la loss ne prend en compte que l'optimisation sur notre token, le modèle va voir
que ça fonctionne donc il va continuer a modifier son interprétation des autres tokens tant que on a une 
amélioration dans notre cas.

# Adapter la régularisation du styleGAN au modèle de diffusion

Dans le cas de styleGAN, on procède comme suit. On tire un vecteur dans Z transformé par le mapping network en wz, on génère son image
avec le nouveau générateur et celui d'origine et on calcule la différence entre les deux, qui est notre
terme de régularisation. Cela force le générateur a ne pas modifier son comportement sur les autres latents.
Les auteurs de l'article ont également remarqué que pour améliorer cette méthode, il était idéal d'interpoler
entre wp (le pivot) et wz à mi-distance, et de calculer le terme de regularisation sur ce nouveau latent 
wp + 1/2(wp-wz) qu'on notera wz2.

Maintenant il faut trouver une adaptation au modèle de diffusion. Premièrement on va laisser de côté l'interpolation
et se concentrer uniquement sur le choix des tokens. On peut déjà faire comme styleGAN et générer une image aléatoire.
Cela correspond à générer une image avec aucun conditionnement par le texte, on donne une chaîne vide "" au
text_encoder. Cependant on peut probablement être plus précis avec un modèle de diffusion. Un styleGAN est entrainé
sur un espace latent qui génère des images sembables (ex : des visages) donc un finetuning d'un visage peut
potentiellement affecté tous les autres, comme on l'a vu avec l'exemple de la forme des yeux. Dans notre cas,
le modèle peut générer beaucoup d'éléments différents, et le finetuning d'un personnage ne va probablement pas 
altérer des concepts différents comme un paysage. On peut donc conditionner la génération sur les personnages
avec des prompts comme "a character". Pour l'interpolation, il est possiblement mieux de la faire non pas
dans l'espace juste après le tokenizer, mais dans l'espace après le text encoder. En effet dans cet espace
la géométrie est liée à la sémantique, donc en interpolant entre "tok1" et "character" on devrait arriver
à un entre deux comme dans styleGAN. On peut donc tirer des vecteurs dans l'hypersphère à mi-distance entre
"tok1" et "character" dans cet espace.





