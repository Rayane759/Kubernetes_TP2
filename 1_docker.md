# TP sur la conteneurisation 

## Objectifs du TP
- Maîtriser la construction d'images Docker
- Comprendre et utiliser les multi-stage builds
- Optimiser le processus de build et la taille des images
- Savoir exécuter et débugger des conteneurs

Livrable : 
- Dockerfile pour les deux exercices 
- Un compte rendu avec les difficultés rencontrées, et les résultats de chaque exercice. 

## Prérequis

- Installer docker ou podman,
- Avoir un compte docker hub pour pousser les images.

## Exercice 1 : Construire et exécuter une application Python basique 

Dans cet exercice, vous allez devoir conteneuriser et lancer une API Python en suivant le protocole suivant : 

1. Récupérer le code de l'application Python 
2. Rédiger le Dockerfile associé 
3. Documenter ce dernier en indiquant les layers importantes
4. Construire l'image avec Docker 
5. Exécuter cette dernière 
6. Faire un appel sur l'API
7. Consulter les logs de l'API
8. Push sur docker hub son image 

## Exercice 2 : Conteneurisation d'une API Java : Optimisation et multi stage build. 

Dans ce second exercice, vous suivrez la même procédure que l'exercice précédent mais vous devrez : 
- Mettre en place deux stages 
    - build de l'application avec un jdk
    - run de l'application avec un jre


