## Evaluation d'une méthode de modification

Avant de résoudre ce problème, il faut se demander comment on évalue la qualité d'une edit d'une image, et qu'est-ce qui permet de dire qu'effectivement les edits d'images de W
sont moins bonnes que celles du reste de R512. On va voir trois critères qui permettent de faire cette évaluation

### Distortion

La distortion mesure la distance entre deux images. On choisit une norme sur l'espace des images, et on calcule la différence entre les deux vecteurs.
Une bonne edit a donc une distortion faible car l'image doit ressembler à celle de départ.

### Perceptual quality

La perceptual quality indique si l'image est réaliste. Ce n'est pas la même chose que la distortion, car une image peut être proche d'une image réaliste sans avoir pour autant de sens,
par exemple un visage auquel on aurait enlevé la bouche peut être proche du visage complet mais il n'aura pas de sens réel.

### Editability




