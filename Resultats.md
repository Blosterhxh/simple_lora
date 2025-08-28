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
