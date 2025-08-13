# L'absence de regularisation dans l'article

Sur le code du github de textual inversion :
https://github.com/rinongal/textual_inversion/blob/424192de1518a358f1b648e0e781fdbe3f40c210/configs/stable-diffusion/v1-finetune_unfrozen.yaml on voit ce fichier de configuration utilisé
pour le pivotal tuning qui indique que la loss est calculée par le model LatentDiffusion, et cette loss dont on peut retrouver le chemin avec training_step -> shared_step -> forward
-> p_losses dans ce fichier https://github.com/rinongal/textual_inversion/blob/424192de1518a358f1b648e0e781fdbe3f40c210/ldm/models/diffusion/ddpm.py#L351 ne prend en compte que la différence
entre l'image générée et l'image cible, il n'y a pas de terme de régularisation. Il y a probablement une raison non expliquée dans l'article qui explique ce choix, mais on peut quand même
essayer d'ajouter une régularisation, d'autant plus qu'ils constatent une perte d'editability avec le pivotal tuning qui est peut être due à un finetuning trop large.
